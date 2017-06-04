---
title: Maya
published: true
---

# General Resources

* [Autodesk Developer Center](http://usa.autodesk.com/adsk/servlet/index?siteID=123112&id=9469002)
* [Maya Knowledge Network](http://usa.autodesk.com/adsk/servlet/index?id=16075355&siteID=123112&linkID=10809894)
* [Maya Documentation](http://knowledge.autodesk.com/support/maya/learn-explore/caas/CloudHelp/cloudhelp/ENU/123112/files/maya-documentation-html.html)
* [Maya Learning Channel](https://www.youtube.com/user/MayaHowTos) @YouTube
* [Duncan's Corner](http://area.autodesk.com/blogs/duncan)

## Forums

* [CG-Society](http://forums.cgsociety.org/forumdisplay.php?f=89)
* [Autodesk AREA](http://forums.autodesk.com/t5/programming/bd-p/area-b50)
* [Creative Crash](http://www.creativecrash.com/forums/api/topics)

# Programming

* <cite>["New To The Api? Then Read This!"](http://www.creativecrash.com/forums/api/topics/new-to-the-api-then-read-this-33)</cite> by Robert Bateman
* <cite>["Maya API: Where to start!?!??!?"](http://mayastation.typepad.com/maya-station/2009/12/maya-api-where-to-start.html)</cite> @ Maya Station
* [Maya How-To's](http://ewertb.soundlinker.com/maya.php) by Bryan Ewert
    * A comprehensive collection of the tips for MEL & API.
* [Rob The Bloke](http://nccastaff.bournemouth.ac.uk/jmacey/RobTheBloke/www/) (Robert Bateman)
    * There are many examples about Maya API! I've learnt so much from Rob's web site in  my early career. It is a great resource for Maya API beginner. :)
* [Building User Interfaces](http://download.autodesk.com/us/maya/2011help/PyMel/ui.html) with PyMEL.

## Dependency-Graph & Evaluation Framework

This is the most important concept that every Maya API programmer should be familiar with. Before Maya 2015, the DG evaulation is a two-stage process: __dirty propagation & attribute computation__:

* [DevTV: Introduction to Maya Dependency Graph Programming](http://download.autodesk.com/media/adn/DevTV_Introduction_to_Maya_Dependency_Graph_Programming/DevTV%20-%20Introduction%20to%20Maya%20Dependency%20Graph%20Programming.html)
* [Introduction to Maya Dependency Graph Programming.](http://www.gdcvault.com/play/1014524/Introduction-to-Maya-Dependency-Graph) Naiqi Weng. GDC'11.

In the evaluation framework before Maya 2015, as a plug-in developer, we only can do `per-node` parallelism, which is not an optimal solution to expolit all computing resources. Since Maya 2016, there is a new evaluation mode named __"Parallel"__, it outperforms the rigging evaluation by using `per-subgraph` parallelism.

* [Supercharged Animation Performance in Maya 2016.](https://www.youtube.com/watch?v=KKC7A9bbUuk) Video tutorial from Autodesk. <span class="orange">__Highly recommended!__</span>
* [Using Parallel Maya.](http://static-dc.autodesk.net/content/dam/autodesk/www/Company/files/Parallel_Maya.pdf) Technical white paper from Autodesk.
* [Locate Animation Bottlenecks In The Profiler.](https://knowledge.autodesk.com/support/maya/learn-explore/caas/CloudHelp/cloudhelp/2016/ENU/Maya/files/GUID-B1D67D2A-1283-488D-90EB-B7F16E26A118-htm.html)


# 3rd Party Plug-ins

* [ADN plugins](http://labs.autodesk.com/utilities/ADN_plugins)
* [HOT](http://www.nico-rehberg.de/shader.html) (Houdini Ocean Toolkit) for Maya.
* [SOuP](http://www.soup-dev.com).
* [Pose Space Deformer](http://www.djx.com.au/blog/2015/05/09/comet-posedeformer-for-maya-2016/). Originally developed by [Michael Comet](http://www.comet-cartoons.com/), now maintained by [David Johnson (djx)](http://www.djx.com.au/blog/downloads/).
