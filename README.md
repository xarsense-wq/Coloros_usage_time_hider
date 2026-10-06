# ColorOS 电池页面个性化/屏幕使用时间（v2.8-coloros17）

> 全局“亮屏活动时长”会根据用户对应用的隐藏/改时配置自动重算，按当前 ColorOS 统计区间
> 的原始应用前台时长计算显示层差值；不写入 BatteryStats，也不改息屏活动时长。

这是一个 LSPosed 模块，用来分别控制 ColorOS 电池页面中的两种显示效果：

- **隐藏应用**：勾选后，该应用条目从电池排行列表移除。
- **修改显示时间**：勾选后，条目继续显示，但 `|` 后的页面时间改为用户填写的
  `XX小时XX分钟XX秒`；点进应用“耗电详情”后，前台活动时长同步为该值，后台活动时长按
  原始前台/后台比例自动换算。

两个模式相互独立。同一个应用同时勾选时，隐藏优先；取消任意一个模式不会影响另一个模式。
首次启动只默认勾选并隐藏模块自身 `com.bule.color`，抖音、QQ、微信等不会被新增到任何模式。

## ColorOS 17 适配

- ColorOS 17（当前 PJD110 `17.0.0.106`）的排行列表入口改为
  `SipperListPreference.b0(List)`，详情前台/后台显示入口改为
  `PowerControlStatsPreference.J(String)` / `L(String)`。
- 模块现在优先 Hook 这些新入口，同时保留旧版 `Y/I/E` 方法名回退，因此旧版和
  ColorOS 17 共用一个 APK。

## 版本与作用域

- 版本：`2.8-coloros17`（versionCode `11`）
- 模块包：`com.bule.color`
- 唯一宿主包：`com.oplus.battery`
- 实际 Hook 进程：`com.oplus.battery:ui`
- 排行 Hook：`com.oplus.powermanager.powercurve.graph.SipperListPreference.Y(java.util.List)`
- 详情 Hook：`PowerControlStatsPreference.I(String)`（JADX：`m16663I`，前台）和
  `E(String)`（JADX：`m16659E`，后台）
- 比例换算入口：`i9.c1.J0(double,double,long,long)`（今天/时间区间）与
  `i9.c1.H0(int, PowerControlBarGraph$a)`（过去 7 天/单日）
- 全局亮屏 Hook：`PowerCurvePreference.e0(long,long,C5018k,boolean)` 读取当前统计区间的
  原始亮屏时长并按配置计算差值；最终只在 `PowerCurvePreference.r(TextView,String)` 中匹配
  `screen_on_time`，替换亮屏文字，不匹配 `screen_off_time`。
- LSPosed 作用域只选择：`电池 / com.oplus.battery`

桌面应用名称和白色主标题：`ColorOS 屏幕使用时间`；紫色小标题：`COLOROS LSPOSED · bule不bule`。
配置页是深紫色卡片式 UI，提供“隐藏应用 / 修改显示时间”分段按钮、搜索、应用图标、
数量提示、当前模式选中项置顶、打开电池、强行停止并重新打开电池，以及 `bule不bule` 水印。

## 原理

ColorOS 电池排行的列表元素是 `PowerSipper`。本模块在 `Y(List)` 调用前读取自己的只读
`ContentProvider`，返回每个包名的 `hide_enabled`、`time_enabled` 和该包自己的 `time_seconds`。

- 命中隐藏集合：从传给电池页面的列表复制品中移除对象。
- 命中时间集合：保留对象，只改 `PowerSipper.mSummary` 中 `|` 后的文本。
- 进入该应用耗电详情时：用户填写值作为新的前台活动时长；后台活动时长按
  `round(原后台秒数 × 新前台秒数 ÷ 原前台秒数)` 换算。原前台为 0 时后台显示为空。
- 两个原始时长字段 `mUsageTime`、`mFGTime`、`mBGTime`、耗电量、百分比以及 BatteryStats
  均不写入。

全局亮屏显示的自动计算规则为：

```text
新亮屏时长 = 原亮屏时长 + Σ(新应用前台时长 - 原应用前台时长)
```

其中，隐藏应用的新时长为 0；只改时间的应用使用用户填写的秒数；同一应用同时勾选时隐藏优先，
只计算一次。计算使用当前统计区间从 ColorOS 原始应用列表得到的完整包级前台时长快照，不受排行折叠、
只显示前 5 条或排序方式影响；结果下限为 0、上限为当前统计区间长度，避免出现亮屏时间超过实际区间的假值。
过去多日的平均视图会把差值按 ColorOS 的日数同步取平均，并把上限同步限制在单日长度。详情页、排行页和总览页
都只改最终显示字符串，所有值为 0 的时间单位均省略；息屏活动时长保持原值。

比例计算以 ColorOS 传入的原始毫秒值为锚点，换算过程中用大整数避免乘法溢出。今天/区间参数只在
当前方法调用内替换；过去 7 天的单日对象会在显示完成后恢复，因此图表源数据、BatteryStats 和后续
系统计算都不会被模块污染。

模块检查 `mPackageName`、`mPackageWithHighestDrain` 和 `mApplicationInfo.packageName` 三个
包名来源。配置保存在模块私有 `SharedPreferences`，旧版 v2.3 的单集合会迁移到“修改显示时间”
集合，以保留原配置；每个时间目标独立保存自己的时长；模块自身只加入隐藏集合一次，之后用户取消勾选会被保存。

## 构建

要求 Windows PowerShell、JDK 17、Android SDK Platform 35、Build Tools 35.0.0。

```powershell
$env:JAVA_HOME = 'C:\Program Files\Zulu\zulu-17'
$env:ANDROID_SDK_ROOT = 'C:\Android\Sdk'
.\build.ps1
```

输出：`build/ColorOS-App-Usage-Time-Hider-v2.7-auto.apk`。

## 安装与使用

```powershell
adb install -r .\build\ColorOS-App-Usage-Time-Hider-v2.7-auto.apk
```

1. 在 LSPosed 启用模块，作用域只勾选 `com.oplus.battery`。
2. 打开模块应用，首次默认位于“隐藏应用”模式，模块自身已勾选并置顶。
3. 在“隐藏应用”模式勾选目标，目标会从排行消失；在“修改显示时间”模式勾选目标后，立即弹出该应用自己的
   小时、分钟、秒输入框。点“保存并勾选”才会保存；点“取消”则不勾选。已勾选的时间目标再次点击应用卡片，
   可以重新编辑自己的时长。页面会省略值为 0 的单位：例如 `5分钟`、`30秒`、`1小时5分钟`；全为 0 时不显示时间文本。
4. 每个模式的勾选即时保存，当前模式勾选项自动置顶；切换模式不会清空另一模式。
5. 重新打开电池页面使 Hook 重新读取配置。

时间输入规则：小时为非负整数，分钟和秒自动限制在 `0–59`。例如填写 `1 / 2 / 3`，页面显示
`1小时2分钟3秒`。

“强行停止并重新打开电池”按钮会先执行 `su -c id -u`。Magisk 等 Root 管理器会在此时弹出授权；
若没有 Root，模块直接打开电池应用详情，用户手动点击“强行停止”后返回即可重新进入电池页面。

## 验证

```powershell
adb shell content query --user 0 --uri content://com.bule.color.config/hidden
adb shell am force-stop com.oplus.battery
adb shell am start -a android.settings.BATTERY_SAVER_SETTINGS
adb logcat -d -v brief | Select-String ColorOSBatteryFilter
```

日志会报告隐藏/时间配置数量、列表移除和 summary 修改数量；详情页完成换算后还会记录新的前台/后台秒数。
不要使用 `dumpsys batterystats --reset`，也不要清除 `com.oplus.battery` 数据；本模块只改变页面显示。
