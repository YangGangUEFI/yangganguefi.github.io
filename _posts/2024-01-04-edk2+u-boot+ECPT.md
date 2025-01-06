---
title: 一次横跨 EDK2 和 U-Boot 的问题排查
date: 2025-01-04 00:00:00 +0800
categories: [EDK2, UBOOT]
tags: [uefi, edk2, u-boot]     # TAG names should always be lowercase
---

在开源社区中，代码的质量和一致性至关重要。随着项目的不断发展，开发者们需要不断地审查和修复代码中的问题，以确保软件的稳定性和可靠性。本文将分享我在处理 EDK2 和 U-Boot 之间的一个具体问题时的思考过程和解决过程。

### 0x00 起因

最近，tianocore/edk2 社区在经历了从 email patch 转向 GitHub PR 进行代码审查后，也将 Bug 管理系统从 Bugzilla 转移到了 GitHub Issues。这一转变在一定程度上使得社区和开发者的工作能够更加集中于 GitHub 这一平台。同时，对于下游用户来说，他们也能更方便地关注感兴趣的内容、报告问题或为开源社区做出贡献。

于是，作为一名高强度的“网上冲浪”选手，这几天我在 Issues 列表中便瞥见了这样一条：[\[Bug\]: \<Mismatch of EFI Conformance Profile Table GUID when UEFI application reads from EFI System Table\>](https://github.com/tianocore/edk2/issues/10579)。

### 0x01 思考

在第一时间，我自然会查看这个 GUID 的定义：

![GUID1](/assets/media/posts/edk2+u-boot+ECPT/GUID1.png)

![GUID2](/assets/media/posts/edk2+u-boot+ECPT/GUID2.png)

显然，EDK2 中的定义并不是错误的 `36122546-F7EF-4C8F-BD9B-EB8525B50C0B`。那么，问题出在哪里呢？

于是，我在 GitHub 上搜索了 GUID 中的部分关键字 [0x36122546](https://github.com/search?q=0x36122546&type=code)。

![SearchResult](/assets/media/posts/edk2+u-boot+ECPT/SearchResult.png)

搜索结果的第二条提供了重要线索，甚至指出了问题的根源（至少在 u-boot 或其历史版本中，这个 GUID 的定义是错误的）。

接下来，我需要实际查看一下 u-boot 的代码，以便明确问题所在。

### 0x02 排查

在快速浏览 GitHub 上的 u-boot 仓库后，我发现它仅是一个镜像。为了确保信息的准确性，需要再去它的原始仓库 [https://source.denx.de/u-boot/u-boot](https://source.denx.de/u-boot/u-boot) 进一步查看。

最终，我可以确认在 u-boot 代码中这个 GUID 的定义自添加以来都是：

```
#define EFI_CONFORMANCE_PROFILES_TABLE_GUID \
	EFI_GUID(0x36122546, 0xf7ef, 0x4c8f, 0xbd, 0x9b, \
		 0xeb, 0x85, 0x25, 0xb5, 0x0c, 0x0b)
```

显然，这个定义是不正确的。

虽然提出 Issue 的这位朋友只提到了 `Tested environment: Qemu Arm64`，并没有说明测试环境中是否存在 u-boot。但考虑到 u-boot 在 ARM 生态中的重要地位，这一点不容忽视。

同时，参考 u-boot 这段代码在提交时的一些信息 [efi: Create ECPT table](https://github.com/u-boot/u-boot/commit/6b92c1735205eef308a9e33ec90330a3e6d27fc3)，可以看出这段代码是用来在 u-boot 加载 UEFI 环境时创建 ECPT（EFI Conformance Profile Table）的，并且**这个功能在默认情况下是打开的**。

![Uboot-ECPT](/assets/media/posts/edk2+u-boot+ECPT/Uboot-ECPT.png)

结合以上信息，我们可以大胆假设，提出 Issue 的这位朋友应该是通过 u-boot 进入 UEFI 环境并启动到 UEFI Shell，然后查看到了这个错误的 GUID。

接下来，我们需要修正这个 GUID 的定义，并提交给 u-boot 社区进行审核。

### 0x03 解决问题

首先，需要了解 u-boot 提交代码的流程。我们可以在代码的 README 文件中找到以下这些信息。

```
Contributing
============

The U-Boot projects depends on contributions from the user community.
If you want to participate, please, have a look at the 'General'
section of https://docs.u-boot.org/en/latest/develop/index.html
where we describe coding standards and the patch submission process.
```

通过链接内的内容，我们了解到 u-boot 项目是使用 email 来接收 patch 和 review 的。

根据提示，我们可以先在 [https://patchwork.ozlabs.org/project/uboot/list/](https://patchwork.ozlabs.org/project/uboot/list/) 注册一个账号（发送到 u-boot mailing list 的 patch 最终可以在这个网站上显示），然后在 [https://lists.denx.de/listinfo/u-boot](https://lists.denx.de/listinfo/u-boot) 订阅邮件列表。（如果不订阅，发送补丁后可能会收到 `Post by non-member to a members-only list` 的提示，不过即便订阅后也需要审核，因此理论上不订阅也是可行的。）

接下来，我们需要创建 patch 并将其发送到邮件列表以供审核。

代码的改动相对简单，因此不再赘述。以下是提交的基本流程：

1. 在本地修改代码并提交。

2. 使用 `git format-patch -1` 生成一个包含提交信息的 .patch 文件。

3. 通过 `./scripts/checkpatch.pl` 检查补丁/提交的格式，如果不符合要求，则需要进行相应的修改。

4. 使用 `./scripts/get_maintainer.pl` 获取补丁的维护者信息，以确保将补丁发送给合适的人员。（之前我忽略了这一步）

5. 最后，使用 `git send-email --to u-boot@lists.denx.de --cc xxx@xxx xxx.patch` 将改动发送到邮件列表。

在完成补丁提交后，我们需要耐心等待审核结果，并关注后续的进展。

### 0x04 后续

经过一段时间的等待，我之前发送的补丁已经在前面提到的两个网站上显示出来。维护者也给予了回复，并对补丁标注了 `Reviewed-by`。这意味着代码改动有望在 u-boot 的某个合并窗口中合入主线代码。

我们可以在 [https://lists.denx.de/pipermail/u-boot/2025-January/576425.html](https://lists.denx.de/pipermail/u-boot/2025-January/576425.html) 查阅到这些信息。

最后，回到 EDK2 的 Issue 页面，我回复了相关信息，包括询问使用的是否是 u-boot 环境，以及上述补丁的信息。我们可以在 [https://github.com/tianocore/edk2/issues/10579](https://github.com/tianocore/edk2/issues/10579) 上查看，期待他的回复。

### 0xFF 后记

这是一个简单的改动，但是却连接了两个开源社区。通过这个过程，我对 u-boot 有了初步的了解（尤其是在 UEFI 方面），未来也许可以进一步深入探索。
