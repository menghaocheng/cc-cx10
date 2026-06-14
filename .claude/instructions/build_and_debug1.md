**目的**

本文件只记录镜像编译、镜像更新和更新后最小确认流程。具体模块实现、热更新细节和专项验收不要放在这里。

**环境**

Linux 编译环境：
（可通过对比本机IP是否在192.168.164或192.168.11(windows会话) 网段来判断是否应该用这套环境）

```bash
ssh mhc@192.168.164.2
```

测试宿主：

```bash
adb connect 192.168.11.45:5555
```

测试设备：

```bash
adb connect 192.168.11.45:5004
```

测试设备是运行在测试宿主上的容器化 AOSP；宿主是 Debian。

**编译镜像前确认**

如果 arm96 相关产物有更新，先同步到镜像实际依赖的 active prebuilts，再编完整镜像：

```bash
cd /home/mhc/56T/cx10/android10/vendor/hello/arm96
./make.sh --build cx
./sync_active_prebuilts.sh

cd /home/mhc/56T/cx10/android10
./make_image.sh --build mp
```

编译完成后记录镜像 md5，方便确认宿主拿到的是新包：

```bash
md5sum /home/mhc/56T/cx10/android10/out/target/product/sky1_evb/images/sky1_evb-10-user-super.img.tgz
find /home/mhc/56T/cx10/android10/IMAGE -name 'sky1_evb-10-user-super.img.tgz' -printf '%T@ %p\n' | sort -nr | head -3
```

**更新测试设备镜像**

默认流程：

```bash
adb connect 192.168.11.45:5555
adb -s 192.168.11.45:5555 shell 'cd /data/local && docker rm -f con4; ./update_image.sh; ./load_cx10_image.sh; ./b4_build_cx10_con4.sh'
```

如果 `update_image.sh` 因 scp 权限问题无法拉取镜像，直接 push 本地新镜像到宿主，再继续加载和重建容器：

```bash
adb -s 192.168.11.45:5555 push \
  /home/mhc/56T/cx10/android10/out/target/product/sky1_evb/images/sky1_evb-10-user-super.img.tgz \
  /data/local/sky1_evb-10-user-super.img.tgz

adb -s 192.168.11.45:5555 shell 'cd /data/local && md5sum sky1_evb-10-user-super.img.tgz 2>/dev/null || true'
adb -s 192.168.11.45:5555 shell 'cd /data/local && ./load_cx10_image.sh && ./b4_build_cx10_con4.sh'
```

**ADB 连接恢复**

`con4` 创建成功后，如果 `adb connect 192.168.11.45:5004` 返回 `Connection refused`，先从宿主确认容器是否已运行和端口是否映射：

```bash
adb -s 192.168.11.45:5555 shell 'docker ps -a | grep con4 || true'
adb -s 192.168.11.45:5555 shell 'docker inspect con4 --format "status={{.State.Status}} running={{.State.Running}} ports={{json .NetworkSettings.Ports}}" 2>&1 || true'
```

如果容器已运行且 5004 已映射到容器 5555，重启容器内 TCP adbd：

```bash
adb -s 192.168.11.45:5555 shell 'docker exec con4 /system/bin/sh -c "/system/bin/setprop service.adb.tcp.port 5555; /system/bin/setprop persist.adb.tcp.port 5555; /system/bin/setprop ctl.restart adbd"'
adb disconnect 192.168.11.45:5004
adb connect 192.168.11.45:5004
```
