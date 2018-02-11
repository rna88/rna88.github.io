---
title: "Custom Code Folding in Vim"
layout: post
date: 2018-01-24 00:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
- code
- folding
- vim
- commands
- regex
blog: true
star: true
author: rna88
description: Using commands to automatically fold code in vim.
---

## A Better Way

While Vim has built in support for folding based on various methods. neither of these options worked well for a particular codebase I was dealing with. However through Vim's command mode it is possible to write small scripts that will fold code in any way you find  convenient.

## Folding Allman Style Brackets

Typically I use Allman style indentation for C++ projects, where every bracket is on a new line, seen below:

```cpp
if ( true )
{
	std::cout << "Hello World\n";
}
```

To produce an automatic fold for this style of indentation we first have to understand how to make a manual one, to do so we must: 

1. Place the vim cursor on the line or character where we want the fold to start
2. Press z+f
3. Make a movement to where we want the fold to end

For example to fold from an open bracket to a closing one the key sequence would be `zf%`, and we can also open folds using `zo`. The results of using these key sequences to fold brackets is shown below.

![Markdown Gif][1]

Through Vim's command mode we can run any set of keystrokes using the [normal!](http://learnvimscriptthehardway.stevelosh.com/chapters/29.html#avoiding-mappings) keyword (note that we can type `:help :<command>`into Vim to get the details of any of its functions). Now if we place the cursor over an open bracket and type `:normal! zf%zo` Vim will create a fold to the corresponding closed bracketi, and then open it. 

To have fully automatic code folding we need to globally apply the previous set of commands across every open bracket that is on its own line. Vim allows us to use regular expression pattern matching for this task; we can try typing `:help :global` to view the syntax details, shown below:

```vim
:[range]g[lobal]/{pattern}/[cmd]
                        Execute the Ex command [cmd] (default ":p") on the
                        lines within [range] where {pattern} matches.
```

Putting the parts together we end up with our final command:


```
:%g/^[ \t]*{/ normal! zf%zo
```

Where,

`%` The range over which the regex is applied, in this case the entire file.
`g` Tells Vim to apply the regex globally across the range.
`/^[ \t]*{/` The regex itself, contained within forward slashes.
`normal! zf%zo` The command to run.

To elaborate a little on the regex itself:  

`^` Start matching from the beginning of the line.
`[ \t]*` Match any combination of spaces and tabs, 0 or more times.
`{` The character we are after.


 To make using the script more convenient, we can make a custom mapping to it by adding the following line to our vim config:

```
:command Refold %g/^[ \t]*{/ normal! zf% zo  
```

![Markdown Gif][2]


## Folding Java style Brackets

In the same way we derived a method to fold Allman style brackets, we can fold any other style. For instance the following script will fold Java style indentation:

```
:%g/.*{$/ normal! $zf% zo
```

![Markdown Gif][3]

## References 

http://learnvimscriptthehardway.stevelosh.com/chapters/29.html#avoiding-mappings

[1]: /assets/gifs/manualFolding.gif
[2]: /assets/gifs/autoFolding.gif
[3]: /assets/gifs/JautoFolding.gif
