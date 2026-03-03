---
title: "CopilotChat.nvim使用指南"
date: 2025-06-25 21:12:00 +0800
tags: Neovim
categories: 技术
---


### 1.前言

  在当下AI辅助开发工具愈发花里胡哨的情况下，对于依旧使用**VIM**开发的“原始人”来说，需要寻找一些可以为自己的VIM平台赋能AI能力的工具。


---

### 2.待解决的问题

  * 代码补全
  * 代码review
  * 工作台内Chat
  * 文档总结
  * 代码解释
  * ...

---

### 3.配置copilot.lua && CopilotChat.nvim

* 1. 配置copilot.lua

  * 以下是我的配置，参考了官方推荐配置，做了一些小调整

  ```lua
    {
    "zbirenbaum/copilot.lua",
    cmd = "Copilot",
    event = "InsertEnter",
    config = function()
      require("copilot").setup({
        panel = {
          enabled = true,
          auto_refresh = false,
          keymap = {
            jump_prev = "[[",
            jump_next = "]]",
            -- accept = "<C-l>",
            refresh = "gr",
            open = "<M-CR>",
          },
          layout = {
            position = "bottom", -- | top | left | right
            ratio = 0.4,
          },
        },
        suggestion = {
          enabled = true,
          auto_trigger = true,
          debounce = 75,
          keymap = {
            accept = "<C-l>",
            accept_word = false,
            accept_line = false,
            -- next = "<C-]>",
            prev = "<C-[>",
            dismiss = "<C-]>",
          },
        },
        filetypes = {
          yaml = false,
          markdown = false,
          help = false,
          gitcommit = false,
          gitrebase = false,
          hgcommit = false,
          svn = false,
          cvs = false,
          -- ["."] = false,
        },
        copilot_node_command = "node", -- Node.js version must be > 16.x
        server_opts_overrides = {},
      })
    end,
  },
  ```

  * 配置CopilotChat.lua
  ```lua
    {
    "CopilotC-Nvim/CopilotChat.nvim",
    dependencies = {
      { "zbirenbaum/copilot.lua" }, -- or zbirenbaum/copilot.lua
      { "nvim-lua/plenary.nvim", branch = "master" }, -- for curl, log and async functions
    },
    build = "make tiktoken", -- Only on MacOS or Linux
    opts = {
      -- See Configuration section for options
      prompts = {
        ExplainCN = {
          prompt = "> /COPILOT_EXPLAIN\n\n简要概括一下选中的代码功能。",
        },
        CommitCN = {
          prompt = "> #git:staged\n\n为这一段提交编写一个摘要message",
        },
        Review = {
          prompt = "> /COPILOT_REVIEW\n\n帮我review一下选中的代码,用中文回答我",
          mapping = "<leader><F4>rc",
        },
        Fix = {
          prompt = "> /COPILOT_GENERATE\n\nThere is a problem in this code. Rewrite the code to show it with the bug fixed.",
          mapping = "<leader><F4>fc",
        },
        LeetCodeTeacher = {
          prompt = "当你准备好了，我会开始给你描述题目",
          system_prompt = "你是一名程序教培机构的资深算法老师，我会给你一些leetcode上的算法题目，你需要详细为我拆解、分析、总结题解，最重要的是总结一些同类型题目的特点以及下次碰到这种类型题时候初始思路应该是怎么样的，我使用的语言是golang",
          mapping = "<leader><F4>lt",
        },
      },
      window = {
        layout = "vertical", -- 'vertical', 'horizontal', 'float', 'replace'
        width = 0.4,
        -- height = 0.4, -- fractional height of parent, or absolute height in rows when > 1
        -- Options below only apply to floating windows
        relative = "editor", -- 'editor', 'win', 'cursor', 'mouse'
        border = "single", -- 'none', single', 'double', 'rounded', 'solid', 'shadow'
        row = nil, -- row position of the window, default is centered
        col = nil, -- column position of the window, default is centered
        title = "Copilot Chat", -- title of chat window
        footer = nil, -- footer of chat window
        zindex = 1, -- determines if window is on top or below other floating windows
      },
      mappings = {
        reset = {
          normal = "<C-n>",
          insert = "<C-n>",
        },
      },
      model = "gpt-5.1",
    },
    -- See Commands section for default commands if you want to lazy load on them
  },
  ```


  ### 4.CopilotChat.nvim

  [https://github.com/CopilotC-Nvim/CopilotChat.nvim](copilotChat.nvim)

  #### 4.1.使用

  Core Concepts

  * Resources(#<name>) - 添加特定内容(文件、git diff、urls)到提示词中
  * Tools(@<name>) - 授权LLM使用函数
  * Sticky Prompts(><text>) - 添加贯穿单次聊天的全局提示词
  * Models($<model>) - 选择当前聊天使用的模型
  * Prompts(/PromptName) - 在公共任务中使用预定义的提示词


