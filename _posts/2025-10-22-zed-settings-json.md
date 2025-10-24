---
title: Zed 编辑器配置分享
date: 2025-10-22 00:00:00 +0800
categories: [IDE]
tags: [ide, zed, rust]     # TAG names should always be lowercase
---

随着 [Zed](https://zed.dev/) 在 Windows 上的逐渐完善（现在已经正式发布 stable 版本），所以最近一直都在使用它作为日用编辑器。虽然还未迎来 non-UTF8 的支持，但是只是用来写/看代码似乎也够用了。重要的是内存占用低，并且键位与 VSCODE 类似（或者说一样），可以快速上手。

但是作为一个编辑器，它还是需要做一些配置，才能用起来更舒服，下面分享一下我的配置及简单的说明（部分）：

```
{
  "project_panel": {
    "button": true
  },

  "agent": {
    "default_model": {
      "provider": "google",
      "model": "gemini-2.5-flash"  // 免费
    },
    "use_modifier_to_send": true,  // 防误触
    "single_file_review": true
  },

  "icon_theme": "Material Icon Theme",
  "theme": "Catppuccin Mocha",  // 这个主题感觉不错

  "ui_font_family": "Source Code Pro",  // 字体，个人喜好
  "ui_font_size": 18,  // 大了还是小了？
  "buffer_font_family": "Source Code Pro",  // 字体，个人喜好
  "buffer_font_size": 18,
  "agent_ui_font_size": 18,

  "tab_size": 2,  // 你用啥？
  "format_on_save": "off",  // 能跑就别动了
  "remove_trailing_whitespace_on_save": false,  // 能跑就别动了
  "ensure_final_newline_on_save": false,  // 能跑就别动了

  "title_bar": {
    "show_branch_icon": true,
    "show_menus": true  // 汉堡菜单不喜欢
  },

  "git": {
    "inline_blame": {  // 谁把代码改崩了？
      "enabled": true,
      "show_commit_summary": true
    }
  },

  "git_hosting_providers": [  // 似乎不如 GitLens 好用
    {
      "provider": "gitlab", // GitHub, GitLab, Bitbucket, SourceHut and Codeberg?
      "name": "Self Host Git",
      "base_url": "http://192.168.1.110/"  // 记得换
    }
  ],

  "tabs": {
    "git_status": true
  },

  "minimap": {
    "show": "always"  // 你用这个来干嘛？
  },

  "lsp": {
    "clangd": {  // 代码跳转全靠它了
      "binary": {
        "path": "D:\\clangd\\bin\\clangd.exe",
        "arguments": ["--compile-commands-dir=.edkCode", "--header-insertion=never"]  // 谁能给 Zed 做个 EdkCode 扩展？
      }
    },
    
  }
}
```
