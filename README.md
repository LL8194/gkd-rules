# GKD 规则：HyperOS 安装弹窗与通知

针对 HyperOS / 小米，处理「用 Shizuku / InstallerX 之类静默装应用」时冒出来的两处拦截；
另外附带两条日常烦人弹窗的规则（钉钉开屏广告、影梭更新提示）。

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

## 四条规则分别干什么

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

未实测。若某版本钉钉的跳过按钮是纯图标没有文本，这条匹配不到，需要换成 `[vid="..."]` 之类。

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
