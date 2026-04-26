# codecompanion.nvim

- **Repo:** https://github.com/olimorris/codecompanion.nvim
- **Version:** v19.12.0 (latest release)
- **License:** Apache-2.0 (`LICENSE`)

## What it does

A Neovim plugin that turns the editor into a full AI coding workbench:
inline chat buffer, in-place edits, slash-command "actions" (refactor,
explain, test), agent/tool support, MCP client, and adapters for
Anthropic, OpenAI, Gemini, Ollama, OpenRouter, xAI, DeepSeek, Mistral,
HuggingFace, and others. Workflow is keyboard-first and integrates with
`tree-sitter`, `telescope`, and the LSP for context.

## Install

```lua
-- lazy.nvim
{
  "olimorris/codecompanion.nvim",
  dependencies = {
    "nvim-lua/plenary.nvim",
    "nvim-treesitter/nvim-treesitter",
  },
  opts = {
    strategies = {
      chat = { adapter = "anthropic" },
      inline = { adapter = "anthropic" },
    },
  },
}
```

## Usage

```vim
:CodeCompanionChat       " open chat split
:CodeCompanionActions    " palette of refactor/test/explain
:'<,'>CodeCompanion /buffer rewrite this as async
```

## Why it's interesting

The Neovim ecosystem has many "ChatGPT in a buffer" plugins; this one
goes further by treating the chat as a programmable workflow surface
with tools, agents, and an MCP client, while still feeling like
idiomatic Neovim. Works fully offline against Ollama and is the
strongest non-VSCode option if you want first-class agentic editing
without leaving the terminal.
