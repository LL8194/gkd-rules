# GKD 规则：HyperOS 安装弹窗与通知

针对 HyperOS / 小米，处理「用 Shizuku / InstallerX 之类静默装应用」时冒出来的两处拦截；
另外附带日常烦人广告的规则：钉钉（开屏 + 首页横幅推广卡）、影梭更新提示、微信小程序广告浮层。

## 订阅链接（GKD → 订阅 → 右下角 + 粘贴）

```
https://raw.githubusercontent.com/LL8194/gkd-rules/main/gkd.json5
```

国内直连不通时用加速（前两个当前实测可用）：

```
https://ghfast.top/https://raw.githubusercontent.com/LL8194/gkd-rules/main/gkd.json5
https://gh-proxy.com/https://raw.githubusercontent.com/LL8194/gkd-rules/main/gkd.json5
https://cdn.jsdelivr.net/gh/LL8194/gkd-rules@main/gkd.json5
```

> `ghproxy.net` 有缓存，会一直返回旧版本，别用它。

## 六条规则分别干什么

### ① 系统界面 `com.android.systemui` — 设备管理员安装提示-自动清除

静默安装完成后，系统会发一条「由您的管理员安装」的通知。这个通知渠道
`DEVICE_ADMIN_ALERTS` 的 `mBlockableSystem=false`、`mImportanceLockedDefaultApp=true`，
**系统层面禁止用户屏蔽**，只能等 72 小时自动消失。

所以这条规则不屏蔽、只清除：命中正文后水平滑出去，效果等同手动右滑。
`actionCd: 2000` 表示 2 秒内不重复触发，避免连点。

### ② 安全中心 `com.miui.securitycenter` — USB安装提示-自动继续安装

弹窗来自 `com.miui.permcenter.install.AdbInstallVerifyActivity`，底下有 10 秒倒计时。
三个候选系统键（`security_adb_install_enable`、`adb_install_packages`、
`perm_adb_install_notify`）在 secure / global / system 三个命名空间里都是 `null`，
所以 ADB 侧关不掉，只能替你把按钮点掉。

匹配做了双重保险：必须同时出现「继续安装」按钮 **和** 「正在通过USB安装此应用」提示语，
并限定弹窗出现后 8 秒窗口内尝试，避免在别的界面误点同文案按钮。

### ③ 钉钉 `com.alibaba.android.rimet` — 开屏广告

开屏广告是第三方品牌投放的大图（右下/右上角带「跳过(3)」倒计时）。这条在 App 启动后
20 秒窗口内，命中「跳过」开头的短文本就点掉，`actionMaximum: 1` 保证每个生命周期只动一次，
不会在聊天页误点同文案元素。

未实测到广告画面：2026-11-17 用 `am force-stop` 冷启动连拍 4 帧（0.4/0.8/1.2/1.6s），
0.4s 还是「钉钉 9.0」闪屏、0.8s 就已经进首页，全程没有开屏广告页。所以这条是否生效无法确认，
实际命中情况要看下次真出广告时的执行记录。钉钉首页本身是原生渲染（179 个节点、有文本），
不是自绘，所以按文字匹配的思路是可行的。

**⚠️ 代价**：这条等于自动同意「通过 USB 安装应用」，任何经该通道推来的安装请求都会放行，
不再有人工确认。只要还想人工看一眼，就在 GKD 里把这一组单独关掉。

### ④ 影梭 `com.zcshou.gogogo` — 更新弹窗-忽略一次

影梭（虚拟定位）每次启动都会弹一个更新提示（弹窗标题是版本号，如 `v2.1.0`，正文列更新日志，
底部两个按钮「忽略一次」「立即下载」）。这条在启动后 20 秒窗口内点「忽略一次」，
`actionMaximum: 1` 保证每个生命周期只点一次，**不点「立即下载」**，因此不会更新。

定位方式是 resource-id 而不是文案：

- 「忽略一次」= `com.zcshou.gogogo:id/update_ignore`
- 保险条件：同屏必须存在 `update_title`，避免别的页面误点

实测环境：影梭 1.12.3（弹窗提示可更新到 2.1.0）。节点结构由 `adb uiautomator dump` 实抓。

> 影梭 2.1.0 的升级提示写明「从 2.0.2 升级至 2.1.0 会重建本地数据库，清空位置历史、路线历史、
> 地图搜索历史」。这条规则等于永久钉在当前版本，不要更新就这样留着。

## 环境

### ⑤ 钉钉 `com.alibaba.android.rimet` — 首页横幅广告-关闭

首页消息列表顶部那条横幅推广卡（如「千问办公 | 你的『欠账清单』AI 已生成」），
整卡是一张图，文案在图片里，抓不到文字，只能按节点结构认：

```
(52,508)-(1168,880)  FrameLayout  vid=chatting_card_view      ← 横幅卡片容器
  (52,508)-(1168,880) ImageView   vid=image                   ← 广告图（可点，点了会跳广告）
  (1083,528)-(1148,593) FrameLayout vid=layout_close clickable=true ← 右上角 X
    (1089,534)-(1141,586) TextView vid=icft_close
```

规则只点 `layout_close`，并要求同屏存在 `chatting_card_view`，避免点到聊天页/对话框里
同名的关闭按钮。`actionMaximum: 1` + `matchTime: 15000`：每次进 App 最多关一次。

### ⑥ 微信 `com.tencent.mm` — 小程序广告浮层-关闭

微信里扫码骑车走的是小程序（包名还是微信 `com.tencent.mm`，Activity 是
`...plugin.appbrand.ui.AppBrandUI01`），小程序名有时叫「美团骑行」有时叫「美团」，
但规则不认具体小程序名，只认结构，两个都覆盖。

实测（小程序「美团骑行」右侧的「疯狂水世界」广告浮层）：

```
「广告」角标 TextView (923,395)-(988,447)
关闭 X    ImageView  (1125,399)-(1170,444)   ← 无 text、无 desc、clickable=false
```

小程序里这些东西是网页渲染的，只有零星节点带文字，关闭按钮完全没名字，所以只能按
**位置 + 尺寸** 认。三重保险，缺一不点：

1. 目标必须是裸 `ImageView`（`text=null` 且 `desc=null`）、宽高都 < 80px；
2. 必须落在右上角区域（`left>1000`、`top<600`）；
3. 同屏还要有「广告」字样 **和** 小程序自带胶囊 `desc="关闭"`
   （那个胶囊本身 156×144，尺寸条件已排除，不会被误点）。

**⚠️ 覆盖范围**：只覆盖「右上角小浮层」这一种。你截图里的「外卖营销弹窗」
（点外卖送次卡 / 首单外卖 0 元吃，X 在画面下方中间）是另一种版式，位置不同，
这一条的尺寸/位置条件匹配不到它 —— 那种需要在弹窗真出现时再抓一次节点单独加。

## 环境

- 设备：Xiaomi 17
- 系统：HyperOS 4.0.6
- GKD：1.13.2

规则里的节点文案来自这套环境的实际界面。其它机型/系统版本上文案不同会匹配不到，
需要自己改 `matches` 里的文字。

## 单独导入（不想加订阅）

GKD → 订阅 → 本地订阅 → 应用规则 → 右上角 `+`，分别贴下面文件里的对象（注释之外的 `{ }` 整段）：

- `GKD规则1-自动清除管理员安装通知.json5`
- `GKD规则2-自动确认USB安装提示.json5`
- `GKD规则3-钉钉开屏广告.json5`
- `GKD规则4-影梭忽略更新.json5`
- `GKD规则5-钉钉首页横幅广告.json5`
- `GKD规则6-微信小程序广告关闭.json5`

也可以直接用文件导入：`gkd.json5` 已通过 adb 推到手机 `/sdcard/Download/gkd.json5`，
在 GKD → 订阅 → 本地订阅 → 添加，选这个文件即可。
