---
title"Font resizing with urxvt+tmux"
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

However if we are in a tmux session then using these sequences directly won't work. We would first need to detach from tmux, enter the escape sequence, and then reattach the session. While detaching must be done with the <prefix>+d shortcut in tmux, executing the correct escape sequence and reattaching can be done more conveniently in a bash script, for example:

{% highlight bash %}
#!/bin/bash
printf '\33]50;%s\007' "xft:DejaVu Sans Mono-$1"

session=`
tmux -2 list-sessions |\
awk '
BEGIN{}
{
  if ($11 == "")
    {
      print $1
    }
}
END{}
' | sed 's/://'
`
tmux -2 attach-session -t $session
{% endhighlight %}

The first part of the script just executes the escape sequence to resize the font using a value given as an argument to the script. The second part attempts to find the detached tmux session by looking at the last column in the list-sessions ouput, shown below. 

{% highlight bash %}
> tmux -2 list-sessions
0: 1 windows (created Sat Jan  2 22:36:31 2018) [227x58] (attached)
1: 2 windows (created Sat Jan  2 22:38:22 2018) [273x71] (attached)
2: 1 windows (created Sun Jan  3 16:47:46 2018) [202x31]
{% endhighlight %}

The session we have detached from has no "(attached)" column, so we know the session ID in the first column is 2 (of course if we have multiple detached sessions this method might not return the correct value, but this isn't an issue for my workflow). The sed invocation just removes the semicolon after the retrieved session ID.

To use the script we just make an alias to it within our shell and invoke it with the desired font size as an argument.

## Improvements

The script as shown above is a bit verbose for what it does, and also uses "sed" when it would be more efficient to remove the pipe and use the "sub" function availabe within "awk". The whole script could be condensed into one line like so:

{% highlight bash %}
printf '\33]50;%s\007' "xft:DejaVu Sans Mono-$1"; tmux -2 attach-session -t `tmux -2 list-sessions | awk '{if ($11 == ""){str = $1; sub(/:/,"",str); print str}}'`
{% endhighlight %}
