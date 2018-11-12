---
title: Maya
published: true
---

# General Resources

* [Maya Developer Center](https://www.autodesk.com/developer-network/platform-technologies/maya)
* [Maya Documentation](http://knowledge.autodesk.com/support/maya/learn-explore/caas/CloudHelp/cloudhelp/ENU/123112/files/maya-documentation-html.html)
* [Maya Learning Channel](https://www.youtube.com/user/MayaHowTos) @YouTube
* [Maya Training Material](https://github.com/ADN-DevTech/Maya-Training-Material) (a little outdated, but still useful)
* [Duncan's Corner](http://area.autodesk.com/blogs/duncan)
* [Maya Python programming for production](http://zurbrigg.com/tutorials/) by Chris Zurbrigg.

## Maya Programming Forums

* [Autodesk AREA](http://forums.autodesk.com/t5/programming/bd-p/area-b50)
* [CG-Society](http://forums.cgsociety.org/forumdisplay.php?f=89)
* [Highend3D](http://www.creativecrash.com/forums/api/topics)

# Programming

* [Maya Scripting Primer]({{< ref "2018-11-12-maya-scripting-primer.md" >}}).
* [New To The Api? Then Read This!](http://www.creativecrash.com/forums/api/topics/new-to-the-api-then-read-this-33) by Robert Bateman.
* [Maya API: Where to start!?!??!?](http://mayastation.typepad.com/maya-station/2009/12/maya-api-where-to-start.html) at Maya Station.
* [Maya How-To's](http://ewertb.soundlinker.com/maya.php) by Bryan Ewert.
    * A comprehensive collection of the tips for MEL & API!! (Don't miss it!)
* [Rob The Bloke](http://nccastaff.bournemouth.ac.uk/jmacey/RobTheBloke/www/) (Robert Bateman)
    * There are many examples about Maya API! I've learnt so much from Rob's web site in  my early career. It is a great resource for Maya API beginner. :)
* [Building User Interfaces](http://download.autodesk.com/us/maya/2011help/PyMel/ui.html) with PyMEL.

## Dependency-Graph & Evaluation Framework

This is the most important concept that every Maya API programmer should be familiar with. Before Maya 2015, the DG evaulation is a two-stage process: __dirty propagation & attribute computation__:

* [DevTV: Introduction to Maya Dependency Graph Programming](http://download.autodesk.com/media/adn/DevTV_Introduction_to_Maya_Dependency_Graph_Programming/DevTV%20-%20Introduction%20to%20Maya%20Dependency%20Graph%20Programming.html)
* [Introduction to Maya Dependency Graph Programming.](http://www.gdcvault.com/play/1014524/Introduction-to-Maya-Dependency-Graph) Naiqi Weng. GDC'11.
* [Dependency Graph Plug-in Basics](http://help.autodesk.com/view/MAYAUL/2018/ENU/?guid=__files_GUID_A9070270_9B5D_4511_8012_BC948149884D_htm)

In the evaluation framework before Maya 2015, as a plug-in developer, we only can do `per-node` parallelism, which is not an optimal solution to expolit all computing resources. Since Maya 2016, there is a new evaluation mode named __"Parallel"__, it outperforms the rigging evaluation by using `per-subgraph` parallelism.

* [Supercharged Animation Performance in Maya 2016.](https://www.youtube.com/watch?v=KKC7A9bbUuk) Video tutorial from Autodesk. <span class="orange">__Highly recommended!__</span>
* [Using Parallel Maya.](http://download.autodesk.com/us/company/files/2018/UsingParallelMaya.html) Technical white paper from Autodesk.
* [Locate Animation Bottlenecks In The Profiler.](https://knowledge.autodesk.com/support/maya/learn-explore/caas/CloudHelp/cloudhelp/2016/ENU/Maya/files/GUID-B1D67D2A-1283-488D-90EB-B7F16E26A118-htm.html)

# 3rd Party Plug-ins

* [HOT](http://www.nico-rehberg.de/shader.html) (Houdini Ocean Toolkit) for Maya.
* [SOuP](http://www.soup-dev.com).
* [Pose Space Deformer](http://www.djx.com.au/blog/2015/05/09/comet-posedeformer-for-maya-2016/). Originally developed by [Michael Comet](http://www.comet-cartoons.com/), now maintained by [David Johnson (djx)](http://www.djx.com.au/blog/downloads/).
* [Free scripts/plug-ins](https://www.highend3d.com/maya/scripts-plugins/c/downloads) @Highend3D