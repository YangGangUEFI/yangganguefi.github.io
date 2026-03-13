---
title: Edk2 Build 之失效的 BuildOptions
date: 2026-03-07 00:00:00 +0800
categories: [EDK2，BUILDOPTIONS]
tags: [edk2，buildoptions]     # TAG names should always be lowercase
---

在将第三方开源项目移植到 UEFI(EDK2) 环境时，为了避免修改原始代码中存在的部分警告，大多时候需要在 INF 文件的 `BuildOptions` 中添加相应的编译选项。但是多数时候仅是简单的使用，并未对其做过多的了解。这其中可能就有些“想当然”的坑要踩。

### 失效的 BuildOptions

最近在更新 [LvglLib](https://github.com/YangGangUEFI/LvglPkg) 时，遇到一个 `warning C4244` 的编译报错。

但是明明 LivlLib.inf 中已经有了 `MSFT:*_*_*_CC_FLAGS = /wd4244` 这个编译选项，为什么没有生效呢？

我们来看一下比较完整的报错：

```
misc/lv_color.h(355): warning C4244: 'function': conversion from 'int' to 'uint8_t', possible loss of data
```

眼力好的同学可能已经发现端倪，能猜出一二了。

### 由 [Depex] 的继承展开联想

众所周知，当我们需要为某一个 DXE Driver 引入一些平台/项目特定的 Depex 时，可以在 DSC 文件中为其指定一个 NULL library instance，在这个 library 的 INF 中加入 `[Depex]` 字段，xxxDxe 便可继承这些 Depex。

```
  Path-To-Dxe-Driver/xxxDxe.inf {
    <LibraryClasses>
      NULL|Path-To-Null-Library/NullLibrary.inf
  }
```

有了这个经验，便“想当然”地以为使用 Library 的 UEFI_APPLICATION/DXE_DRIVER 等模块也可以“继承”其中的 `[BuildOptions]`。但事实真的如此吗？

> 提出问题似乎已经预示了答案。如果你感兴趣，也可以查阅 [INF Specification](https://tianocore-docs.github.io/edk2-InfSpecification/draft/#edk-ii-module-information-inf-file-specification) 来更详细的了解相关 Section 的用法。

抛开 Spec 描述，我们可以直接找到 LvglLib 及使用它的 App 的 Makefile 然后对比其中的 `CC_FLAGS`。

`Build\Lvgl\RELEASE_VS2022\X64\LvglPkg\Application\UefiDashboard\UefiDashboard\Makefile` 中只有 `CC_FLAGS = /nologo /c /WX /GS /W4 /Gs32768 /D UNICODE /O1b2s /GL /Gy /FIAutoGen.h /EHs-c- /GR- /GF /Gw /volatileMetadata-` 这是 tools_def 默认的配置，而 LvglLib 的 Makefile 中则在后面附加了 INF 中设置的编译选项。结果显而易见。

### 具体是怎么失效的

为了使用到 LvglLib 的模块可以完整使用 lvgl 的接口，LvglLib 并未做过多包装，直接在 LvglLib.h 中 `#include <lvgl.h>` 来引入 lvgl 的头文件。

最近小米的同学在 `lvgl\src\misc\lv_color.h` 中添加了一个 inline 函数：

```c
static inline lv_color_t lv_color16_to_color(lv_color16_t c)
{
    return lv_color_make(c.red << 3, c.green << 2, c.blue << 3);
}
```

`<<` 会将变量提升为 `int`，而 `lv_color_make` 的参数类型是 `uint8_t`。

App(UefiDashboard.inf) 中的源码通过 `#include <LvglLib.h>` 引入了 `lvgl.h`，进而引入了 `lv_color.h`。
前面我们知道了 INF 中的 `[BuildOptions]` 并没有“继承”的概念，而 UefiDashboard.inf 也没有设置 `[BuildOptions]`，所以在 `/WX` 的条件下，就产生了 `warning C4244` 的编译警告报错。

猛地看起来就像 LvglLib 的编译选项没有生效一样。

### 怎么解决

解决的办法有很多种，我们来逐一看看。

#### 1. 在 INF 中设置 `[BuildOptions]`

在 UefiDashboard.inf 中添加 `[BuildOptions]`：

```ini
[BuildOptions]
  MSFT:*_*_*_CC_FLAGS = /wd4244
```

但这样需要更新所有的 App INF 文件，比较麻烦。

#### 2. 在 DSC 中设置 `[BuildOptions]`

语法同 INF 中一样，DSC 的设置会写入 `Build\Lvgl\RELEASE_VS2022\X64\TOOLS_DEF.X64` 这会应用到所有参与编译的模块。

但是能控制的 DSC 有限，只有 `LvglPkg\LvglPkg.dsc` 一个。

#### 3. 在 LvglLib.h 中配置

```c
#if defined(_MSC_VER)
#pragma warning(disable: 4244)
#endif
```

在 LvglLib.h 中配置 `#pragma warning(disable: 4244)` 可以禁用这个警告，而不需要修改 INF 或 DSC 文件。

这样使用 LvglLib 的 App 便都能正常编译了（除非再用另一种方式引入了 `lv_color.h`）。

#### 4. 直接修改 lvgl 源码

这是最彻底的方法，只需简单的强制转换即可。

```c
return lv_color_make((uint8_t)(c.red << 3), (uint8_t)(c.green << 2), (uint8_t)(c.blue << 3));
```

[相关改动](https://github.com/lvgl/lvgl/pull/9810)已被 lvgl merge，但是实际使用还是要等到下一次 release 了。
