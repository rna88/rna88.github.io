---
title: "Font resizing with urxvt+tmux"
layout: post
date: 2018-01-04 00:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
- font
- resizing
- urxvt
- tmux
blog: true
star: true
author: rna88
description: How to resize fonts.
---

## The Problem
Since switching from Terminator to urxvt as my default terminal emulator, I had been missing the ability to resize the terminals font. While Terminator has keyboard shortcuts for this feature, a bit more work is required to get similar functionality with urxvt.

## A Stop-gap Solution 
In urxvt it is possible to [resize fonts using an escape sequence](https://man.cx/urxvt(1)#heading12), for instance:

{% highlight bash %}
printf '\33]50;%s\007' "xft:DejaVu Sans Mono-12"
{% endhighlight %}

Entering this sequence into a urxvt terminal will change the font to "DejaVu Sans Mono" and size it to 12.  

However if we are in a tmux session then using these sequences directly won't work. We would first need to detach from tmux, enter the escape sequence, and then reattach the session. While detaching must be done with the <tmux prefix>+d shortcut, executing the correct escape sequence and reattaching can be done more conveniently in a bash script, for example:

{% highlight bash %}
  1 #!/bin/bash
  2 printf '\33]50;%s\007' "xft:DejaVu Sans Mono-$1"
  3 
  4 session=`
  5 tmux -2 list-sessions |\
  6 awk '
  7 BEGIN{}
  8 {
  9   if ($11 == "")
 10     {
 11       print $1
 12     }
 13 }
 14 END{}
 15 ' | sed 's/://'
 16 `
 17 tmux -2 attach-session -t $session
{% endhighlight %}

The first part of the script at line 2 just executes the escape sequence to resize the font using a value given as an argument to the script. The second part starting from line 4 attempts to find our detached tmux session by examining the list-sessions ouput, shown below. 

{% highlight bash %}
> tmux -2 list-sessions
0: 1 windows (created Sat Jan  2 22:36:31 2018) [227x58] (attached)
1: 2 windows (created Sat Jan  2 22:38:22 2018) [273x71] (attached)
2: 1 windows (created Sun Jan  3 16:47:46 2018) [202x31]
{% endhighlight %}

The first column of output lists the tmux session ID, and the last column whether the tmux session is currently attached. The script uses "awk" to check whether the last column is empty, and if so passes the first column to *sed*, where the colon at the end of the session ID is removed. Passing the ID to the attach-session command completes the script.

To use the script we just make an alias to it within our shell and invoke it with the desired font size as an argument after detaching from tmux.

## Final Improvements

While the script is pretty small and works fine as is, it doesn't hurt to optimize it if we can. The main inefficiency with the previous implementation lies with our usage of *sed*, as it introduces a needless pipe operation. Awk itself has the ability to handle regular expression based search-and-replace operations through the "sub" function. Making use of this function, removing the BEGIN and END sections, and removing the explicit "sessions" variable we get the following:


{% highlight bash %}
  1 #!/bin/bash
  2 printf '\33]50;%s\007' "xft:DejaVu Sans Mono-$1"
  3 
  4 tmux -2 attach-session -t `
  5 tmux -2 list-sessions |\
  6 awk '
  7 {
  8   if ($11 == ""){
  9     str = $1
 10     sub(/:/,"",str)
 11     print str
 12   }
 13 }
 14 ' `   
{% endhighlight %}

If we wanted to, we could further condense things into a one-liner like so:

{% highlight bash %}
printf '\33]50;%s\007' "xft:DejaVu Sans Mono-$1"; tmux -2 attach-session -t `tmux -2 list-sessions | awk '{if ($11 == ""){str = $1; sub(/:/,"",str); print str}}'`
{% endhighlight %}
