+++
title = "Some initial thoughts on Neovim and Lua plugin development"
date = "2025-01-11"

[taxonomies]
tags = ["neovim"]
+++
I wrote a [bioinformatics Neovim plugin](https://github.com/jimrybarski/bioinformatics.nvim) recently. It's nothing super special - it just provides some sequence manipulation functionality like generating or searching for a reverse complement, performing a pairwise alignment, etc. It really surprised me how often I found myself using it in day-to-day work right after I installed it.

This was my first Lua codebase and it definitely left me underwhelmed with the language. I mean, it's fine - it's a very small language and you can get proficient pretty quickly. First-class functions are easy and it feels like you're being guided by the language towards using them. But overall, it just feels unpolished - the only heterogenous data type is a table (an associative array) that returns `nil` when unmapped keys are accessed. So more or less the same API as `defaultdict` in Python. This can make debugging a nightmare, as a typo in a field name where `nil` is an expected value makes it seem like the value just never gets set. Maybe this is less of a problem with experience, but it would still be preferable if static analysis tools could detect this kind of error.

I could go on - default global values with opt-in local scoping is definitely getting things backwards. The lack of a built-in unit testing framework is also disappointing. I had [difficulties](https://github.com/notomo/vusted/issues/23#issuecomment-2564888517) getting Vusted (a tool for unit testing Neovim plugins) installed because I didn't run the installer as root (why would you ever need root?). Finally, because the `vim` object in Neovim plugins can't be accessed by an LSP you're unable to view or reference the internal Neovim API, and it just appears to the LSP like an undefined variable. It's possible to disable those lints on a per-variable basis, but this is still pretty suboptimal. There does seem to be [plugin](https://www.reddit.com/r/neovim/comments/x3bd4i/how_can_i_get_lsp_to_recognize_builtin_neovim_api/) for this so maybe this is just growing pains.

That said, once you figure out the idioms and boilerplate, plugin development becomes super easy. The language provides so few abstractions that there's really only one way to do anything, which I appreciate.
