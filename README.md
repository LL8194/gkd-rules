# GKD 规则：HyperOS 安装弹窗与通知

针对 HyperOS / 小米，处理「用 Shizuku / InstallerX 之类静默装应用」时冒出来的两处拦截；
另外附带日常烦人广告的规则：钉钉（开屏 + 首页横幅推广卡）、影梭更新提示、
微信小程序（开屏「跳过」+ 右上角浮层关闭）、美团开屏「跳过」。

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

## 七条规则分别干什么

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

### ⑥ 微信 `com.tencent.mm` — 小程序广告（开屏跳过 + 浮层关闭）

日常扫码骑车打开的是**微信小程序**：窗口是 `com.tencent.mm` 下的
`...plugin.appbrand.ui.AppBrandUI00`，包名始终是微信，小程序名有时叫「美团骑行」
有时叫「美团」，规则不认名字、只认节点，两个都覆盖。

2026-11-18 实机抓节点（`adb uiautomator dump`）：

```
开屏广告： 「广告 ∨」TextView [78,191][169,256]
          「跳过」  TextView [273,191][373,256]，右侧「2 秒」倒计时
          → [text="跳过"][text.length<10]，实测点 (323,223) 可跳过

首页浮层： 关闭 X = 图标字体 TextView \ue003 [1062,390][1124,455]
          容器 FrameLayout [1069,392][1189,489]
          → 实测点 (1125,440) 可关闭

同页干扰： 右上角「我的」笑脸图标**也是** \ue003 [1056,357][1118,422]
          → 与广告 X 只差几像素，用 top>=385 排除，避免误点「我的」
```

**踩过的坑**：11-17 那版按「右上角裸 `ImageView`」写的（`ImageView[text=null][desc=null]
[left>1000][top<600][width<80][height<80]`），而 11-18 实测到的关闭 X 是 `TextView`
（图标字体），类名一变整条规则就匹配不上、一次都不触发。小程序不同版本渲染方式会变，
条件里别写死 `ImageView`。

**⚠️ 覆盖范围**：只覆盖「开屏跳过」和「右上角浮层」。订单页/还车页那种「点外卖 送次卡」
营销大弹窗，在美团 App 的容器里是无障碍看不见的（见 ⑦）；微信小程序里是否同样可抓，
留给下次真出现弹窗时再补一条。

### ⑦ 美团 `com.sankuai.meituan` — 小程序开屏-跳过（备用）

美团 App 用 `am force-stop` 冷启动两次，0.6 秒时已进首页，**没抓到过开屏广告页**，
所以这条目前是备用，留着不影响正常使用。

**⚠️ 为什么美团这边不写营销弹窗规则**：美团 App 内的小程序容器是
`com.meituan.msc.modules.container.MSCActivity`，它把广告弹窗画在**不暴露给无障碍的自绘层**。
弹窗在屏时连抓三次节点树，字节数完全一致（179706 B），里面：

- 「点外卖 / 送次卡 / 下单随机领 / 黑钻体验卡 / 关闭」计数**全为 0**
- 整棵树 `ImageView` 计数 **0**
- 只有页面本身的纯文本（「订单详情」「扫码用车」「车辆报修」等）是节点

节点类规则无从匹配，硬按截图坐标点也会点到广告图上（实测点 (596,1503) 直接跳进了外卖活动页）。
这类弹窗要么手动关，要么走「截图识别 + 坐标点击」的非节点方案。

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
- `GKD规则7-美团小程序开屏跳过.json5`

也可以直接用文件导入：`gkd.json5` 已通过 adb 推到手机 `/sdcard/Download/gkd.json5`，
在 GKD → 订阅 → 本地订阅 → 添加，选这个文件即可。
