---
title: 一次横跨 EDK2 和 U-Boot 的问题排查
date: 2025-01-04 00:00:00 +0800
categories: [EDK2, UBOOT]
tags: [uefi, edk2, u-boot]     # TAG names should always be lowercase
---

### 0x00 起因

继从 email patch 转到 GitHub PR 进行 Code Review 之后，最近 `tianocore/edk2` 社区也将 Bug 管理系统从 Bugzilla 转移到了 GitHub Issues 上来。这些转移在一定程度上使得社区/开发者的工作能够更加集中在 GitHub 这一个系统上面。对于下游的使用者来说，也能更方便的关注一些感兴趣的内容，或者汇报一些 Issue，或者为开源社区做一些贡献。

这两天在“网上冲浪”时，在 Issues 列表中便瞥见了这样一条：[\[Bug\]: \<Mismatch of EFI Conformance Profile Table GUID when UEFI application reads from EFI System Table\>](https://github.com/tianocore/edk2/issues/10579)。

### 0x01 思考

第一时间肯定会去看一眼这个 GUID 的定义：

![GUID1](/assets/media/posts/edk2+u-boot+ECPT/GUID1.png)

![GUID2](/assets/media/posts/edk2+u-boot+ECPT/GUID2.png)

很明显，EDK2 这里的定义并不是错误的 `36122546-F7EF-4C8F-BD9B-EB8525B50C0B`。那么问题会是谁的呢？

于是，直接在 GitHub 里搜索了 GUID 中的部分关键字 [0x36122546](https://github.com/search?q=0x36122546&type=code)。

![SearchResult](/assets/media/posts/edk2+u-boot+ECPT/SearchResult.png)

这里结果的第二条，便提供了很大的线索，甚至说它已经指出了问题所在（至少 u-boot 或其历史版本中的这个 GUID 的定义是存在错误的）。

### 0x02 排查

快速看了下 GitHub 里的这个 u-boot 仓库，发现它只是一个 mirror，保险起见再去它的原始仓库 https://source.denx.de/u-boot/u-boot 看一下。

最终发现在最新的 u-boot 代码中这个 GUID 的定义都是

```
#define EFI_CONFORMANCE_PROFILES_TABLE_GUID \
	EFI_GUID(0x36122546, 0xf7ef, 0x4c8f, 0xbd, 0x9b, \
		 0xeb, 0x85, 0x25, 0xb5, 0x0c, 0x0b)
```

虽然原始 Issue 中那位朋友只提到了 `Tested environment: Qemu Arm64` 而没有提到测试环境中是否有 u-boot 的存在。但是考虑到在 ARM 生态中 u-boot 又占据了非常重要的地位。同时参考 u-boot 这段代码在提交时的一些信息 [efi: Create ECPT table](https://github.com/u-boot/u-boot/commit/6b92c1735205eef308a9e33ec90330a3e6d27fc3)，他是用来在 u-boot 加载 UEFI 环境（这也是一个后续可以更深入了解一些的内容，应该类似与现在的 UefiPayload）时创建 ECPT(EFI Conformance Profile Table) 的，并且这个功能在默认情况下是打开的。

![Uboot-ECPT](/assets/media/posts/edk2+u-boot+ECPT/Uboot-ECPT.png)


结合以上信息，我们已经可以大胆假设提出 Issue 的这位朋友应该是通过 u-boot 进入 UEFI 环境并启动到 UEFI Shell 下，然后查看到了这个错误的 GUID。

### 0x03 解决问题

首先要确定 u-boot 提交代码的流程。

在代码的 README 中可以找到这些信息。

```
Contributing
============

The U-Boot projects depends on contributions from the user community.
If you want to participate, please, have a look at the 'General'
section of https://docs.u-boot.org/en/latest/develop/index.html
where we describe coding standards and the patch submission process.
```

u-boot 是使用 email 来接收 patch 及 review 的。

根据提示我们可以先在 https://patchwork.ozlabs.org/project/uboot/list/ 注册一个账号（发送个 u-boot email 的 patch 最终可以在这个网站上看到），然后在 https://lists.denx.de/listinfo/u-boot 订阅一下。（不然在发送 patch 之后可能会收到 `Post by non-member to a members-only list` 的提示，不过即便订阅后也是需要审核，所以理论上不订阅好像也行）

接下来，我们需要创建 patch 然后发送到邮件列表，以供审核。

代码改动很简单，便不多说了。简单说一下提交的流程。

在本地修改为代码，并 commit 之后，使用 `git format-patch -1`，这会生成一个包含提交信息的 .patch 文件。

接着我们需要使用 u-boot 的 `./scripts/checkpatch.pl` 检查一下 patch/commit 的格式，如果不正确的话需要修改一下。

最后我们通过 `git send-email --to u-boot@lists.denx.de --cc xxx@xxx xxx.patch` 将改动发送到 mailing list。

### 0x04 后续

经过一段时间的等待，之前发送的 patch 已经出现在前面提到的两个网站中。maintainer 也给予了回复，并且对 patch 给出了 `Reviewed-by`。代码改动应该会在 u-boot 的某个合并窗口中合入到主线代码中。

我们可以在 https://lists.denx.de/pipermail/u-boot/2025-January/576425.html 查看到这些信息。

最后，回到 EDK2 的 Issue 上来，我回复了一些信息给他，包括使用的是否是 u-boot 环境，以及上面的 patch 信息。在 https://github.com/tianocore/edk2/issues/10579 可以看到，我们等待一下他的回复吧。

### 0xFF 后记

这是一个简单的改动，但是却连接了两个开源社区。通过这个改动，也对 u-boot 有了一些初步的了解（UEFI 方面）。后面也许可以再做一些更深入的探索。
