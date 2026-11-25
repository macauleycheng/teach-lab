# 如何手動從已安裝的廠商 SONiC 設備取出 Marvell SAI(.deb)

本文以 Accton ECS5550-54X(Marvell Aldrin2, arm64)上的 SONiC-OS-Lite-1.6.4_202405_ols 為例。
實際取出結果請見 `REPORT.md`(套件 `mrvllibsai 1.14.0-1`)。

文中的佔位符,請自行替換:

- `<DEVICE_IP>`:設備 IP
- `<USER>`:登入帳號(需要能執行 docker 與 sudo)
- `<PLATFORM>`:平台目錄名,例如 `arm64-accton_ecs5550_54x-r0`
- `<SYNCD_IMAGE>`:syncd 映像,例如 `docker-syncd-mrvl:Lite-1.6.4_202405_ols`

## 0. 前提與現況

- 廠商 SONiC 的 SAI 不在宿主機上,而是在 syncd 容器映像內(套件 `mrvllibsai`,內含 `libsai.so`)。
- 即使 syncd 容器目前是 down(例如 port 配置導致 SAI 初始化失敗),映像仍在磁碟上。
  不需要把 syncd 拉起來,用映像建立臨時容器讀檔即可。
- 臨時容器使用 `--network none` 與 `--entrypoint sh`,不會執行 syncd 啟動腳本,也不會碰到 ASIC 或 redis。
- 全程是唯讀操作,不需要 restart/stop 任何服務、不改 config_db、不 reboot。
- 暫存檔放在設備的 `/tmp`,最後要清除。

## 1. 登入並找到 syncd 映像

```bash
ssh <USER>@<DEVICE_IP>

docker images | grep -i syncd
docker ps -a | grep -i syncd      # 確認容器狀態(down 也沒關係)
```

記下映像名稱與 tag,後面以 `<SYNCD_IMAGE>` 表示。

## 2. 用臨時容器找出 SAI 套件

```bash
docker run --rm --network none --entrypoint sh <SYNCD_IMAGE> -c \
  'cat /etc/os-release; dpkg -l | grep -i -E "sai|syncd|mrvl|marvell|cpss"'
```

預期可見 `mrvllibsai`(SAI 本體)、`libsairedis`、`libsaimetadata`、`syncd`。
再列出 mrvllibsai 內含檔案:

```bash
docker run --rm --network none --entrypoint sh <SYNCD_IMAGE> -c \
  'dpkg -L mrvllibsai; dpkg -s mrvllibsai'
```

實際範例包含 `/usr/lib/libsai.so`、`/usr/lib/libXdkCpss.so`、`/usr/bin/mrvlcmd`、
`/opt/marvell/firmware/*.fw`、`/fwFiles/phy/*.hdr`、`/usr/include/sai/*.h`。

## 3. 重新打包成 .deb

### 3.1 若有 dpkg-repack

```bash
docker run --rm --network none --entrypoint sh -v /tmp/sai_out:/out <SYNCD_IMAGE> -c \
  'cd /out && dpkg-repack mrvllibsai'
```

### 3.2 沒有 dpkg-repack 時(本例的情況)

等同 dpkg-repack:用 `dpkg -L` 取檔案、複製、用 `dpkg -s` 還原 control、再 `dpkg-deb -b`。
下列腳本在容器內執行,輸出到掛載的 `/out`:

```bash
sudo mkdir -p /tmp/sai_out && sudo chmod 777 /tmp/sai_out

docker run --rm --network none --entrypoint sh -v /tmp/sai_out:/out <SYNCD_IMAGE> -c '
set -e
PKG=mrvllibsai
W=/tmp/repack/$PKG
mkdir -p $W/DEBIAN

# 1) 複製檔案(只複製一般檔案與 symlink,跳過目錄)
dpkg -L $PKG | while read f; do
    if [ -f "$f" ] || [ -L "$f" ]; then
        mkdir -p "$W$(dirname "$f")"
        cp -a "$f" "$W$f"
    fi
done

# 2) 由 dpkg -s 還原 control(去掉 Status 與 Installed-Size 行)
dpkg -s $PKG | grep -v -E "^(Status|Installed-Size|Conffiles| )" > $W/DEBIAN/control

# 3) 打包
dpkg-deb -b $W /out/${PKG}_$(dpkg -s $PKG | sed -n "s/^Version: //p")_$(dpkg --print-architecture).deb
'
```

注意:`dpkg -s` 的多行 Description 續行以空白開頭;上面的 grep 會把續行刪掉。
若想保留完整 Description,請改手動編輯 control,或只刪除 `Status:` 與 `Installed-Size:` 兩行。
另外,syncd、libsairedis、libsaimetadata 也可以用同樣方式取出(僅供比對,自己 build 的樹會自行編譯)。

重新打包後的 .deb 沒有原廠 md5sums,也可能沒有 changelog,只影響文件,不影響功能。

## 4. 把檔案帶出設備

方式 A:直接在設備上用 scp 傳到工作機(`<WORKSTATION>` 為你的電腦)。

```bash
scp /tmp/sai_out/*.deb <WORKSTATION_USER>@<WORKSTATION>:~/sai-extract/
```

方式 B:從工作機拉取。

```bash
mkdir -p ~/sai-extract && cd ~/sai-extract
scp '<USER>@<DEVICE_IP>:/tmp/sai_out/*' .
```

方式 C:直接用 `docker cp`(需要先 `docker create` 一個不啟動的容器)。

```bash
docker create --name sai_tmp --entrypoint sh <SYNCD_IMAGE>
docker cp sai_tmp:/usr/lib/libsai.so /tmp/sai_out/
docker cp sai_tmp:/usr/lib/libXdkCpss.so /tmp/sai_out/
docker rm sai_tmp
```

## 5. 驗證 sha256

在設備上先算原檔,再與工作機上的檔案比對:

```bash
# 設備:容器內原檔
docker run --rm --network none --entrypoint sh <SYNCD_IMAGE> -c \
  'sha256sum /usr/lib/libsai.so /usr/lib/libXdkCpss.so'

# 工作機:重打包 deb 內的檔案
cd ~/sai-extract
sha256sum *.deb
dpkg-deb -x mrvllibsai_*_arm64.deb /tmp/chk && sha256sum /tmp/chk/usr/lib/libsai.so /tmp/chk/usr/lib/libXdkCpss.so
rm -rf /tmp/chk
```

兩邊 `libsai.so`、`libXdkCpss.so` 的 sha256 必須一致。

## 6. 備份機板設定

機板設定(sai.profile、port_config.ini、各種 XML)與 SAI 版本相關,必須與 SAI 一起帶走。

```bash
sudo tar czf /tmp/sai_out/device_<PLATFORM>.tgz -C /usr/share/sonic/device <PLATFORM>
```

內容通常包含:`<SKU>/sai.profile`、`profile.ini`、`port_config.ini`、`SAI-*.xml`、`ASK-*.xml` 與 `.md5`、
`platform_asic`、`default_sku`、`installer.conf`、`plugins/`、`pmon_daemon_control.json`、`sonic_platform-*.whl`。

## 7. 宿主機的 mvcpss.ko 與 mrvlprestera

`mvcpss.ko` 是 Marvell CPSS 的 kernel module,libsai 透過它存取 ASIC,需與 libsai 版本相配,
並且必須與 kernel 的 vermagic 相同。

```bash
uname -r
lsmod | grep -E "mvcpss|mvpp2|prestera"
find /lib/modules/$(uname -r) -name "mvcpss.ko*"
dpkg -S $(find /lib/modules/$(uname -r) -name "mvcpss.ko*")     # 看是哪個套件提供
sha256sum $(find /lib/modules/$(uname -r) -name "mvcpss.ko*")
strings <mvcpss.ko 路徑> | grep vermagic                          # 設備上沒有 modinfo 時可用

sudo cp $(find /lib/modules/$(uname -r) -name "mvcpss.ko") /tmp/sai_out/

dpkg -l | grep -i -E "mrvlprestera|prestera|platform"
dpkg -L mrvlprestera > /tmp/sai_out/mrvlprestera_filelist.txt
```

宿主機平台套件(例如 `sonic-platform-accton-ecs5550-54x`)若位於 `/lib/...` 且系統為 usrmerge,
用 3.2 的方式重打包會出錯(`/lib` 是 symlink)。建議只取 `mvcpss.ko` 單檔,再用 `dpkg -L` 保留檔案清單即可。

## 8. 收集版本資訊

```bash
# SAI API 版本
docker run --rm --network none --entrypoint sh <SYNCD_IMAGE> -c \
  'grep -E "define SAI_(MAJOR|MINOR|REVISION)" /usr/include/sai/saiversion.h'

# libsai 依賴是否可解析
docker run --rm --network none --entrypoint sh <SYNCD_IMAGE> -c 'ldd /usr/lib/libsai.so'

# 容器 OS
docker run --rm --network none --entrypoint sh <SYNCD_IMAGE> -c 'cat /etc/os-release'

# 宿主機 image 版本、kernel
cat /etc/sonic/sonic_version.yml
uname -a
dpkg -l > /tmp/sai_out/host_dpkg.txt
```

這些資訊決定你的 build 樹能不能用這份 SAI:SAI API 版本、Debian 版本(glibc)、kernel 版本都要檢查。

## 9. 清理設備

```bash
docker ps -a | grep sai_tmp        # 若用方式 C 且沒刪乾淨,請 docker rm sai_tmp
rm -rf /tmp/sai_out /tmp/repack
```

`docker run --rm` 的臨時容器結束後會自動刪除。工作完成後,設備上的服務狀態應該與取出前一致。

## 10. 已知限制與注意事項

- usrmerge:宿主機平台套件含 `/lib/...` 路徑,重打包 .deb 時會出錯。請只取單檔(如 `mvcpss.ko`)。
- uid/gid:重打包的檔案擁有者是廠商 build 時的 uid/gid(例如 256377993/256377345)。
  安裝前或安裝後建議 `chown -R root:root`,或打包時使用 `dpkg-deb --root-owner-group -b`。
- kernel vermagic:`mvcpss.ko` 只能載入同版本 kernel(本例 `6.1.0-22-2-arm64`)。
  你的 kernel 不同時,不能直接載入,必須用對應原始碼重新編譯。
- 版本要成套:`libsai.so`、`mvcpss.ko`、XML/profile.ini 要同一版本,不要混用。
- 架構與 OS:arm64 的 SAI 不能用在 x86;libsai.so 以 Debian 12 的 glibc/libstdc++ 連結,
  目標 base 太舊(如 bullseye)可能無法載入。
- SAI API 版本與 sairedis:SAI 1.14 的 libsai 只能搭配 SAI header 相近的 sairedis/syncd。
  兩者差距大時,屬性與函式簽名可能不相容。
- 未驗證初始化:本方法只確認檔案完整與依賴可解析,無法保證 libsai 在其他 image 上能成功 `create_switch`。
- 授權:`libsai.so`、`libXdkCpss.so` 是 Marvell 的 binary,再發佈前請確認授權條款。

## 11. 如何把取出的 SAI 放進 sonic-buildimage(概要)

以下路徑已在 `/home/manager/marvell-sonic-buildimage`(rls-202511.01 分支)實際查過。
**但請先看 `COMPAT.md`:此樹需要的 SAI 是 1.17.x,與取出的 1.14.0 差距大,直接替換有高風險。**

1. SAI 套件定義在 `platform/marvell-prestera/sai.mk`:
   - 變數 `MRVL_SAI_VERSION`(目前 `1.17.1-21`)與 `MRVL_SAI`(`mrvllibsai_<ver>_<arch>.deb`)。
   - 預設從 `MRVL_SAI_URL_PREFIX`(GitHub `Marvell-switching/sonic-marvell-binaries`)下載,
     由 `SONIC_ONLINE_DEBS` 處理。
2. 使用本地 deb 的做法有兩種:
   - 把 `MRVL_SAI_VERSION` 改成取出的版本,並把 deb 放在可被下載的位置(自架 HTTP/鏡像),修改 `MRVL_SAI_URL_PREFIX`。
   - 或把 deb 放在 `target/debs/<distro>/`,並避免 make 重新下載(需視你的 build 流程驗證,此項為建議,未實測)。
3. syncd 容器引入 SAI 的位置:`platform/marvell-prestera/docker-syncd-mrvl-prestera.mk`
   (`$(DOCKER_SYNCD_BASE)_DEPENDS += $(MRVL_SAI)`),以及 `docker-syncd-mrvl-prestera/Dockerfile.j2`。
4. 機板設定放在 `device/marvell/<platform>/<SKU>/`(sai.profile、port_config.ini、XML)。
   本樹沒有 `device/accton/arm64-accton_ecs5550_54x-r0`,需自行新增,內容取自第 6 節備份。
5. 平台 kernel module:`platform/marvell-prestera/sonic-platform-marvell/debian/rules` 從原始碼建置 `mvcpss.ko`;
   廠商預編譯的 `mvcpss.ko` 若 vermagic 不符,不可直接用。
6. 版本控制:`sai.mk` 內 `$(MRVL_SAI)_SKIP_VERSION=y` 已略過版本鎖定。
