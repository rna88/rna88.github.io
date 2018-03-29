---
title: "Spawning Processes from the Shell"
layout: post
date: 2018-02-15 00:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
- process
- spawn
- shell
- disown
- nohup
- setsid
blog: true
star: true
author: rna88
description: Describes 3 methods to spawn process from shell
---

## Multitasking in the Shell

One of the first annoyances I encountered when SSH'ing into remote machines was running a one-off process or script from the terminal without having its output interfere with the rest of my session, or having the script die as soon as I lose connection to the host. Fortunately we can silence a processes stdout/stderr streams by adding `>/dev/null 2>&1`to its invocation. Also we can either append `&` to the previous call  or hit CTRL+z to move the process to the background, preventing the terminal from being blocked while the process runs. 

```
<process> >/dev/null 2>&1 &
```

Unfortunately if we lose connection or close our session any process that we kicked off will be killed. This occurs due to the process being spawned as a child of our terminal. You can confirm this by checking the output of the command `pstree` which shows a hierarchy of the processes running on your system, a snippet of which is shown below. 

```
pool@ABCgeeb ~ ➜  ./runloop.sh >/dev/null 2>&1 &      
[1] 30330                                                                      
pool@ABCgeeb ~ ➜  pstree | grep -B 2 -A 0 runloop     
        |              `-zsh-+-grep                                            
        |                    |-pstree                                          
        |                    `-runloop.sh---sleep                              
```

Notice how the *runloop*, *pstree*, and *grep* processes that we have ran in the shall all show up as its children. The shell itself is also a child of another process, in my case tmux (not shown above). To prevent our *runloop* process from being killed when its parent zsh dies, we need to change its relationship with the shell process.

We'll cover 3 potential methods to deal with this problem.

## Disown

If we are using bash, zsh, or ksh as our shell, we can use the `disown` command to remove any process from the shells job list, thereby preventing the process from terminating if the shell closes. The following command trace shows a sample: 

```
pool@ABCgeeb ~ ➜  ./runloop.sh >/dev/null 2>&1 &
[1] 9908
pool@ABCgeeb ~ ➜  disown %./runloop.sh
pool@ABCgeeb ~ ➜  exit
```

Opening a new terminal and running pstree, we can see that the *runloop* process is still running, and is parented to the root process. 

```
pool@ABCgeeb ~ ➜  pstree | grep -B 1 -A 1 runloop
        |-rtkit-daemon---2*[{rtkit-daemon}]                                    
        |-runloop.sh---sleep                                                   
        |-sh---tint2                                                           
```

One of the limitations of this method is that it is a two step procedure: we have to first run our process, then disown it. However the major disadvantage is that `disown` is limited to bash/zsh/ksh, so a more generic alternative is preferable. 

## Nohup

The `nohup` command can be used to prevent any process from receiving the `SIGHUP` signal, preventing the process from dying when its parent shell closes. For example:

```
pool@ABCgeeb ~ ➜  nohup ./runloop.sh >/dev/null 2>&1 &
[1] 6848
pool@ABCgeeb ~ ➜  pstree | grep -B 1 -A 1 runloop
        |              |     `-pstree
        |              |-zsh---runloop.sh---sleep
        |              `-zsh---vim-+-python2---31*[{python2}]
```

Closing the terminal and running `pstree` again in a new window we see that *runloop* is parented to the root process, like in our previous example with `disown`:

```
pool@ABCgeeb ~ ➜  pstree | grep -B 1 -A 1 runloop
        |-rtkit-daemon---2*[{rtkit-daemon}]
        |-runloop.sh---sleep
        |-systemd-+-(sd-pam)
```

One of the advantages of `nohup` is that its behaviour is defined in the POSIX standard, so we can be confident it behaves as expected on POSIX compliant systems. However `nohup` does not separate the process from the parent shells job list, unlike `disown`. This can be a problem if you often switch between background/foreground jobs.

## Setsid

Another POSIX standard command we can use is `setsid`. Running any process with `setsid` will [effectively](http://pubs.opengroup.org/onlinepubs/9699919799/) detach it from the parent shell as well as remove it from the parents job list. 

```
pool@ABCgeeb ~ ➜  setsid ./runloop.sh > /dev/null 2>&1
pool@ABCgeeb ~ ➜  pstree | grep -B 1 -A 1 runloop
        |-rtkit-daemon---2*[{rtkit-daemon}]
        |-runloop.sh---sleep
        |-systemd-+-(sd-pam)
```

Notice how the *runloop* process is immediately parented to the root process. This is due to the process being placed in its own session, hence the parent terminals status has no effect on it, Of all the methods discussed, `setsid` is the most painless way of kicking off terminal jobs that you don't intend to interact with.

## But what about screen or tmux?

While it's possible to use screen/tmux sessions as a solution, if you just want to essentially daemonize a process these tools are a bit heavy-handed compared to the alternatives discussed. Also it's important to keep in mind workflow portability - i.e. tmux may not be available on every machine, and you may not have the privileges to install it either.


## References

For some more information on some of the topics outlined in this article, you can check the following:

<http://pubs.opengroup.org/onlinepubs/9699919799/functions/setsid.html>
<https://askubuntu.com/questions/611968/differences-between-command-disown-and-nohup-command-disown>
<https://unix.stackexchange.com/questions/3886/difference-between-nohup-disown-and#148698>
<http://www.enderunix.org/docs/eng/daemon.php>
