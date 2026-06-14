# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目定位

本仓库是 **Tango ARM32→ARM64 二进制翻译系统的逆向复刻与加固工程**，核心任务是对原始二进制进行"黑盒"安全加固，消除逆向路标，提升分析与篡改成本。

**主线目录**: `android10/vendor/hello/arm96/`（旧路径 `android10/vendor/vclusters/opensource/thirdpart/tango_reimpl/` 已废弃，仅作 hardening/构建/验收工作区）

详细当前状态见 `.claude/context/tango_hardening.md`，详细构建/调试模块说明见 `android10/vendor/hello/arm96/CLAUDE.md` 与 `android10/vendor/hello/arm96/debug.md`。

---

## 双开发环境（按本机 IP 网段自动选择）

本项目有两套独立的编译/测试环境，**根据当前 Windows/Linux 会话主机的 IP 网段**选择对应一套，详见 `.claude/instructions/build_and_debug1.md`（环境1）和 `.claude/instructions/build_and_debug2.md`（环境2）：

| | 环境1 | 环境2 |
|---|---|---|
| 触发网段 | `192.168.164.x` 或 `192.168.11.x` | `192.168.2.x` |
| 编译机 | `ssh mhc@192.168.164.2` | `ssh mc@192.168.2.18` |
| 仓库路径 | `/home/mhc/56T/cx10` | `/home/mc/2T/cx10` |
| 测试宿主 | `adb connect 192.168.11.45:5555` | `adb connect 192.168.2.19:5555` |
| 测试设备 | `adb connect 192.168.11.45:5004` | `adb connect 192.168.2.19:5004` |
| 容器重建脚本 | `b4_build_cx10_con4.sh` | `mb4_build_cx10_con4.sh` |

判断方法：在当前会话执行 `ipconfig`/`ip route` 查看本机 IP，落在哪个网段就用哪套环境。

### SSH 执行约定（已验证）

编译机已配置免密公钥登录，可直接非交互执行：

```bash
ssh -o BatchMode=yes mc@192.168.2.18 '<command>'   # 环境2
ssh -o BatchMode=yes mhc@192.168.164.2 '<command>' # 环境1（未验证）
```

### 完整镜像编译+更新

```bash
cd <仓库路径>/android10/vendor/hello/arm96
./make.sh --build cx
./sync_active_prebuilts.sh

cd <仓库路径>/android10
./make_image.sh --build mp

# 设备侧：docker rm -f con4 && update_image.sh（或手动 push tgz）→ load_cx10_image.sh → <对应容器重建脚本>
```

### 快速 debug：只更新六件套并重启服务（无需重编整镜像）

`arm96` 内部各组件相互 md5/hash 校验，**六件套必须同批构建、整套替换，不能只换其中一个**。具体哪几件是密钥融合硬耦合（arm96/arm96server/arm96d/libarm96）、哪几件是松耦合（arm96_sidecar_gen/arm96j.jar），见 `android10/vendor/hello/arm96/CLAUDE.md` 的"密钥派生链 / 六件套哈希耦合关系"小节。

```bash
cd <仓库路径>/android10/vendor/hello/arm96
./make.sh --build cx

# 读取本次构建的实际输出目录（OUT_BIN 所在目录），不要硬编码 cx/bin
cat build/last_build.env
# 示例输出：
#   BUILD_TARGET=cx
#   OUT_BIN=<仓库路径>/android10/vendor/hello/arm96/build/out/cx/bin/arm96
#   VERSION=V0.69.0
```

六件套文件名（在 `$(dirname OUT_BIN)` 目录下，已实测确认）及设备端目标路径：

| 文件名 | 设备路径 |
|---|---|
| `arm96` | `/system/bin/arm96` |
| `libarm96` | `/system/bin/libarm96` |
| `arm96server` | `/system/bin/arm96server` |
| `arm96d` | `/vendor/bin/arm96d` |
| `arm96_sidecar_gen` | `/vendor/bin/arm96_sidecar_gen` |
| `arm96j.jar` | `/system/framework/arm96j.jar` |

注：该目录下还可能残留 `*.preencrypt*` 等中间产物，push 时只取上表六个文件名，不要整目录 push。

```bash
OUT=$(dirname $(grep OUT_BIN= build/last_build.env | cut -d= -f2))
SER=192.168.2.19:5004   # 环境2测试设备

adb connect $SER
adb -s $SER push $OUT/arm96             /system/bin/arm96
adb -s $SER push $OUT/libarm96          /system/bin/libarm96
adb -s $SER push $OUT/arm96server       /system/bin/arm96server
adb -s $SER push $OUT/arm96d            /vendor/bin/arm96d
adb -s $SER push $OUT/arm96_sidecar_gen /vendor/bin/arm96_sidecar_gen
adb -s $SER push $OUT/arm96j.jar        /system/framework/arm96j.jar

adb -s $SER shell '
  chmod 755 /system/bin/arm96 /system/bin/libarm96 /system/bin/arm96server /vendor/bin/arm96d /vendor/bin/arm96_sidecar_gen
  chmod 644 /system/framework/arm96j.jar
  stop arm96j
  stop arm96d
  setprop ctl.restart zygote_secondary
  start arm96d
  start arm96j
'

adb -s $SER shell 'sh /data/local/tmp/verify_mustpass.sh'
```

`arm96j`/`arm96d` 是通过 `.rc` 注册的 service（`src/arm96j/init.arm96j.rc`、`src/arm96d/init.arm96d.rc`），用 `stop`/`start` 重启，不要只重启 `zygote_secondary`。

### 已知版本差异提示

2026-06-14 实测环境2构建产出 `VERSION=V0.69.0`，高于 `.claude/context/tango_hardening.md` 中记录的当前候选 `V0.67.0`/规划中的 `V0.68`。说明 V0.68/V0.69 已落地但上下文文档尚未同步——开新会话涉及版本号判断时，应以 `build/last_build.env` 的实测值为准，必要时核对/更新 `tango_hardening.md`。

---

## 构建命令

### 主线加固构建（hardening track）

```sh
# 标准构建
cd android10/vendor/vclusters/opensource/thirdpart/tango_reimpl
./make.sh

# 指定 profile（vc/cs/myt）
./make.sh --build vc

# 5 分钟评估版（过期 300s，interval 180s）
./make_5min.sh
./make_5min.sh --build myt

# 打包产物
bash shell/package.sh
# 输出：build/last_build.env 记录 BUILD_TARGET/VERSION/OUT_BIN
```

产物输出到 `build/out/<target>/bin/`（含 `arm96`/`libarm96`/`arm96server` 三件套）。

### AOSP 镜像构建

```sh
cd android10/
./make_image.sh --build mp       # 全编
source build/envsetup.sh >/dev/null && lunch sky1_evb-user
make arm96_translator_reimpl     # 单独编译 C++ 源码模块
```

### C++ 源码模块独立编译

```sh
cd android10/vendor/vclusters/opensource/thirdpart/tango_reimpl
./build_arm96_reimpl_arm64.sh     # arm96 runtime
./build_tangolic_reimpl_arm64.sh  # 授权服务
./build_tango_pretranslator_reimpl_arm64.sh  # 预翻译器
```

### 设备侧验证

```sh
# adb 连接约定（自动根据当前主机网段选择 A 组或 B 组）
SER_A=192.168.11.45:5004;  HOST_SER_A=192.168.11.45:5555   # A 组
SER_B=192.168.2.45:5004;   HOST_SER_B=192.168.2.45:5555    # B 组

# 自动选择：与当前主机同网段者优先（A 组 192.168.11.x，B 组 192.168.2.x）
HOST_IP=$(ip route get 1 2>/dev/null | grep -oP 'src \K[\d.]+' || hostname -I | awk '{print $1}')
if echo "$HOST_IP" | grep -q "^192\.168\.2\."; then
  SER="$SER_B"; HOST_SER="$HOST_SER_B"
elif echo "$HOST_IP" | grep -q "^192\.168\.11\."; then
  SER="$SER_A"; HOST_SER="$HOST_SER_A"
else
  SER="$SER_A"; HOST_SER="$HOST_SER_A"   # 默认 A 组
fi

# 推送更新（每次更新 arm96/libarm96/arm96server 后必须重启 zygote_secondary）
adb connect "$SER"
adb -s "$SER" push build/out/vc/bin/arm96 /data/local/tmp/arm96.new
adb -s "$SER" push build/out/vc/bin/libarm96 /data/local/tmp/libarm96.new
adb -s "$SER" push build/out/vc/bin/arm96server /data/local/tmp/arm96server.new
adb -s "$SER" shell '
  cp -f /data/local/tmp/arm96.new /system/bin/arm96 && chmod 755 /system/bin/arm96
  cp -f /data/local/tmp/libarm96.new /system/bin/libarm96 && chmod 755 /system/bin/libarm96
  cp -f /data/local/tmp/arm96server.new /system/bin/arm96server && chmod 755 /system/bin/arm96server
  setprop ctl.restart zygote_secondary
'

# 设备内验收（必须在容器内执行）
adb -s "$SER" shell 'sh /data/local/tmp/verify_mustpass.sh'
```

---

## 架构概览

### 三组件 Split 模型

```
arm96（loader，静态链接 C）← 反调试、平台绑定、密钥派生
  ├─ 解密 libarm96（AES-128-GCM，memfd 执行）
  └─ 启动 arm96server（daemon，PID 追踪反调试）

libarm96（加密 core）← 原始 tango_translator 二进制经 patch/加密

arm96server（daemon）← 扫描 32 位进程，PID 追踪 + TracerPid 检测
```

**binfmt_misc flags 必须为 `POC`**（含 `P` = preserve-argv0）。

### 密钥派生链（5 层融合）

```
base_key ⊕ ISAR0 ⊕ MAC_HASH ⊕ EXPIRE_MASK ⊕ arm96server_token
```

每层将硬件/授权特征融入 AES-128-GCM 解密密钥；攻击者无法通过单一 NOP 绕过。

### 反调试机制层次

1. arm96server 追踪 UID 10000–19999 的 32 位进程，TracerPid > 0 即杀
2. loader 启动时 scan `/proc/self/maps` 检测 frida/gadget/xposed/lsposed
3. loader `.text` 段 SHA-256 自校验，sentinel magic 为 HMAC 派生值
4. 空壳检测：launch 后 200ms 探测 arm96server 是否 TRACEME，未做则下毒

### 加固版本状态（以 `.claude/context/tango_hardening.md` 为准）

- **当前发布版本**: `V0.49.0`（2026-05-03）
  - 输入指纹门禁（E4901）+ patch 前镜像断言（E4902）+ patch 后白名单核验（E4903）+ orig_size 一致性校验（E4905）
  - 必过验收：`12 PASS / 0 FAIL / 0 SKIP`
- **下一版本**: V0.50（周期 challenge-response 持续认证）
- 详细迭代历史见 `.claude/context/tango_hardening.md`

---

## 分支边界（关键约束）

- `android10/vendor/vclusters/opensource/thirdpart/tango_reimpl/src/` 是 C++ 源码主线区域
- `main/1.0` 分支：只做 hardening、验收链、公共文档维护；**禁止修改** `src/tango_translator_reimpl/` 目录结构
- `main/2.0` 分支：负责 C++ 源码持续迭代和目录级收敛
- 当前交付名已从 `tango_translator` 收敛为 `arm96`（V0.34+）
- 源码模块主名为 `arm96_translator_reimpl`，兼容目标 `tango_translator_reimpl`（`src/` 目录名不变）

---

## 核心源码模块

| 模块 | 源码目录 | 产物 | 功能 |
|------|---------|------|------|
| arm96（loader） | `hardening/src/loader.c` | `/system/bin/arm96` | 反调试、密钥派生、memfd 解密执行 |
| arm96server | `hardening/src/arm96server.c` | `/system/bin/arm96server` | PID 追踪、Guardian 双向互保 |
| libarm96（core） | `hardening/src/main.py`（patch） | `/system/bin/libarm96` | 经 patch 的原始翻译器 |
| tangolic | `src/tangolic_reimpl/` | 历史 prebuilt | 授权服务（V0.33+ 最小镜像已移除） |
| pretranslator | `src/tango_pretranslator_reimpl/` | 历史 prebuilt | 静态预翻译（V0.33+ 最小镜像已移除） |

---

## 设备管理

根据当前主机 IP 网段自动选择：
- A 组：容器 `adb connect 192.168.11.45:5004` / 宿主 `adb connect 192.168.11.45:5555`
- B 组：容器 `adb connect 192.168.2.45:5004` / 宿主 `adb connect 192.168.2.45:5555`（当前同网段）

容器起不来时：在宿主运行 `/data/local/b4_build_cx10_con4.sh` 重新构建镜像
- 容器内 binfmt 注册：`echo ":arm96:M::\\x7fELF\\x01\\x01\\x01\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x02\\x00\\x28\\x00:\\xff\\xff\\xff\\xff\\xff\\xff\\xff\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\xfe\\xff\\xff\\xff:/system/bin/arm96:POC" > /proc/sys/fs/binfmt_misc/register`

---

## 必过用例快速验收（TC-001 ~ TC-010）

| 用例 | 命令 | 预期 |
|------|------|------|
| TC-001 | `arm96 -v` | 仅 1 行版本号（如 `V0.41`） |
| TC-001b | `arm96 -i` | 仅 2 行授权信息 |
| TC-002 | `arm96 -h` 或 `arm96` | 0 字节输出 |
| TC-003 | `arm96 -a` | 0 字节输出 |
| TC-004 | reboot（授权期内） | zygote_secondary 正常启动 |
| TC-005 | 启动 32 位应用 | 持续运行 |
| TC-006 | 启动 32 位应用（已过期） | 周期性 SIGALRM 杀进程 |
| TC-007 | reboot（已过期） | 系统正常启动，无 crash |
| TC-008 | `kill -9 arm96server` | 所有 32 位应用立即退出 |
| TC-009 | `strace -p <app_pid>` | EPERM |
| TC-010 | `strace -p <arm96server_pid>` | EPERM |

设备内自动化入口：`verify_mustpass.sh`；宿主侧编排入口：`verify_mustpass_host.sh`（覆盖 reboot 类用例）。

---

## 快速代码定位

关键函数位置：
- 反调试入口：`hardening/src/loader.c` — `antidebug_early_check()`
- 密钥派生：`hardening/src/loader.c` — `reconstruct_key()`
- 平台绑定（5 层检查）：`hardening/src/loader.c` — `platform_check()`
- ARM96server 追踪逻辑：`hardening/src/arm96server.c` — `scan_and_seize()`
- 加固 patch 入口：`hardening/src/main.py` — `patch_binary()`
- 加密入口：`hardening/src/encrypt_core.py`
- 版本配置中心：`hardening/src/version.txt`
- Syscall 添加同时更新 `src/tango_translator_reimpl/syscalls.h`（结构定义）和 `src/tango_translator_reimpl/syscall_handler.cpp`（逻辑实现）

---

## 反编译支持（ida-mcp）

ida-mcp 是运行在 Windows 侧的服务，提供二进制反编译能力。使用前需确认服务已启动，服务不可用时提醒用户手动启动。

**服务地址**（SSE 流式端点）：
```sh
# A 组：http://192.168.11.26:8745/sse
# B 组：http://192.168.2.2:8745/sse（当前同网段）
```

**手动启动命令**（在 Windows 侧执行）：
```cmd
# 启动 ida-mcp-server
uv run idalib-mcp --host <本机IP> --port 8745 D:\WorkScript\ida-mcp\tango\<binary>
```

二进制路径示例：`tango_pretranslator.arm64`、`tango_translator`、`tangolic`
