---
layout: post 
title: How to determine direct instances in Maya?
date: '2011-10-16T14:47:00.000+08:00'
categories: []
tags: [MayaAPI, Python]
published: True

---

Any instanced node in Maya has multiple parent transforms, therefore, the basic idea to determine the instanced node is to check the number of its parent nodes. However, if we have <span class="blue">_both instanced shapes and some other transform nodes from the same hierarchy_</span>, it gets a little harder to handle this situation just by MEL.

Let's consider the following case:

1. Create an instance of pCube1 as pCube2.
2. Group them into group1.
3. Create another instance group2 from group1.

![Maya instances](/images/maya/maya_instanced_case.jpg)

Okay, now our goal is to find out the direct instances (indicated by the dash lines). How could we achieve this purpose with Maya API? 

# MItDag.isInstanced(indirect=<span class="blue">False</span>)

We all don't wanna parse the strings of all dag paths to figure out these direct instances, don't we? Fortunately, `MItDag.isInstanced` is what we need! But there is one thing should be aware of: <span class="orange">isInstanced() returns true for not only the spawned instances but also the original ones (i.e. group1\|pCube1\|pCubeShape1).</span>

In order to distinguish the original from these nodes qualified by isInstanced(), we need to examine the identifications of their parent transforms further.

{% highlight python linenos %}
import maya.OpenMaya as om
dagIter = om.MItDag(om.MItDag.kBreadthFirst)
dagPath = om.MDagPath()
fnDagNode = om.MFnDagNode()
 
detectIndirect = False
 
while not dagIter.isDone():
    if dagIter.isInstanced(detectIndirect):
 
        dagIter.getPath(dagPath)
        dagPath.pop(1)
 
        fnDagNode.setObject(dagIter.currentItem())
 
        if fnDagNode.parent(0) != dagPath.node():
            print dagIter.fullPathName()
 
    dagIter.next()
{% endhighlight %}

Here is the result:
<samp>|group2|pCube1
|group2|pCube2
|group1|pCube2|pCubeShape1
|group2|pCube2|pCubeShape1</samp>

<span class="red">__Note:__</span> _"dagPath.instanceNumber() != 0"_ is not a sufficient condition for `line#16`, since it can't filter out "group2\|pCube1\|pCubeShape1"! 

# Reference

* <cite>[How do I determine if a shape node has been instanced?](http://ewertb.soundlinker.com/mel/mel.085.htm)</cite> by Bryan Ewert.
