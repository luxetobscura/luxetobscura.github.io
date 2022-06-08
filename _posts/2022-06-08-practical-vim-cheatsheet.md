---
layout: post
title: ":memo: Practical vim Cheatsheet"
date: 2022-06-08
---
This is my `vim` cheatsheet. There are many like it, but this one is mine.

<img src="../assets/get-that-vim-out-of-my-face.jpg" alt="" width="500"/>

This is mostly me trying to answer my perpetual question of "How did I do that one thing in `vim` that one time?" without losing valuable coding brain cycles flicking through tens of reference tabs. Here you go, future me. You're welcome.

This is also part of my attempt to force myself into using `vim`. As the quote goes, "the shortest `vim`-edited Markdown document is longer than the longest memory." Or something.

This cheatsheet may be useful for the following audiences:
- Those who have a `vim` cheatsheet with basic commands in front of them, but still find using `vim` painfully awkward and aren't sure how to put it all together (i.e. me :sob:).
- Windows sysadmins who touched a BSD/Linux box once and ended up owning it for their team/organization (also me :sweat_smile:).
- Me. I made this for myself. If you happen to find it useful, that's awesome too!

## Exiting
"Oh no, I tried editing a file and it opened in `vi`/`vim`!" Pick your antidote.

| Save | Exit  | Save and exit |
| ---- | ----  | ------------- |
| `:w` | `:q`  | `:wq`         |
|      | `ZQ`  | `ZZ`          |
|      |       | `:x`          |

## Navigation
- `5G`: Go to line `5`

## Deleting stuff in insert mode
- `Ctrl-w`: delete previous word
- `Ctrl-u`: delete current line

## Insert-normal sub mode (execute normal mode commands from insert mode)
General syntax:
```
Ctrl-o [command]
```

Undo a mistake in insert mode:
```
Ctrl-o u
```

## Search and replace
General syntax:
```
:[range]s/pattern/string/[flags] [count]
```

Within the current line, replace the first occurrence of `x` with `y`:
```
:s/x/y/
```

Within the current file (`%`), replace all occurrences of `x` with `y`:
```
:%s/x/y/g
```

Between lines `5` and `10`, replace all occurrences of `x` with `y`:
```
:5,10s/x/y/g
```

## Copy inside elements
- Copy selected word:
  - `yiw` (alphanumeric word only)
  - `yiW` (variable names)
- Copy inside quotes:
  - `yi'`
  - `yi"`
- Copy inside brackets/braces:
  - `yi[`
  - `yi(`
  - `yi{`
  - `yi<`

## Multiple line edit
1. `Ctrl-v` (visual block mode)
2. `j`/`k` (select lines; other navigation commands like `gg`/`G` also work)
3. `I/A` (visual block insert mode)
4. Perform desired edits
5. `Esc` (apply changes and return to visual block mode)

![](/assets/multi-line-edit.gif)

Alternatively, for a few lines:
1. `i`/`a` (insert mode)
2. Perform desired edits
3. `Esc` (return to normal mode)
4. `j`/`k` (navigate to next line)
5. `.` (repeat last edit)
6. Repeat until done

![](/assets/multi-line-edit-2.gif)

## Immersion
The fastest way to learn is to force yourself to fully immerse. Coming from a Windows consumer/administrator background, here are the methods I've used ~~to make my life harder~~.
- VSCode: [vscodevim](https://marketplace.visualstudio.com/items?itemName=vscodevim.vim)
  - Make sure to enable the "Use System Clipboard" setting: `"vim.useSystemClipboard": true`
- Sublime Text 3: [Vintageous](https://packagecontrol.io/packages/Vintageous)
- PowerShell: `Set-PSReadLineOption -EditMode vi`
  - The best way to use this is to integrate it into your PowerShell profile. [Here's mine if you're interested](https://github.com/luxetobscura/luxetobscura-PowerShellProfile), complete with setup steps.

## Resources
Shoutouts to the people who make learning `vim` possible.
- [`vimtutor`](https://linux.die.net/man/1/vimtutor) is the OG. If you're completely new to `vi`/`vim`, find your nearest Linux machine and fire up `vimtutor` first before you go any further.
- [ViEmu's graphical cheat sheet](http://www.viemu.com/a_vi_vim_graphical_cheat_sheet_tutorial.html) is probably the only printed piece of paper I keep at my desk nowadays.
- [Linuxize.com](https://linuxize.com) has an insane number of no-BS Linux tutorials, including a bunch for `vim`.
- [Vimhelp.org](https://vimhelp.org) is the official website for `vim`'s help pages. Prepare to have your queries answered in excruciating detail.

## To do:
- [ ] Add short animated clips for more complex tasks
- [ ] Register manipulation
- [ ] External commands
- [ ] Multiple file management
- [ ] Stop confusing the `o` and `O` commands
