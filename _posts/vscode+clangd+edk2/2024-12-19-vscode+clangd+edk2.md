---
title: 使用 VSCODE+clangd 实现 EDK2 代码的导航/跳转
date: 2024-12-19 00:00:00 +0800
categories: [UEFI, EDK2]
tags: [uefi, edk2, vscode]     # TAG names should always be lowercase
---

## 使用 VSCODE+clangd 实现 EDK2 代码的导航/跳转

“工欲善其事，必先利其器”，阅读代码亦是如此。对于开发人员来说，Editor/IDE 便是“器”中之一。

相较于稍显“传统”的 Source Insight，很多 UEFI 开发者现在可能更倾向于使用 VSCODE 来查看 [EDK2](https://github.com/tianocore/edk2) 代码，并且通常会依赖 Microsoft 官方的 [C/C++插件(vscode-cpptools)](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools) 来实现源代码的导航与跳转。

然而，一提到 VSCODE，性能上的批评似乎总是难以避免（尽管微软的优化据说在类似技术栈中已经算是比较出色的）。对于 `C/C++` 开发者而言，`vscode-cpptools` 可能更是那个“罪魁祸首”，其使用过程中内存占用可能较大，并且会在 `%LocalAppData%/Microsoft/vscode-cpptools` 目录生成大量缓存文件。

如果你也遇到类似的困扰，不妨尝试一下结合 `VSCODE` + `clangd` + `compile_commands.json` 的方案，这可能帮助你解决一些问题。

具体如何来使用，可以按照以下步骤来操作。

### 前置条件/环境配置

1. 卸载 [vscode-cpptools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools) 插件

    如果之前安装了这个插件，那么需要先卸载它。在 VSCODE 的 Extensions 中搜索 `C/C++` 找到如下图这个插件，然后点击 Uninstall。

    ![vscode-cpptools](/assets/media/posts/vscode+clangd+edk2/vscode-cpptools.png)

2. 安装 [vscode-clangd](https://marketplace.visualstudio.com/items?itemName=llvm-vs-code-extensions.vscode-clangd) 插件
   
    在 VSCODE 的 Extensions 中搜索 `clangd` 找到如下图这个插件，然后点击 Install。

    ![vscode-clangd](/assets/media/posts/vscode+clangd+edk2/vscode-clangd.png)

3. 下载 [clangd](https://github.com/clangd/clangd/releases/tag/18.1.3) 并将其解压到合适的位置

4. 安装 [Edk2code](https://marketplace.visualstudio.com/items?itemName=intel-corporation.edk2code) 插件
   
    在 VSCODE 的 Extensions 中搜索 `Edk2Code` 找到如下图这个插件，然后点击 Install。

    ![Edk2Code](/assets/media/posts/vscode+clangd+edk2/Edk2Code.png)

    > 该插件对 `.inf` `.dsc` `.fdf` 以及 `.asl` 等文件提供了语法高亮、代码导航等支持，可以自动搜索当前工作区的一些基本信息，同时能够辅助开发人员自动完成 `CompileInfo` 相关的配置。

### 生成 `compile_commands.json` 文件

> `compile_commands.json` 是 `clangd` 用来理解代码的关键文件，有了他，才能保证代码能够准确的跳转。

1. 确保你的 EDK2 代码包含了 [BaseTools: Generate compile information in build report](https://github.com/tianocore/edk2/pull/4135) 的这两条改动
   
    ![edk2_basetool](/assets/media/posts/vscode+clangd+edk2/edk2_basetool.png)

2. 编译代码时，加入参数：`-y report.txt -Y COMPILE_INFO`
   
    以 `OvmfPkg` 为例，对应的命令为：

    ```
    build -p OvmfPkg\OvmfPkgIa32X64.dsc -a IA32 -a X64 -t VS2022    -y report.txt -Y COMPILE_INFO
    ```

    代码编译时，EDK2 的 `BuildReport.py` 会在 `Build\Ovmf3264\DEBUG_VS2022` 下创建 `CompileInfo` 文件夹，代码编译完成后便可在其中找`compile_commands.json` 文件。

    ![CompileInfoDir](/assets/media/posts/vscode+clangd+edk2/CompileInfoDir.png)

### VSCODE 插件配置和使用
1. vscode-clangd 配置

    打开 VSCODE，按 `CTRL+,` 进入设置，搜索 `Clangd: Path`，然后将 `clangd` 的执行文件路径填写进去，例如 `D:\clangd\bin\clangd.exe`。

    ![clangd-path](/assets/media/posts/vscode+clangd+edk2/clangd-path.png)

2. Edk2Code 配置

    > 注意：如果你之前安装过此插件，请确保将其更新至 [Edk2Code_1.0.8](https://github.com/intel/Edk2Code/releases/tag/Edk2Code_1.0.8) 版本。此版本包含了针对 vscode-clangd 配置文件做的一点微小的工作。

    按 `CTRL+shift+p` 输入 `EDK2: Rebuild index database`，选中并回车。
   
    ![rebuild-index](/assets/media/posts/vscode+clangd+edk2/rebuild-index.png)

    在弹出的文件选择界面选择 `Build` 目录
   
    ![build-select](/assets/media/posts/vscode+clangd+edk2/build-select.png)

    在如下弹窗中选择对应要使用的 BuildTarget（也可以全选）
   
    ![build-target-select](/assets/media/posts/vscode+clangd+edk2/build-target-select.png)

    如果看到如下提示，需要点击 `Fix`。插件将自动修正或添加 `clangd.arguments` 到当前工作区的 `settings.json` 文件中
   
    ![clangd-argument-fix](/assets/media/posts/vscode+clangd+edk2/clangd-argument-fix.png)

    ![clangd-settings](/assets/media/posts/vscode+clangd+edk2/clangd-settings.png)

3. 重启 clangd language server

    按下 `CTRL+shift+p`，输入 `clangd: Restart language server`，选中并回车。以让前面的配置生效。
   
    ![clangd-restart](/assets/media/posts/vscode+clangd+edk2/clangd-restart.png)

4. 验证/使用

    通过以上步骤的配置，现在代码应该已经可以正常跳转了。我们来做几个试验看一下效果。

   * 函数的跳转

        打开 `MdeModulePkg\Universal\BdsDxe\BdsEntry.c` 这个文件。接着找到 `PlatformBootManagerBeforeConsole ();` 这个函数调用的位置。

        这个函数在 EDK2 中有多个 Library 实现的，我们看看能不能正确跳转到他的实际位置。

        把鼠标指针放在这个函数上面，可以看到弹出的窗口中能够正确展示函数的参数及注释等信息。

        ![test-function](/assets/media/posts/vscode+clangd+edk2/test-function.png)

        然后按一下 `Alt+F12`(Peek Definition)，因为前面我们选择的是 OVMF 的 CompileInfo，这里可以看到，准确找到了 `OvmfPkg\Library\PlatformBootManagerLib\BdsPlatform.c` 中的实现。

        ![test-function-jump](/assets/media/posts/vscode+clangd+edk2/test-function-jump.png)

   * 全局变量

        如法炮制，找一个 gBS 的调用，也可以正常跳转。

        ![test-global-variable-jump](/assets/media/posts/vscode+clangd+edk2/test-global-variable-jump.png)

   * enum struct MACRO 等

        `LoadOptionTypeMax` 是 enum 类型的常量。

        ![test-enum](/assets/media/posts/vscode+clangd+edk2/test-enum.png)

        `LoadOptions[Index].Description` 是比较常见的 `EFI_BOOT_MANAGER_LOAD_OPTION` 中的启动项描述。

        ![test-struct](/assets/media/posts/vscode+clangd+edk2/test-struct.png)

        DEBUG 相关的宏。

        ![test-macro](/assets/media/posts/vscode+clangd+edk2/test-macro.png)

        可以看到，以上几种类型均可以正确的跳转到对应位置。
        还有一些其他的示例，这里就不一一展示了，大家可以配置完成后实际体验一下。

### 其他注意事项

1. 必须配置好 `compile_commands.json` 文件才能正常工作。如果没有这个文件，或者代码尚未编译，跳转功能可能无法使用。在这种情况下，可能需要生成一个 `.clangd` 配置文件。
2. `BuildReport.py` 生成的 Workspace Path 是转换成 lowercase 的，VSCODE 在跳转文件时似乎会按照绝对路径打开文件而不是使用相对路径的文件，可能需要一个合适的解决方案。

### 结语

使用类似的方法，也可以在其他编辑器（如 Sublime Text、Vim 等）中加以配置，然后便可以在一个更加轻量化的环境下阅读代码了。

另外，别忘记删掉之前 `vscode-cpptools` 的缓存哦！