# HonorLiquidGlassRestore（荣耀液态玻璃恢复）

> ## ⚠️ 本模块由 AI 生成 / This module is AI-generated
> 本项目由大语言模型（AI）在人类指导下生成并迭代维护，包括全部逆向分析、Hook 代码、
> 构建流水线与文档。代码未经人工长期审计，请自行评估风险后使用；欢迎人工审查与 PR。
>
> This project was generated and iterated by a large language model (AI) under human
> direction, including all reverse engineering, hook code, the build pipeline and these docs.
> The code has not been long-term audited by humans — evaluate the risk yourself;
> human review and PRs are welcome.

恢复 MagicOS11 阉割的液态玻璃效果 by VoreulCH@Github

面向荣耀 MagicOS 11 的 LSPosed 模块：系统 OTA 更新后，控制中心/通知中心的「液态玻璃」材质被降级为普通磨砂玻璃。本模块在系统界面与桌面进程内还原被阉割的渲染路径与材质位，让玻璃重新通透起来，并把**锁屏通知卡片与快捷按钮**也纳入液态玻璃渲染。所有参数实时可调（**遮罩浓度、模糊半径、光泽、折射、色散、边光**），支持一键套用**荣耀桌面文件夹的原生玻璃配方**。默认参数已按逐项调校的定稿配方烤入——装上即得调校后的观感，无需任何 setprop。

An LSPosed module for Honor MagicOS 11: the system update silently downgraded the
liquid-glass material of the control center / notification shade to plain frosted
glass. This module restores the gutted render path and material bits inside
SystemUI and the launcher, extends liquid glass to the **lock-screen notification
cards and shortcut buttons**, and exposes a full set of live-tunable optics
parameters (veil, blur radius, gloss, refraction, dispersion, rim light),
including a one-switch preset that applies the **stock glass recipe of the
Honor launcher folder**. The default values are baked-in and pre-calibrated —
a fresh install reproduces the tuned look with zero setprop.

## 为什么做这个模块 / Why

MagicOS 11 的系统更新从三个层面阉割了液态玻璃：

1. **机型降级**：`MachineConfig` 将本机判定为 `COMPAT`（兼容模式），渲染管线走降级路径；
2. **材质位摘除**：`hn_transparent_material_glass` 配置从 `"11"` 改为 `"1"`，LIQUID 材质从特效列表消失；
3. **模糊类型回退**：`TransparentModeManager` 初始化时把 `mBlurType` 缓存为磨砂（`0x10000`），锁屏后即触发降级渲染。

结果：更新前通透的液态玻璃变成了一层发灰的磨砂。

The update cut liquid glass at three layers: the machine was re-classified as
`COMPAT` (downgraded render pipeline), the material bit went from `"11"` to
`"1"` (LIQUID dropped from the effect list), and `mBlurType` was cached as
frosted (`0x10000`). Net result: clear liquid glass turned into flat frosted gray.

## 功能一览 / Features

| 功能 Feature | 说明 Description |
| --- | --- |
| 还原 LIQUID 材质位 / Restore LIQUID material bit | Hook `getTransparentModeState()`（大方法，AOT 不会内联），在 `mBlurType` 缓存后改写字段为 `0x20000`；后续所有 `isLiquidBlurType()` 判断（无论是否被内联）读到的都是液态 / Hooks the big `getTransparentModeState()` (AOT never inlines it) and overwrites the cached `mBlurType` field to `0x20000`; every later `isLiquidBlurType()` check reads liquid |
| 强制 HIGH 机型路径 / Force HIGH machine tier | Hook 机型判定，将 `COMPAT` 提升为 `HIGH`，解锁完整渲染管线（含实时模糊） / Lifts `COMPAT` to `HIGH` to unlock the full render pipeline (incl. real-time blur) |
| 材质类型改判 / Material type fix | Hook `StyleType.getMaterialType()` 返回液态材质 / Makes `StyleType.getMaterialType()` report liquid |
| 面板遮罩可调 / Panel veil | 控制中心背景由 `SurfaceControlEx.screenshot()` 原生模糊并叠加 62% 深色遮罩（`#9e1a1a1a`）生成——这是通透度的最大杠杆；`mask` 参数实时缩放其 alpha / The panel background is blurred and tinted (62% dark veil) natively at screenshot time — the dominant opacity lever, live-scalable via the `mask` prop |
| 模糊半径/光泽可调 / Blur radius & gloss | `radius` 缩放背景模糊半径，`sat` 放大背景饱和度（光泽感） / `radius` scales background blur, `sat` boosts saturation (gloss) |
| 通知卡片底色 / Notification card tint | 透明模式下通知卡背景是全透明的（`#00ffffff`），通透调高后文字难读；`nmask` 为卡片单独叠加绝对灰度底色，不动共享位图 / Notification cards are fully transparent in this mode; `nmask` gives each card its own readable backdrop without touching the shared panel bitmap |
| 文件夹玻璃配方 / Launcher-folder recipe | 从桌面反编译得到的原生配方（折射 0.3 / 深度 0.54 / 厚度 0.62 / 色散 0.2），`folder=1` 一键套用 / The stock folder optics (refraction 0.3 / depth 0.54 / thickness 0.62 / dispersion 0.2) recovered from the launcher — one switch applies it |
| 折射/厚度/色散分调 / Independent optics | `refract` / `thick` / `disp` 分别缩放透镜弯折、边缘折射带宽度与彩虹色散 / Scale lens bending, refraction-band width and chromatic fringe separately |
| 边光可调 / Rim light | `edge` 缩放 `EdgeLightParamEx` 边光带宽度（构造器级 Hook，内联免疫），`rim` 缩放边光亮度 / `edge` scales the rim band width (constructor hook, inline-proof), `rim` scales its brightness |
| 锁屏玻璃 / Lock-screen glass | 静态壁纸下原厂锁屏卡片/按钮本就渲染扁平（模糊位图只对动态/杂志壁纸生成）。模块从锐利壁纸合成轻量模糊位图（走原厂 `ViewBlur.blurBitmap` 配方管线），重放给锁屏渲染链，卡片与快捷按钮即获得与通知栏一致的液态玻璃 / Static wallpapers never get a blur bitmap, so lock-screen cards/buttons render flat. The module synthesizes one from the sharp wallpaper through the stock `ViewBlur.blurBitmap` recipe and re-feeds the keyguard render chain — cards and buttons get the same liquid glass as the shade |
| 全参数实时生效 / All props live | 改 prop 后收起再重拉面板即生效，无需重启 SystemUI / Change a prop, re-pull the panel — no restart needed |
| 烤入定稿默认值 / Baked-in recipe | 全部默认值已按逐项调校的定稿配方烤入（遮罩 0.7 / 模糊 0.7 / 光泽 1.2 / 文件夹配方开 / 锁屏链开），`setprop X ""` 清空即回落烤入值 / All defaults are baked in pre-calibrated; clear a prop to fall back to the baked value |

## 原理 / How it works

```
SystemUI 启动
  └─ TransparentModeManager.getTransparentModeState()   ← 大方法，AOT 不内联
       └─ 缓存 mBlurType = 0x10000 (磨砂/FROSTED)
            ↓ hook 改写字段
       mBlurType = 0x20000 (液态/LIQUID)
  └─ 拉下控制中心
       └─ WindowBlurView → SurfaceControlEx.screenshot()
            ├─ 原生模糊 + 62% 深色遮罩生成背景位图   ← hook 缩放遮罩 alpha
            └─ setForPanelTwiceBlur() 存入面板位图
       └─ HnBlurView → ControlCenterViewModel.turnOnViewBlur()
            ├─ HnMaterialFactory.createMaterial(2)     ← 液态材质
            ├─ BlurParametersConfig.initBulrParams(2)  ← hook：遮罩/折射/厚度/色散/通知卡底色
            └─ EdgeLightParamEx(250, 6.0f)             ← hook：边光宽度/亮度
```

关键工程决策：

- **只 Hook 大方法或构造器**：Vector 等 LegacyBridge 管理器没有 deopt，小 getter 会被 AOT 内联绕过——本模块的目标方法（`getTransparentModeState`、`initBulrParams`、`withTransparency`、`EdgeLightParamEx` 构造器）要么本身够大，要么是构造器，全部内联免疫。
- **改字段而非改返回值**：上游 hook 改写被缓存的状态字段，让后续（可能被内联的）读取者直接看到修正值。
- **不碰共享位图**：早期版本曾按面板索引暗化位图，但渲染端 `getPanelBitmap()` 优先返回 `[0]` 号位图——拉一次通知会把控制中心一起染黑。通知卡片底色改为卡片级着色（`controlNtfMaskColor` 绝对 alpha）后互不干扰。

Key engineering decisions: only big methods or constructors are hooked (no deopt
in LegacyBridge managers — small getters get inlined away by AOT); state is fixed
by writing cached fields rather than patching return values; and the shared panel
bitmap is never touched (the renderer prefers `bitmap[0]`, so per-index edits leak
across panels — notification cards are tinted at card level instead).

## 环境要求 / Requirements

* 已 root 的荣耀 MagicOS 11（Android 17）设备：Magisk / KernelSU / ReSukkiSU + LSPosed（或 Vector 等第三方管理器）。
  A rooted Honor MagicOS 11 (Android 17) device: Magisk / KernelSU / ReSukkiSU + LSPosed (or a third-party manager such as Vector).
* 系统界面（`com.android.systemui`）与荣耀桌面（`com.hihonor.android.launcher`、`com.hihonor.desktop.systemui`）。
  Scopes: SystemUI and the Honor launcher.
* 仅在 MagicOS 11 上实测；其他版本类名可能变化，Hook 未命中时功能静默失效（不影响系统稳定）。
  Tested on MagicOS 11 only; on other versions the hooks may silently miss (nothing breaks).

## 使用方法 / Usage

1. 安装 APK，在 LSPosed / Vector 中启用模块；作用域勾选 **系统界面** 与 **荣耀桌面**（安装包内已声明推荐作用域，管理器通常可一键应用）。
   Install the APK, enable the module, select the **System UI** and **Honor launcher** scopes (recommended scope is declared in the APK manifest).
2. 强制停止系统界面或重启 SystemUI（`su -c "kill $(pidof com.android.systemui)"`）。
   Restart SystemUI to activate.
3. 下拉控制中心：磨砂应已变回通透的液态玻璃。
   Pull down the control center — frosted should be liquid again.

### 实时调参 / Live tuning

所有参数走 `persist.sys.lgr.*` 系统属性，`setprop` 后收起再重拉面板即生效；**默认值即定稿配方（烤入 APK），正常使用无需任何设置**，以下仅供微调：

All parameters are system properties; after `setprop`, re-pull the panel to see
the change. **The defaults ARE the tuned recipe (baked into the APK) — no setup
needed for normal use**; the table below is for fine-tuning only:

| 属性 Prop | 范围 Range | 默认 Default | 作用 Effect |
| --- | --- | --- | --- |
| `persist.sys.lgr.mask` | 0.0–1.0 | 0.7 | 面板遮罩浓度（原厂 62%，越小越通透） Panel veil opacity (stock 62%) |
| `persist.sys.lgr.radius` | 0.3–2.0 | 0.7 | 背景模糊半径倍率 Background blur radius |
| `persist.sys.lgr.tblur` | 0.1–2.0 | 0.6 | 磁贴/通知卡片自身二次模糊倍率（残留磨砂感的主要来源） Per-tile second-pass blur scale (the residual frosted look) |
| `persist.sys.lgr.sat` | 0.8–2.0 | 1.2 | 光泽/饱和度倍率 Gloss (saturation) |
| `persist.sys.lgr.nmask` | 0.0–1.0 | 0.4 | 通知卡片底色浓度 Notification card tint |
| `persist.sys.lgr.folder` | 0/1 | 1 | 套用桌面文件夹玻璃配方（折射/厚度/色散绝对值） Apply the launcher-folder optics recipe |
| `persist.sys.lgr.refract` | 0.2–3.0 | 1.0 | 折射与深度倍率（透镜弯折） Refraction & depth |
| `persist.sys.lgr.thick` | 0.2–3.0 | 1.4 | 玻璃厚度（边缘折射带宽度）；folder 模式下作为 0.62 基准的倍率 Thickness (refraction band width); in folder mode a multiplier on the 0.62 base |
| `persist.sys.lgr.disp` | 0.0–3.0 | 0.25 | 色散强度（彩虹边缘） Dispersion (chromatic fringe) |
| `persist.sys.lgr.edge` | 0.5–3.0 | 1.4 | 边光带宽度 Rim-light band width |
| `persist.sys.lgr.rim` | 0.15–1.0 | 1.0 | 边光亮度 Rim-light brightness |
| `persist.sys.lgr.kbblur` | 0–40 | 4 | 锁屏合成位图模糊半径（0=原厂重磨砂 bokeh） Lock-screen synthesized bitmap blur radius (0 = stock heavy bokeh) |
| `persist.sys.lgr.kveil` | 0.0–2.0 | 0.7 | 锁屏卡片独立纱深浅 Lock-screen-only veil multiplier |
| `persist.sys.lgr.kbbright` | 0.0–0.5 | 0.22 | 锁屏卡面亮度 Lock-screen card face brightness |
| `persist.sys.lgr.kedge` | 0.5–3.0 | 1.4 | 锁屏独立边光宽度 Lock-screen-only rim width |
| `persist.sys.lgr.kbg`/`keng`/`kicon` | 0/1 | 1 | 锁屏玻璃链三开关（位图捕获/引擎通知/按钮路由），全部关闭即完全回退原厂锁屏渲染 The three lock-screen chain switches; all off = full stock lock-screen rendering |

示例 / Example:

```
adb shell su -c "setprop persist.sys.lgr.mask 0.7"      # 更沉稳的遮罩
adb shell su -c "setprop persist.sys.lgr.kveil 0.8"     # 锁屏卡片更深一点
adb shell su -c "setprop persist.sys.lgr.mask \"\""     # 清空=回落烤入默认值
```

**逃生门 / Escape hatch**：锁屏如遇任何异常，`setprop persist.sys.lgr.kbg 0 && setprop persist.sys.lgr.keng 0 && setprop persist.sys.lgr.kicon 0` 后重启 SystemUI 即完全回到原厂锁屏渲染（面板效果不受影响）。
If the lock screen ever misbehaves, zero out `kbg`/`keng`/`kicon` and restart SystemUI — the lock screen returns to 100% stock rendering (panel effects unaffected).

### 已知问题 / Known issues

* **AOT 内联**：极少数情况下系统重新全量编译 SystemUI 后 Hook 可能被绕过（表现为效果回退到磨砂）。执行 `su -c "cmd package compile -m interpret-only -f com.android.systemui"` 后重启 SystemUI 即可恢复。
  If the effect ever reverts after a full AOT recompile, force interpret-only compilation for SystemUI and restart it.
* 文件夹的宽折射带叠在清晰壁纸上最明显；控制中心背景经过预模糊，折射观感天然更含蓄。
  The folder-style refraction band is most visible over sharp content; the panel background is pre-blurred, so refraction reads more subtly there.

## 从源码构建 / Build from source

本项目刻意不依赖 Gradle / Android Studio：

```
powershell -ExecutionPolicy Bypass -File build.ps1
```

流程：`ecj` 编译 Java 8 → `d8` 转 dex → `aapt2` 打包 → `zipalign` 对齐 → `apksigner` 签名。
产物输出至 `HonorLiquidGlassRestore.apk`。

需自行下载放入 `tools/`（体积原因未提交本仓库）：`ecj.jar`（Maven Central）、
`android.jar`（SDK platform-36）、build-tools（`d8`/`aapt2`/`zipalign`/`apksigner`）、
`api-82.jar`（Xposed API 82，仅编译期引用）。

The build deliberately avoids Gradle: `ecj` → `d8` → `aapt2` → `zipalign` → `apksigner`.
Drop the toolchain pieces (ecj, android.jar platform-36, build-tools, Xposed api-82)
into `tools/` before building.

## 项目结构 / Project layout

```
├── AndroidManifest.xml        # xposedmodule 元数据与作用域 / module meta + scope
├── assets/xposed_init         # Xposed 入口声明 / entry declaration
├── res/values/strings.xml     # 应用名与描述 / app name + description
├── res/values/arrays.xml      # 推荐作用域数组 / recommended scope array
├── src/io/github/voreulch/liquidglass/
│   └── MainHook.java          # 全部 Hook 逻辑 / all hook logic
├── inject_dex.py              # dex 注入打包辅助 / packaging helper
└── build.ps1                  # 一键构建流水线 / one-shot build pipeline
```

## 版本历史 / Changelog

* **2.5.18**（2026-10-09）：**锁屏液态玻璃 + 定稿配方烤入**。①锁屏通知卡片与快捷按钮获得与通知栏一致的液态玻璃（静态壁纸下原厂锁屏本就无模糊位图、渲染扁平；模块从锐利壁纸经原厂 `ViewBlur.blurBitmap` 管线合成轻量位图并重放给锁屏渲染链）；②卡片配方逐项对齐通知栏：卡面提亮（`kbbright`）、独立纱深浅（`kveil`）、独立边光宽度（`kedge`）、合成模糊半径（`kbblur`）；③全部调校参数烤入 APK 默认值，装上即得定稿观感，`setprop` 覆盖、清空回落；④移除 v2.3 时代的实验性引擎旁路代码（kglass）与被取代的旧合成路径，代码瘦身 440 行。
  **Lock-screen liquid glass + baked-in recipe.** Lock-screen cards/buttons now match the shade look (stock renders them flat on static wallpapers — no blur bitmap is ever generated; the module synthesizes one from the sharp wallpaper through the stock `ViewBlur.blurBitmap` pipeline and re-feeds the keyguard chain). Card recipe aligned to the shade item by item (face brightness, veil, rim width, blur radius), all tuned defaults baked into the APK, and the v2.3-era experimental engine-bypass code removed (−440 lines).
* **2.2**（2026-10-07）：新增 `tblur` 参数——磁贴与通知卡片在已模糊背景上还有一层自身二次模糊（`blurRadius` ≈5.6px），这是残留磨砂感的主要来源；该参数独立缩放这层模糊，与背景模糊 `radius` 解耦，磁贴可单独变得通透。
  New `tblur` prop: tiles / notification cards re-blur the already-blurred panel bitmap with their own `blurRadius` (~5.6px) — the residual frosted look. Scales that second pass independently of the background blur.
* **2.1**（2026-10-07）：首个公开发布——还原液态材质位/HIGH 机型路径/材质类型；面板遮罩、模糊半径、光泽、通知卡片底色、折射/厚度/色散、边光宽度与亮度全参数实时可调；新增桌面文件夹玻璃配方（`folder=1`）。
  First public release: restores the liquid material bits and HIGH render path, full live-tunable optics, plus the launcher-folder glass recipe.

## 免责声明 / Disclaimer

* 本项目仅供学习与研究 Android Hook 技术使用，请勿用于商业用途。
  For learning and research on Android hook techniques only; do not use commercially.
* 与荣耀公司 / HONOR 无任何关联；相关商标与系统界面版权归原厂所有。
  Not affiliated with Honor Device Co., Ltd.
* 使用本模块产生的任何后果由使用者自行承担。Use at your own risk.
