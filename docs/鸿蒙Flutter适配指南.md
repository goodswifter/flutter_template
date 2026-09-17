# Flutter 支持鸿蒙（HarmonyOS / OHOS）适配指南

本文基于本机实际环境整理：`Flutter 3.41.10-ohos-1.0.1` + DevEco Studio。

---

## 一、前置条件

| 组件                | 说明                                                                                 |
| ------------------- | ------------------------------------------------------------------------------------ |
| 鸿蒙版 Flutter SDK  | 官方稳定版 Flutter **不支持** ohos，必须用社区/华为定制版（如 `3.41.10-ohos-1.0.1`） |
| DevEco Studio       | 安装并完成首次启动，带上 HarmonyOS SDK、ohpm、hvigor、node                           |
| 真机/模拟器（可选） | 调试时需要；先把 toolchain 配通即可                                                  |

本机 FVM 路径示例：

```text
/Users/zad/fvm/versions/3.41.10-ohos-1.0.1
```

---

## 二、配置环境变量（PATH）

`flutter` / `ohpm` / `hvigorw` / `node` 必须在 PATH 中，否则 `command not found` 或 `flutter doctor` 报缺工具。

将以下内容写入 `~/.zshrc`（按本机路径调整）：

```bash
# 鸿蒙 Flutter
export FLUTTER_OH_PATH="/Users/zad/fvm/versions/3.41.10-ohos-1.0.1/bin"

# DevEco 工具链
export OHPM_PATH="/Applications/DevEco-Studio.app/Contents/tools/ohpm/bin"
export HVIGORW_PATH="/Applications/DevEco-Studio.app/Contents/tools/hvigor/bin"
# HarmonyOS / DevEco
export DEVECO_SDK_HOME="/Applications/DevEco-Studio.app/Contents/sdk"

export PATH=$DEVECO_SDK_HOME:$FLUTTER_OH_PATH:$FLUTTER_PATH:$OHPM_PATH:$HVIGORW_PATH:$DART_PATH:$PUB_CACHE:$ANDROID_HOME:$FVM_PATH:$PATH
```

生效：

```bash
source ~/.zshrc
flutter --version   # 应显示带 ohos 的版本号
```

---

## 三、配置 HarmonyOS SDK 路径

### 正确路径（重要）

指向 **含** `default/` **目录的上一级**，不要指向 `default` 本身，也不要写 `~`：

```bash
flutter config --ohos-sdk=/Applications/DevEco-Studio.app/Contents/sdk
```

期望目录结构：

```text
.../Contents/sdk/
└── default/
    ├── openharmony/toolchains/hdc
    ├── hms/
    └── sdk-pkg.json
```

### 常见错误

| 错误写法               | 原因                                                                             |
| ---------------------- | -------------------------------------------------------------------------------- |
| `~/Library/Huawei/Sdk` | `~` 不会被 Flutter 展开；且该目录往往只有模拟器镜像，没有完整 API                |
| `.../sdk/default`      | Flutter 会在其**子目录**里找带 `sdk-pkg.json` 的版本目录，应指向上一级 `.../sdk` |

查看当前配置：

```bash
flutter config --list | grep ohos
```

---

## 四、校验环境

```bash
flutter doctor
```

目标：`HarmonyOS toolchain` 为 ✓，大致包含：

- HarmonyOS SDK 已识别
- ohpm / node / hvigorw 可用

可选预下载引擎：

```bash
flutter precache --ohos
```

---

## 五、给现有项目添加 ohos 平台

### 1. 新增平台目录

在项目根目录执行：

```bash
cd /path/to/your_project

# 若 Android / iOS 包名组织不一致，必须显式指定 --org
flutter create . --platforms=ohos --org com.example
```

本仓库曾出现：

```text
Ambiguous organization in existing files: {com.zad, com.example}.
The --org command line argument must be specified to recreate project.
```

原因：Android 为 `com.example.flutter_combat`，iOS 为 `com.zad.abc`。加 `--org` 即可（按你最终想用的组织名选择，例如 `com.example` 或 `com.zad`）。

### 2. 生成内容

成功后会出现 `ohos/`：

```text
ohos/
├── AppScope/app.json5       # 应用包名、版本
├── entry/                   # 主模块（ArkTS 入口）
├── build-profile.json5      # 编译 / 签名
├── hvigorfile.ts
└── oh-package.json5
```

> `ohos/` 是 Flutter 与鸿蒙原生的桥接层，勿随意删除核心文件。误删可重新执行上面的 `flutter create`。

### 3. 新建项目时直接带上 ohos

```bash
flutter create --platforms=android,ios,ohos --org com.example my_app
```

---

## 六、签名与运行

### 真机签名

1. 用 DevEco Studio 打开项目中的 `ohos/` 目录
2. **File → Project Structure → Signing Configs**，勾选自动签名并完成登录/证书
3. 回到终端：

```bash
flutter devices
flutter run -d <deviceId>
```

或先打包再安装：

```bash
flutter build hap --debug
# 产物一般在：
# ohos/entry/build/default/outputs/default/entry-default-signed.hap
hdc -t <deviceId> install <hap路径>
```

### 模拟器

1. DevEco 启动 HarmonyOS 模拟器
2. `flutter devices` 能看到设备后 `flutter run`

---

## 七、常用命令速查

| 场景            | 命令                                            |
| --------------- | ----------------------------------------------- |
| 配置 SDK        | `flutter config --ohos-sdk=<绝对路径>`          |
| 环境检查        | `flutter doctor`                                |
| 已有工程加 ohos | `flutter create . --platforms=ohos --org <org>` |
| 调试运行        | `flutter run -d <deviceId>`                     |
| 打 debug 包     | `flutter build hap --debug`                     |
| 打 release 包   | `flutter build hap --release`                   |

---

## 八、插件与兼容性注意

- 三方插件需要有 **ohos 实现**；没有则可能编译失败或功能缺失。
- 可到 [OpenHarmony-SIG](https://gitee.com/openharmony-sig) 查找对应 fork。
- Dart / Flutter 业务代码一般可复用；平台通道、原生插件需单独适配。

---

## 九、推荐执行顺序（清单）

1. 安装鸿蒙版 Flutter（或 FVM 切换到 `*-ohos-*`）
2. 安装 DevEco Studio，确认 SDK / ohpm / hvigor / node 可用
3. 配置 PATH 与 `flutter config --ohos-sdk=...`
4. `flutter doctor` 通过 HarmonyOS toolchain
5. `flutter create . --platforms=ohos --org <org>`
6. DevEco 打开 `ohos/` 完成签名
7. `flutter run` / `flutter build hap`

---

## 十、本机关键路径备忘

```text
Flutter(OHOS):  /Users/zad/fvm/versions/3.41.10-ohos-1.0.1
OHOS SDK:       /Applications/DevEco-Studio.app/Contents/sdk
ohpm:           /Applications/DevEco-Studio.app/Contents/tools/ohpm/bin
hvigorw:        /Applications/DevEco-Studio.app/Contents/tools/hvigor/bin
node:           /Applications/DevEco-Studio.app/Contents/tools/node/bin
hdc:            /Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/toolchains
```
