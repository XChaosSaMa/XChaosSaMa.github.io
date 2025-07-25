---
title: "My Terminal Setup for Hacking: Kitty, ZSH & Arch"
date: 2025-07-25
author: "ChaosSec"
tags: ["terminal", "zsh", "kitty", "linux", "arch", "setup"]
summary: "Here's how I configured my hacking terminal using Kitty, ZSH, and some spicy tweaks."
---

## 🐱 Why I Use Kitty

Kitty is a GPU-accelerated terminal emulator that's fast, configurable, and has built-in features that make my workflow smooth. I use it as my daily driver on Arch Linux for all hacking-related tasks.

- Super smooth scrolling
- URL highlighting and navigation
- Tab support with Powerline styling
- Ligature-friendly with custom font rendering

---

## 🎨 Theme and Colors

I'm using a modified [TokyoNight](https://github.com/folke/tokyonight.nvim) inspired color scheme.

Example colors:

- Background: `#1a1b26`
- Foreground: `#a9b1d6`
- Cyan: `#7dcfff`
- Magenta: `#bb9af7`
- Cursor: `#c0caf5`
- Tabs: Active = `#98c379`, Inactive = `#e06c75`

I manage this with a `color.ini` and include it in my `kitty.conf` like so:

```conf
include color.ini
```

---

## 🧱 Font and Look

- **Font:** Hack Nerd Font
- **Size:** 11px
- **Opacity:** 95%
- **Cursor:** Beam (with custom thickness)
- **Tabs:** Powerline style with margin colors

---

## ⌨️ Keybindings

I've added custom navigation bindings:

```conf
map ctrl+left  neighboring_window left
map ctrl+right neighboring_window right
map ctrl+up    neighboring_window up
map ctrl+down  neighboring_window down
```

---

## ⚙️ Shell: ZSH + Powerlevel10k

I use ZSH with the [Powerlevel10k](https://github.com/romkatv/powerlevel10k) theme. Some of my favorite features:

- Git status segments
- Time display
- Background job indicators
- Clean prompt with icons

My shell is set via Kitty config:

```conf
shell zsh
```

---

## 🧩 Plugins and Features

From `.zshrc`, some active features:

- `zsh-autosuggestions`
- `zsh-syntax-highlighting`
- Fast alias expansion
- Custom functions for navigation and pentesting utilities

---

## 📸 Final Result 

<img src="/images/terminal.png" alt="My terminal setup" width="700" style="display:block; margin:auto;"/>

This setup makes my terminal feel smooth, powerful, and personalized for pentesting and daily workflow. If you're into hacking + aesthetic terminals — Kitty + ZSH is a solid combo.

---

Want to see more of my tooling? Hit me up at [Github](https://github.com/XChaosSaMa).
