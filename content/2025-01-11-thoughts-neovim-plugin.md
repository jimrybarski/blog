+++
title = "Some initial thoughts on developing Lua plugins for Neovim"
date = "2025-01-11"

[taxonomies]
tags = ["neovim"]
+++
I wrote a [bioinformatics Neovim plugin](https://github.com/jimrybarski/bioinformatics.nvim) recently. It's nothing super special - it just provides some sequence manipulation functionality like generating or searching for a reverse complement, performing a pairwise alignment, etc. It really surprised me how often I found myself using it in day-to-day work right after I installed it.

This was my first Lua codebase and it definitely left me underwhelmed with the language. I mean, it's fine - it's a very small language and you can get proficient pretty quickly. First-class functions are easy and it feels like you're being guided by the language towards using them. But overall, it just feels unpolished and I simply don't understand the [enthusiasm](https://nflatrea.bearblog.dev/lua-is-so-underrated/) for it. Some issues I had:

- The only compound data type is a table (an associative array) that returns `nil` when unmapped keys are accessed. So more or less the same API as `defaultdict` in Python. This can make debugging a nightmare, as a typo in a field name where `nil` is an expected value makes it seem like the value just never gets set. Maybe this is less of a problem with experience, but it's a completely unforced error. Other languages simply don't have this problem because they have data types that make this error impossible.
- Variables are scoped globally by default, with opt-in local scoping. I just can't imagine a scenario where this is preferable.
- The lack of a built-in unit testing framework is disappointing. I had [trouble](https://github.com/notomo/vusted/issues/23#issuecomment-2564888517) getting Vusted (a tool for unit testing Neovim plugins) installed because I didn't run the installer as root. What is the deal with modern language-specific package managers requiring root (thinking of the node ecosystem in particular here)? My only guess is that this is easier for people who don't understand how to set their shell's path and this is a strategy for not discouraging less-experienced programmers. But that sort of person is just copying-and-pasting anyway, and you can just do what Rustup does: add something to their .bashrc and print a little note about how they need to restart their terminal to be able to immediately use their new program.
- The `vim` object in Neovim plugins can't be accessed by an LSP, so you're unable to view or reference the internal Neovim API, and it just appears to the LSP like an undefined variable. It's possible to disable those lints on a per-variable basis, but this is still pretty suboptimal. There does seem to be [plugin](https://www.reddit.com/r/neovim/comments/x3bd4i/how_can_i_get_lsp_to_recognize_builtin_neovim_api/) for this so maybe this is just growing pains.
- Unintuitive naming for built-in functions, like `#variable_name` to get the length of a string or `gsub()` to do replacement. What does that "g" stand for? It might be "global", but I haven't found a source for this that wasn't just speculation.

That said, once you figure out the idioms and boilerplate, plugin development becomes super easy. The language provides so few abstractions that there's really only one way to do anything, which I appreciate.
