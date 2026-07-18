+++
title = "Things that made me come back to Emacs"
date = 2026-07-18
draft = false
+++

TLDR: Neovim was good, I just personally like Emacs better (especially Org-Mode).

---

I think I can call my experiment to try out Neovim finished, and I am happy to be back to Emacs.

I used to be a GUI Emacs user.<br />
The motivation to try out Neovim was the rise of AI agentic workflows, and that the AI agent harnesses worked best in the "proper" terminals (Claude Code was flickering and jumping around in Emacs's vterm, and it wasn't a good experience).<br />
Thus, I had to move my terminal out of my editor and instead bring my editor into my terminal. Additionally, AI agent sessions are often long-living, so tmux became a requirement (not saying that I couldn't have tmux in my Emacs's vterm, probably I could, it was just another reason to shift from GUI Emacs to a terminal-based workflows).

After moving to terminal+tmux, I tried out TUI Emacs, but some shortcuts weren't working (SHIFT-\* or OPT-\* sequences weren't propagated properly or something like that, I don't remember), I had to remap some stuff, and out of curiosity I decided to try out Neovim.

Neovim was good, most of what I did in Emacs I could do in Neovim, and the plugins mostly gave me experience I wanted (e.g. Neogit vs Magit, oil.nvim vs Dired).

It's just that I had everything working better for me in Emacs.<br />
Particularly, I missed:

-   Org-Mode (publishing, Ox-Hugo, Org-Roam)
-   Dired
-   Embark
-   Magit

So,

-   now I use TUI Emacs mostly (in terminal+tmux)
-   Emacs also got [ghostel](https://github.com/dakra/ghostel) now, which gives me pretty good experience when I use GUI Emacs with embedded terminal (maybe also AI agent harness improved to run smoother)
