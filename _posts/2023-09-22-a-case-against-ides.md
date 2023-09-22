---
layout: post
title:  A Case Against IDEs
---

I've used PyCharm and Vim, and occasionally VSCode. I've enjoyed different features from each.
But I think in the long run Vim wins out.

PyCharm really does have some handy tools. There's a feature to display all occurrences of a string 
to replace and individually confirm each replacement. I find it useful whenever I have 
to port certain projects to new platforms. 90% of the work involves copying files and replacing all the instances of a 
particular string with the new platform name. Sometimes there are instances where you don't want to replace a particular occurrence.

To do something like that while using Vim, you could do it a few ways.

The first thing you could do is run `:vimgrep [pattern][file(s)]` manually go through the
matches with `:cnext` and `:cprev` and change each occurrence. Another option is to use `grep` 
to display all of the occurrences and again manually address each one. 
Or instead of manually modifying each occurrence, after you determine from the 
grep output that you want to modify all of the occurrences, you could run 
`find . -type f -exec sed -i 's/[search]/[replace] /g' {}\;` to do that.

Or you could take a different type of approach and scour the internet for a plugin that does it. 
I haven't used it, but [coc.nvim](https://github.com/neoclide/coc.nvim) 
(teehee) seems like a powerful plugin. It is compatible with both Vim and Neovim, 
although I'm not sure if the same capabilities are supported for both.

Sure, you could do any of those things, but they feel clunky. The CLI commands don't offer the
same granularity with the same ease. And you may not even have access to the internet in the 
moment to download the plugin. PyCharm has this functionality built in.

But PyCharm is so heavy. I've seen it take up 4GB of memory. It often freezes up 
and throttles the computer. For large projects, it's code completion engine
can stall for hours as the program indexes your project and generates the insights. 

VSCode is a lighter alternative that also has powerful code completion features. It's easy to download plugins and navigate project files. But code completion will struggle for larger projects, so it diminishes VSCode's usefulness overall. 

Large projects are really the main reason to get familiar with Vim, and that means also getting
familiar with CLI tools like `grep`, `less`, and `find`. In projects with deeply nested structures and lots of files strewn about, code completion is bad with most IDEs I've used. Especially in C projects. `ctags` is a great CLI tool for getting similar `goto` functionality in Vim, and it has the 
added benefit that you can seamlessly hop between definitions of symbols across languages. 
This is really helpful in projects that contain mixed code. And my Vim process instance is currently using a third of a gigabyte of memory and practically zero CPU time.

Honestly, the find and replace feature in IDEs is great, and there aren't many alternatives
that can stand up to it. But that's really the extent of their usefulness. After that they
just become big, slow monsters, or they can't handle the codebase and their code
completion becomes useless, or both.

You're better off switching to Vim and accepting the pain you have to endure when you 
occasionally have to do extensive renaming. If you really care that much, you could always
download PyCharm to use solely as a find and replace tool. 

