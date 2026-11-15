#Zynkeeper
# Zyn消息防撤回
**注：源码都在release！**
- 一个纯本地运行的安卓「消息撤回查看器」：通过通知监听把收到的消息先存一份，等微信 / QQ 出现「XX 撤回了一条消息」时，用正则认出撤回者，再在同一会话、同一发送者的回溯窗口里反查出他撤回前的那条内容。

- 包名：`com.zyn.keeper` · 当前版本 **1.0.9**（versionCode 11）
- 最低 **Android 5.0（API 21）**，编译 / 目标 SDK 34（Android 14）
- 语言 Kotlin + Jetpack Compose + Material3，存储用原生 SQLite
- 数据只存在本机 `zyn.db`；联网仅用于检查更新与下载新版本，不上传任何聊天内容
- 界面采用米白底 + 陶土橙 + 衬线标题的极简风格

---

## 功能一览

### 撤回还原
- 通知落库后，识别「撤回提示」正则，提取撤回者昵称
- 在回溯窗口内反查该用户上一条消息，还原被撤回的内容
- 撤回时间线：应用、会话、撤回者、被撤回内容、命中规则，可复制 / 删除
- 没抓到原文的撤回也会记一条，并标注「未捕获到原消息」

### 消息历史
- 监听开启期间收到的全部通知留档，支持按会话 / 用户 / 内容搜索
- 消息左侧显示聊天对象头像（抓不到时用首字母圆点兜底），头像按指纹去重并自动回收

### 识别规则（可编辑 + 可测试）
- 内置微信、QQ 两条默认开启规则；企业微信、钉钉、Telegram 三条默认关闭
- 每条规则可改：包名正则、撤回提示正则、撤回者分组、发送者正则、发送者 / 内容分组、自己正则
- 内置正则测试框：改完立刻喂样例文本验证，命中后直接显示提取到的撤回者
- 正则覆盖带昵称、被动句、英文界面、QQ 自定义后缀与图片 / 语音 / 视频 / 文件 / 表情等写法

### 提醒方式
- **悬浮窗卡片**：米白卡片 + 陶土橙标题，停留时间、不透明度可调
- **通知栏提醒**：提示音、震动、是否显示内容均可单独开关
- **B 站弹幕样式**（可选）：撤回提示以弹幕形式从右到左滑过屏幕顶部，字号、颜色、速度可调

### 娱乐玩法（默认关闭）
- 开启后，所有抓到的消息都以弹幕形式滚过屏幕
- 同一句话重复出现会合并显示成「内容 ×2、×3」，与 B 站重复弹幕一致
- 弹幕参数与悬浮窗弹幕各自独立；关闭开关会立刻清屏

### 后台保活（七路叠加）
| 通道 | 实现 |
|---|---|
| 前台服务 | `KeepAliveService`，`specialUse` 类型，`stopWithTask=false`，常驻通知 LOW 优先级 |
| 精确闹钟 | 单次 `setExactAndAllowWhileIdle` 链，触发后自续；无权限退回 `set` |
| WorkManager | 15 分钟心跳，顺带数据清理与断连提示 |
| JobScheduler | `GuardJobService`，15 分钟周期、`persisted`，重启后仍有效 |
| 系统同步 | 自带账号 + `SyncAdapter`（独立 `:sync` 进程），系统每 10 分钟唤醒一次 |
| 1 像素悬浮窗 | 1×1、全透明、不可触摸，抬高进程优先级 |
| 无障碍保活 | `KeepAliveAccessibilityService`，由系统托管、被杀后系统主动重启；`canRetrieveWindowContent=false`，不读取屏幕内容 |

另有开机 / 应用更新 / 任务划掉三个广播接收器补唤醒，唤醒时持 10 秒 WakeLock。通知监听掉线后由 `ListenerGuard` 自动重绑（`requestRebind` + 组件开关兜底 + 1s / 5s / 15s 三次重试）。

---

## 安装

1. 到 [Releases](https://github.com/zynsync/zynkeeper/releases/latest) 下载最新 APK，直接覆盖安装即可（**不要卸载**，卸载会清空本地历史）。
2. 首次打开进入强制引导，按顺序授予：
   1. **通知使用权** —— 系统把通知文本交给本应用，才有数据可反查
   2. **悬浮窗权限** —— 顶部弹卡、弹幕与 1 像素保活窗都依赖它
   3. **电池优化白名单** —— 否则息屏几分钟后监听可能被系统掐掉
   4. **无障碍保活** —— 系统托管的强绑定；本服务只监听窗口切换维持存活，不读取任何屏幕内容
3. 小米 / 华为 / OPPO / vivo 等机型，还需在系统「自启动 / 后台运行」里允许本应用（引导页底部有直达按钮）。
4. 微信 / QQ 内打开「显示消息详情」「锁屏通知显示内容」，否则通知正文可能被折叠成「你收到一条新消息」。

> 引导页连点三次「重新检测」后会出现跳过入口，但跳过任何一项都会让捕获或保活打折。

## 权限与隐私

| 权限 | 用途 |
|---|---|
| 通知使用权（系统授权） | 读取通知标题与正文用于本地留档和撤回还原 |
| 悬浮窗 `SYSTEM_ALERT_WINDOW` | 撤回卡片、弹幕、1 像素保活窗 |
| 电池优化白名单（系统授权） | 保持后台进程不被省电策略清理 |
| 无障碍（系统授权，可选但强烈建议） | 仅用于保活，不读取窗口内容、不监听按键 |
| `RECEIVE_BOOT_COMPLETED` | 开机后自动恢复监听 |
| `WAKE_LOCK` | 唤醒后短暂持锁，确保守护服务真正启动 |
| `SCHEDULE_EXACT_ALARM` | 精确闹钟链（低版本自动降级） |
| `INTERNET` / `ACCESS_NETWORK_STATE` | 仅用于检查更新、下载新版本 |
| `REQUEST_INSTALL_PACKAGES` | 应用内下载完成后调起系统安装器 |
| `QUERY_ALL_PACKAGES` | 把包名显示成可读的应用名 |

- 所有聊天记录、头像、设置都保存在本机数据库与应用私有目录，不上传、不分享、不做分析。
- 联网只发生在「检查更新」和「下载新版本」两个动作里。

## 从源码编译

要求：JDK 17、Android SDK 34、Build Tools 34.0.0、Gradle 8.5（AGP 8.3.2，仓库内已配置阿里云 Maven 镜像）。

```bash
git clone https://github.com/zynsync/zynkeeper.git
cd zynkeeper
./gradlew assembleDebug
# 产物：app/build/outputs/apk/debug/app-debug.apk
```

Release 包需要在 `build.gradle.kts` 自行配置正式签名；仓库默认产出的是 debug 签名包。

更新检测源固定在 `UpdateChecker.DEFAULT_URL`：

```
https://api.github.com/repos/zynsync/zynkeeper/releases/latest
```

必须使用 `api.github.com` 的 JSON 接口；`github.com/.../releases` 网页地址是 HTML，解析不出版本信息。

## 工作原理

```
通知栏 → NotificationListenerService（捕获）
        ├─ 普通消息：昵称 + 内容 → 落库（notices 表）
        └─ 撤回提示：正则命中「XX 撤回了一条消息」
                      └─ 取出「XX」
                          └─ 同一会话、同一发送者、回溯窗口内
                              └─ 查最近一条消息 → 生成撤回事件（recalls 表）
                                  └─ 悬浮窗 / 通知栏 / 弹幕提醒
```

安卓系统「通知历史」数据库属于系统应用，第三方没有读取权限；监听只能拿到当前活跃通知。所以本应用自建了一份本地历史：**监听开启期间**收到的通知才会留档，监听关闭那段时间发生的撤回，任何方案都还原不了。

## 目录结构

```
ZynKeeper/
├─ app/src/main/java/com/zyn/keeper/
│  ├─ ZynApp.kt                  应用入口、通知渠道
│  ├─ data/                      SettingsStore / ZynDb / RuleRepository / Models
│  ├─ core/                      RecallEngine / ListenerGuard / UpdateChecker / Perm / Exporter / AvatarStore
│  ├─ service/                   CaptureService / KeepAlive / FloatingAlert / DanmakuOverlay / AlertNotifier
│  ├─ sync/                      账号 / 认证器 / SyncAdapter（系统同步保活）
│  └─ ui/                        Compose 界面（撤回 / 历史 / 规则 / 设置 / 状态 + 关于 / 样式 / 玩法）
├─ app/src/main/res/xml/         SyncAdapter、Authenticator、无障碍服务配置
├─ app/src/main/AndroidManifest.xml
└─ build.gradle.kts / settings.gradle.kts
```

## 已知限制

1. **系统通知历史读不到**：第三方应用无权访问，旧消息无法补录，只能从开启监听后开始积累。
2. **微信私聊撤回往往不发通知**：微信撤回时会撤销原通知，不一定再发一条撤回提示；群聊通常有。这是微信侧行为，正则无法变出数据。
3. **部分 ROM 会折叠通知正文**：MIUI / ColorOS 等在「通知折叠」「隐藏消息内容」开启时，监听只能拿到「你收到一条消息」。
4. **同名 / 昵称歧义**：群内重名或昵称带空格时可能归属出错，属正则固有局限，可在规则里自行收紧。
5. **无障碍权限可能被系统自动撤回**：部分 ROM 的高级保护模式会关掉它，需要手动重开；其余六条保活通道不受影响。
6. **本工具仅用于自己手机上、自己收到的消息留档**，请遵守微信 / QQ 用户协议与当地法律。
