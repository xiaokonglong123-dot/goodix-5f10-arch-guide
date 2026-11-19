# Goodix 27c6:5f10 on Arch Linux

在 Arch Linux 上启用 Goodix `27c6:5f10` 指纹传感器，并接入 `sudo` 与 `hyprlock`。本文记录一次实际安装与验证过程，包含失败路径、主要坑点和回退方式。

> 本仓库不包含设备 PSK、指纹模板、Windows 注册表文件或任何个人/主机信息。文中的磁盘、路径与用户名均为占位符。

## 结论

| 项目 | 结果 |
| --- | --- |
| 硬件 | Goodix `27c6:5f10`，USB 正常枚举 |
| 官方 `libfprint 1.94.100` | **不支持**该型号 |
| 社区驱动 | `Sbenazar/goodix-5f10-libfprint` 可识别并完成录入、验证 |
| 系统服务 | 使用官方 `fprintd 1.94.5` |
| 登录界面 | `ly` 保持密码登录 |
| `sudo` / `hyprlock` | 指纹优先，超时后回退密码 |
| PSK 来源 | 从本机 Windows 分区的 DPAPI 材料离线、只读恢复 |

本机环境（用于实测，非要求）：

```text
Arch Linux
Linux 7.3.1-1-cachyos
niri + ly + hyprlock
```

## 为什么官方驱动不行

`fprintd` 和 `libfprint` 安装后：

```text
$ fprintd-list "$USER"
No devices available
```

本机 `libfprint` 的 Goodix 驱动是 `goodixmoc`，支持 `5840`、`6090`、`63ac` 等型号，支持列表中没有 `5f10`。udev 也不会给该设备打上指纹支持标记。

继续使用官方驱动时，这个设备只是被 USB 枚举，不会被指纹认证框架接管。

## 驱动与限制

上游项目：

- 驱动：<https://github.com/Sbenazar/goodix-5f10-libfprint>
- PSK 恢复工具：<https://github.com/Sbenazar/goodix-5f10-psk>

使用的驱动提交：

```text
625c733837f05a24bcf81999b654b26ce5530313
```

该传感器是 56×176 图像传感器，不适合 NBIS 细节点匹配。驱动通过 TLS-PSK 与芯片通信，在主机侧使用 sigfm（FAST-9 + BRIEF-256 + RANSAC）匹配原始帧。

关键前提：

1. 每个传感器有独立的 32 字节 TLS-PSK。
2. 有 Windows 分区时，应优先从 Windows 驱动留下的 DPAPI 材料中只读恢复 PSK。
3. Linux-only 写入新 PSK 需要显式启用 `GOODIX5F10_AUTOPROVISION=1`，会改写芯片密钥；写入失败可能无法恢复，因为没有可用的 5F10 固件镜像。
4. 双系统下 Windows 驱动可能改写同一密钥槽，导致两边反复重写。上游已有相关 issue。

本仓库的流程**不使用**自动写入选项。

## 安装步骤

### 1. 安装基础组件

```sh
sudo pacman -S --needed fprintd
sudo pacman -S --needed meson ninja glib2-devel gobject-introspection \
  python-cairo python-gobject umockdev
```

构建依赖用于编译和测试；安装完成后可以移除，见“清理”。

`glib2-devel` 提供 `glib-mkenums`，缺少它时 Meson 会在配置阶段失败。

### 2. 构建完整驱动，而不是单驱动版本

不要用 `-Ddrivers=goodixtls5f10` 生成并安装库。那样构建出的 `libfprint` 只包含这一个驱动，会导致机器上的其他指纹设备全部失效。

正确做法是保留默认全驱动，只追加 5f10：

```sh
git clone https://github.com/Sbenazar/goodix-5f10-libfprint.git
cd goodix-5f10-libfprint
git checkout 625c733837f05a24bcf81999b654b26ce5530313

meson setup build --prefix=/usr --libdir=lib \
  -Ddoc=false -Dgtk-examples=false
ninja -C build

./build/libfprint/fprint-list-supported-devices | grep 27c6:5f10
```

应输出：

```text
27c6:5f10 | Goodix TLS Fingerprint Sensor 5F10
```

文档和 GTK 示例默认启用时会拉取 `gtk-doc` 等额外依赖，和驱动功能无关，因此关闭。

### 3. 打成可卸载的替换包

直接用 `ninja install` 会覆盖包管理器管理的文件，之后升级或回退难以审计。使用 [`packaging/PKGBUILD`](packaging/PKGBUILD) 构建独立包：

```sh
makepkg -s
sudo pacman -U libfprint-goodix5f10-*.pkg.tar.zst
```

包声明：

```text
provides=(libfprint libfprint-2.so)
conflicts=(libfprint)
```

因此会替换官方 `libfprint`，但 `pacman -R libfprint-goodix5f10` 可以干净移除，再重装官方包即可回退。

构建脚本只排除两个与 5f10 无关的测试：

- `drivers` suite：上游 `egis_etu905` 的 USB 回放在本机 umockdev 环境卡死。
- `metainfo-validate`：会访问 freedesktop.org，网络不可达时失败。

核心测试结果：

```text
Ok: 5
Fail: 0
Skipped: 2
```

其中包括 `goodix5f10-config`。

### 4. 恢复 PSK（只读）

先确认 Windows 分区。下面的设备名必须替换为本机实际分区：

```sh
lsblk -f
sudo mkdir -p /mnt/win
sudo mount -t ntfs3 -o ro,noexec,nosuid,nodev /dev/nvmeXn1pN /mnt/win
```

准备隔离的提取环境：

```sh
git clone https://github.com/Sbenazar/goodix-5f10-psk.git
cd goodix-5f10-psk
python3 -m venv venv
venv/bin/pip install impacket pycryptodome
```

`impacket 0.13.1` 的本地注册表解析器会以 `r+b` 打开 hive，只读挂载会直接失败：

```text
OSError: [Errno 30] Read-only file system
```

DPAPI 提取本身只读。将 venv 中的打开模式改为只读：

```sh
sed -i "s/open(hive,'r+b')/open(hive,'rb')/" \
  venv/lib/python*/site-packages/impacket/winregistry.py
```

这是一处本地兼容补丁，`pip` 升级 `impacket` 后会丢失，需要重新应用。

执行恢复，然后立即卸载分区：

```sh
sudo venv/bin/python install_psk.py --win-root /mnt/win
sudo umount /mnt/win
```

安装器会把密钥写到：

```text
/var/lib/fprint/goodix-5f10/psk
```

权限应为 `root:root`、`0600`。不要复制该文件、不要提交到仓库、不要跨机器复用；它是每台设备独有的密钥。

### 5. 验证设备、录入与匹配

```sh
fprintd-list "$USER"
```

实测结果：

```text
found 1 devices
Using device /net/reactivated/Fprint/Device/0
User ... has no fingers enrolled for Goodix TLS Fingerprint Sensor 5F10.
```

录入：

```sh
fprintd-enroll
```

完成后再次列出：

```text
Fingerprints for user ... on Goodix TLS Fingerprint Sensor 5F10 (press):
 - #0: right-index-finger
```

验证：

```sh
fprintd-verify
```

期望输出包含：

```text
Verify result: verify-match (done)
```

如果出现 `enroll-completed`，表示录入阶段已经完成，不是仍在持续采集，也不是错误。

## 接入 sudo 与锁屏

PAM 的认证栈是串行的，`pam_fprintd` 不能与密码输入同时监听。这里采用“指纹优先，10 秒超时后回退密码”。

修改前备份：

```sh
sudo cp --preserve=all /etc/pam.d/sudo /etc/pam.d/sudo.pre-fprint
sudo cp --preserve=all /etc/pam.d/hyprlock /etc/pam.d/hyprlock.pre-fprint
```

在 `include` 语句之前插入：

```pam
auth       sufficient                    pam_fprintd.so       timeout=10
```

以 `sudo` 为例，完整文件见 [`pam/sudo.example`](pam/sudo.example)：

```pam
#%PAM-1.0
auth       sufficient                    pam_fprintd.so       timeout=10
auth       include                       system-auth
account    include                       system-auth
session    include                       system-auth
session    optional                      pam_systemd.so class=none
```

`hyprlock` 完整文件见 [`pam/hyprlock.example`](pam/hyprlock.example)：

```pam
# PAM configuration for hyprlock
auth       sufficient                    pam_fprintd.so       timeout=10
auth       include                       login
```

测试 `sudo`：

```sh
sudo -k
sudo true
```

触摸已录入的手指。匹配成功时不要求密码；不触摸时约 10 秒后出现密码提示。

### 为什么没有接入 ly

`ly` 的登录流程先收集用户名和密码，再启动 PAM 会话。其 PAM conversation 只处理 `PAM_PROMPT_ECHO_ON` / `PAM_PROMPT_ECHO_OFF`，无法在指纹等待期间接受新的密码输入，`PAM_ERROR_MSG` 也会中止会话。

因此接入后可能出现“等待指纹期间无法改用密码”的体验问题。本机选择保留密码登录，不修改 `/etc/pam.d/ly`。

GDM 等图形登录管理器会为密码和指纹分别启动认证栈，行为不同；本方案不适用。

## 清理

驱动运行只需要 `fprintd` 和已安装的 `libfprint-goodix5f10`。构建依赖可移除：

```sh
sudo pacman -Rsu meson ninja glib2-devel gobject-introspection \
  python-cairo python-gobject umockdev
```

可以删除源码树、构建目录与 `venv`。保留以下内容：

- 已安装的驱动包或其可重建的 `PKGBUILD`
- `/var/lib/fprint/goodix-5f10/psk`
- `/etc/pam.d/*.pre-fprint` 备份

## 回退

```sh
sudo pacman -R libfprint-goodix5f10
sudo pacman -S libfprint
sudo cp --preserve=all /etc/pam.d/sudo.pre-fprint /etc/pam.d/sudo
sudo cp --preserve=all /etc/pam.d/hyprlock.pre-fprint /etc/pam.d/hyprlock
```

若要彻底移除指纹数据，先删除已录入指纹：

```sh
fprintd-delete "$USER"
```

再删除 PSK，并按需卸载 `fprintd`。

注意：如果与 Windows 双系统共用传感器，删除或更换 PSK 可能影响 Windows 侧使用同一密钥槽的指纹配置。

## 安全说明

- PSK 是设备身份密钥，不是可直接登录的密码；泄露它不等于泄露系统账户密码。
- 不要把 `Goodix_Cache.bin`、注册表 hive、DPAPI 主密钥或提取后的 PSK 上传到公开仓库。
- 恢复流程只需只读挂载 Windows 分区；不要启用 NTFS 写入。
- `pam_fprintd` 的 `sufficient` 只影响认证；`account`、`session` 仍沿用原有栈。
- 指纹不是唯一认证方式。本配置始终保留密码回退。

## 参考

- <https://github.com/Sbenazar/goodix-5f10-libfprint>
- <https://github.com/Sbenazar/goodix-5f10-psk>
- <https://fprint.freedesktop.org/>
- <https://wiki.archlinux.org/title/Fprint>
- <https://codeberg.org/fairyglade/ly>