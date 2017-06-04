---
categories: []
date: 2011-09-19T14:53:00Z
published: true
tags:
- Python
title: Retrieve file list from a directory tree
---

There is a simple way to traverse a directory tree with python - `os.walk` (os.path.walk is deprecated in 3.0, therefore, we had better avoid to use it!)
<!--more-->

<pre class="prettyprint linenums lang-python">
import os
def trav1(startDirPath):
    result = []
    for curDirPath, subDirList, fileNameList in os.walk(startDirPath):
        filePathList = [os.path.join(curDirPath, x) for x in fileNameList]
        result.extend(filePathList)
    return result
</pre>

Maybe it's sufficient in most cases. However, if you ran into a time-critical situation, you may need a more efficient solution, and it's the show-time for dir command (in Windows, of course).

<pre class="prettyprint linenums lang-python">
import subprocess as sp
def trav2(startDirPath):
    result = []
    args = ["dir", startDirPath, "/s/b/og"]
    routine = sp.Popen(args, shell=True, stdout=sp.PIPE)
    for line in routine.stdout:
        result.append(line.strip())
    return result
</pre>

This trick is just execute a dir command from another `sub-process`, and read results from the pipe. It's simple but efficient for me so far.

Finally, we can use cProfile to examine the efficiency of both methods:

<pre class="prettyprint linenums lang-python">
if __name__ == "__main__":
    import cProfile
    startDirPath = "D:\\photo" # my test directory
    cProfile.run("x = trav1(startDirPath)")
    cProfile.run("y = trav2(startDirPath)")
</pre>

The result (with 16368 items) performed by my PC is as below:
<samp>trav1=> 985949 function calls (983460 primitive calls) in 1.733 CPU seconds
trav2=> 52594 function calls (51751 primitive calls) in 0.635 CPU seconds
</samp>

Desperate times call for desperate measures, dir command is a nice alternative to me!! ;)
