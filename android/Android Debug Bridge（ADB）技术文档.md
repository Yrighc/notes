
## 1. 概述

Android Debug Bridge（ADB）是 Android SDK Platform-Tools 提供的命令行工具，用于在主机（电脑）与 Android 设备（真机/模拟器）之间建立调试通信通道。常用于应用安装、日志抓取、Shell 命令执行、文件传输、端口转发、截图录屏与基础排障。

## 2. 架构与工作原理

ADB 由三部分协作完成指令链路：

- Client：命令行执行的 `adb` 程序
- Server：主机侧后台服务进程（通常监听 TCP 5037）
- Daemon（adbd）：设备侧守护进程，负责接收并执行指令

基本流程：

1) Client 发送命令到主机侧 Server  
2) Server 选择目标设备并建立/复用连接  
3) Server 与设备侧 adbd 通信，由 adbd 执行对应操作并返回结果

## 3. 环境准备

### 3.1 安装 ADB（Platform-Tools）

ADB 随 Android SDK Platform-Tools 发布。安装完成后应满足：

```bash
adb version
```

可输出版本信息。

### 3.2 设备侧设置（真机）

需要开启以下开关：

- 开发者选项（通常通过“关于手机”连续点击“版本号”开启）
- USB 调试（Developer options → USB debugging）

首次连接主机时会出现 USB 调试授权弹窗，未授权时 `adb devices` 通常显示 `unauthorized`。

### 3.3 连接方式

- USB：稳定、推荐作为首次调试链路
- Wi‑Fi：便捷但链路更复杂，需注意安全边界

## 4. 基础命令

### 4.1 设备枚举与状态

```bash
adb devices -l
```

常见状态：

- `device`：可用
- `unauthorized`：未完成调试授权
- `offline`：连接异常或链路不稳定

### 4.2 重启 ADB Server

```bash
adb kill-server
adb start-server
```

### 4.3 选择目标设备

当存在多个设备/模拟器时，使用序列号定向：

```bash
adb -s <serial> shell
```

辅助选项：

- `-d`：仅选择 USB 设备
- `-e`：仅选择模拟器

## 5. Shell 操作

### 5.1 进入交互式 Shell

```bash
adb shell
```

### 5.2 执行单条命令

```bash
adb shell getprop ro.build.version.release
```

常用系统工具（在设备 Shell 内）：

- `getprop`：系统属性查询
- `pm`：Package Manager（包管理）
- `am`：Activity Manager（组件启动/广播等）
- `settings`：系统设置读写
- `dumpsys`：系统服务状态信息汇总输出

## 6. 应用安装与卸载

### 6.1 安装 APK

```bash
adb install path/to/app.apk
```

覆盖安装（保留数据）：

```bash
adb install -r path/to/app.apk
```

允许降级安装（版本号降低时）：

```bash
adb install -r -d path/to/app.apk
```

### 6.2 卸载

```bash
adb uninstall com.example.app
```

卸载但尝试保留数据（行为依 ROM/版本而异）：

```bash
adb uninstall -k com.example.app
```

## 7. 包信息查询

列出所有包名：

```bash
adb shell pm list packages
```

查询包对应 APK 路径：

```bash
adb shell pm path com.example.app
```

## 8. 文件传输

推送（主机 → 设备）：

```bash
adb push ./local.txt /sdcard/Download/local.txt
```

拉取（设备 → 主机）：

```bash
adb pull /sdcard/Download/local.txt ./local.txt
```

说明：

- `/sdcard/` 常对应用户可访问的共享存储区域（不同版本/ROM 细节可能不同）
- 受分区存储（Scoped Storage）与权限策略影响，部分目录可能不可写/不可读

## 9. 日志系统（logcat）

实时查看：

```bash
adb logcat
```

清空缓冲区：

```bash
adb logcat -c
```

按 Tag 过滤：

```bash
adb logcat -s ActivityManager
```

导出一次性快照：

```bash
adb logcat -d > logcat.txt
```

排障关键词常见包括：`FATAL EXCEPTION`、`ANR`、`Crash`、`Exception`。

## 10. 系统状态查询（dumpsys）

查看前台 Activity（输出格式随版本差异较大）：

```bash
adb shell dumpsys activity activities | grep mResumedActivity
```

也可直接查看完整输出：

```bash
adb shell dumpsys activity activities
```

## 11. 截图与录屏

截图（保存到设备后拉回主机）：

```bash
adb shell screencap -p /sdcard/Download/screen.png
adb pull /sdcard/Download/screen.png ./screen.png
```

录屏：

```bash
adb shell screenrecord /sdcard/Download/demo.mp4
adb pull /sdcard/Download/demo.mp4 ./demo.mp4
```

说明：录屏时长限制与编码参数随系统实现而变化。

## 12. 端口转发

### 12.1 forward（主机端口 → 设备端口）

```bash
adb forward tcp:8080 tcp:8080
```

### 12.2 reverse（设备端口 → 主机端口）

```bash
adb reverse tcp:8080 tcp:8080
```

查看已设置规则：

```bash
adb forward --list
```

删除规则：

```bash
adb forward --remove tcp:8080
adb reverse --remove tcp:8080
```

## 13. Wi‑Fi 调试

### 13.1 传统 tcpip 模式（需同一局域网，安全风险更高）

切换设备到 TCP 监听（示例端口 5555）：

```bash
adb tcpip 5555
```

连接设备：

```bash
adb connect <phone_ip>:5555
```

切回 USB 模式：

```bash
adb usb
```

安全注意：tcpip 模式会暴露调试端口，不应在不可信网络环境启用。

### 13.2 Android 11+ Wireless Debugging（配对模式）

一般流程为设备侧开启无线调试并显示配对信息；主机侧使用：

```bash
adb pair <ip>:<pair_port>
adb connect <ip>:<connect_port>
```

端口取值以设备界面提示为准。

## 14. 授权与权限边界

- ADB 授权：主机是否被设备信任（决定 `adb` 是否可用）
- 系统/应用权限：应用运行权限与系统策略（决定具体资源是否可读写）
- Root 能力：多数量产机型不支持 `adbd` 提升为 root；出现 `adbd cannot run as root in production builds` 属正常现象

## 15. 常见问题与处理

### 15.1 `adb devices` 无设备

- 数据线仅充电不传输：更换可传输数据线
- USB 端口/Hub 不稳定：直连主机接口
- 设备连接模式异常：切换为文件传输（MTP）等
- 重启 ADB Server：`adb kill-server && adb start-server`

### 15.2 `unauthorized`

- 解锁设备并确认授权弹窗
- 开发者选项中撤销 USB 调试授权后重连
- 重启 ADB Server 并重新插拔

### 15.3 `offline`

- 重新连接/更换 USB 线与端口
- 重启 ADB Server
- 必要时重启设备

### 15.4 安装失败（INSTALL_FAILED_*）

常见原因包括：签名不一致、版本冲突、存储不足、ABI 不匹配。可通过卸载后重装验证：

```bash
adb uninstall com.example.app
adb install path/to/app.apk
```

## 16. 最小命令集（高频）

```bash
adb version
adb devices -l
adb kill-server
adb start-server
adb shell
adb install -r app.apk
adb uninstall com.example.app
adb logcat
adb push local /sdcard/Download/
adb pull /sdcard/Download/remote ./
```
