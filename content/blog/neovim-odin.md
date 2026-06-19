+++
title = "Neovim setup for Odin"
date = 2026-05-14
tags = ["Odin"]
draft = false
+++

Recently I have been experimenting with Neovim.<br />
This post is about how to set it up for Odin programming.

I will cover:

-   syntax highlighting
-   building/testing/formatting Odin code from Neovim
-   LSP
-   autocomplete
-   debugging


## Prerequisites {#prerequisites}

-   Install [Odin](https://odin-lang.org)
-   Install [ols and odinfmt](https://github.com/DanielGavin/ols) (LSP + formatting)
-   Install `tree-sitter-cli` (for installing Odin Tree-sitter grammar)
-   Install latest llvm and make sure lldb-dap is accessible (for debugging)
-   Set up [lazy.nvim](https://github.com/folke/lazy.nvim)


## Syntax highlighting {#syntax-highlighting}

I recommend using Tree-sitter for syntax highlighting.

For Odin syntax highlighting, Odin grammar should be installed.<br />
The simplest way is to use [nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter).

Add nvim-treesitter to lazy.nvim plugins and install it:

```lua
 {
     "nvim-treesitter/nvim-treesitter",
     lazy = false,
     build = ":TSUpdate",
     opts = {},
     config = function()
         vim.api.nvim_create_autocmd("FileType", {
             pattern = "odin",
             callback = function()
                 vim.treesitter.start()
             end,
         })
     end,
 }
```

Then execute `:TSIntall odin`.


## Building/testing/formatting {#building-testing-formatting}

I have a plugin for this: [odin.nvim](https://github.com/cephei8/odin.nvim).

Add it to lazy.nvim and install it:

```lua
{
    "cephei8/odin.nvim",
    lazy = false,
    opts = {},
}
```

Now it will be possible to build/test/format from Neovim, e.g. execute `:Odin build`.


### Per-project command overrides {#per-project-command-overrides}

`:Odin build` does `odin build .` from the project root directory (by default detected to be the one containing `.git` or `ols.json`).

It will likely be necessary to customize the build command on per-project basis - e.g. you may want to build with `odin build . -debug` etc.<br />
For this purpose, build command can be overriden using per-project `.nvim.lua` file - Neovim will detect and load it, and overrides can be done there.

So,<br />
(1) add `vim.opt.exrc = true` to your config (e.g. `init.lua`), so that Neovim will load `.nvim.lua`<br />
(2) create `.nvim.lua` file in your project root<br />
(3) add the following to `.nvim.lua` file:

```lua
require("odin").setup({
    commands = {
        build = { "odin", "build", ".", "-debug" },
    },
})
```

For convenience, I create keybindings for build/test commands (this goes to `init.lua`):

```lua
vim.keymap.set("n", "<leader>ob", "<cmd>Odin build<cr>", { desc = "Odin build" })
vim.keymap.set("n", "<leader>ot", "<cmd>Odin test<cr>", { desc = "Odin test" })
```


### Alternative to using odin.nvim {#alternative-to-using-odin-dot-nvim}

Before I created [odin.nvim](https://github.com/cephei8/odin.nvim), I used [overseer.nvim](https://github.com/stevearc/overseer.nvim).<br />
It is possible to create a custom task that runs build, parses output and redirects errors to quickfix list.

For running build/test tasks it is basically the same, it is just that overseer.nvim is a generic task runner, whereas odin.nvim is a purpose-build plugin.


## LSP {#lsp}

Setting up `ols` (language server for Odin) is simple with [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig).<br />
Add it to lazy.nvim and install it:

```lua
{
    "neovim/nvim-lspconfig",
    config = function()
        vim.lsp.enable("ols")
    end
}
```

I use the default Neovim keybindings for LSP, except that I also define the following ones:

```lua
vim.keymap.set("n", "gd", vim.lsp.buf.definition, { desc = "LSP definition" })
vim.keymap.set("n", "gD", vim.lsp.buf.declaration, { desc = "LSP declaration" })
vim.keymap.set({ "n", "v" }, "grf", function()
    vim.lsp.buf.format({ async = true })
end, { desc = "LSP format" })
```


## Autocomplete {#autocomplete}

[blink.cmp](https://github.com/Saghen/blink.cmp) is a good completion plugin.

Add it to lazy.nvim and install it:

```lua
{
    "saghen/blink.cmp",
    dependencies = {},
    version = "1.*",
    opts = {
        keymap = { preset = "default" },
        appearance = {
            nerd_font_variant = "mono",
        },
        completion = { documentation = { auto_show = false } },
        sources = {
            default = { "lsp", "path", "snippets", "buffer" },
        },
        fuzzy = { implementation = "prefer_rust_with_warning" },
    },
    opts_extend = { "sources.default" },
}
```

To make it work with LSP, modify the `nvim-lspconfig` lazy.nvim plugin entry:

```lua
{
    "neovim/nvim-lspconfig",
    opts = {
        servers = {
            ols = {},
        },
    },
    config = function(_, opts)
        for server, config in pairs(opts.servers) do
            config.capabilities = require("blink.cmp").get_lsp_capabilities(config.capabilities)
            vim.lsp.config(server, config)
            vim.lsp.enable(server)
        end
    end,
}
```


## Debugging {#debugging}

First of all, install `nvim-dap` and `nvim-dap-ui` (lazy.nvim):

```lua
{ "mfussenegger/nvim-dap" },
{ "rcarriga/nvim-dap-ui", dependencies = { "mfussenegger/nvim-dap", "nvim-neotest/nvim-nio" } },
```

Set up dap-ui and create keybindings for convenience (in `init.lua`):

```lua
local dap = require("dap")
vim.keymap.set("n", "<leader>dc", dap.continue, { desc = "DAP continue" })
vim.keymap.set("n", "<leader>db", dap.toggle_breakpoint, { desc = "DAP toggle breakpoint" })
vim.keymap.set("n", "<leader>dB", dap.set_breakpoint, { desc = "DAP set breakpoint" })
vim.keymap.set("n", "<leader>di", dap.step_into, { desc = "DAP step into" })
vim.keymap.set("n", "<leader>do", dap.step_over, { desc = "DAP step over" })
vim.keymap.set("n", "<leader>dO", dap.step_out, { desc = "DAP step out" })
vim.keymap.set("n", "<leader>dt", dap.terminate, { desc = "DAP terminate" })
vim.keymap.set("n", "<leader>dp", dap.pause, { desc = "DAP pause" })

local dapui = require("dapui")
dapui.setup()
dap.listeners.before.event_initialized.dapui_config = function() dapui.open() end
dap.listeners.before.event_terminated.dapui_config = function() dapui.close() end
dap.listeners.before.event_exited.dapui_config = function() dapui.close() end

vim.keymap.set("n", "<leader>du", dapui.toggle, { desc = "DAP UI toggle" })
vim.keymap.set({ "n", "v" }, "<leader>de", dapui.eval, { desc = "DAP UI eval" })
```

Now, in project's `.nvim.lua` add debug configuration (`.nvim.lua` file was explained in [Per-project command overrides](#per-project-command-overrides) section):

```lua
local dap = require("dap")
local lldb_dap = vim.fn.exepath("lldb-dap")

dap.adapters.lldb = {
	  type = "executable",
	  command = lldb_dap,
}
dap.configurations.odin = {
	  {
		    type = "lldb",
		    request = "launch",
		    name = "Odin Debug",
		    program = root .. "/app",
		    cwd = root,
		    args = {},
		    stopOnEntry = false,
		    preRunCommands = {
			      "command script import " .. root .. "/dev-util/odin_lldb.py",
		    },
	  },
}
```

Note that the configuration uses `dev-util/odin_lldb.py` from the project root - it adds LLDB formatter for Odin types (slices, strings etc.).<br />
It is optional but recommended, as it improves debugging experience.<br />
The formatters were described in [LLDB formatters]({{< relref "doom-emacs-odin#lldb-formatters" >}}) section from [Doom Emacs setup for Odin]({{< relref "doom-emacs-odin" >}}) post.

Start debugging by executing `:DapContinue`.


## Conclusion {#conclusion}

Enjoy Odin programming in Neovim!
