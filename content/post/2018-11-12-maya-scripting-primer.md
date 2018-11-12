---
title: "Maya Scripting Primer"
date: 2018-11-12T20:59:06+08:00
published: true
tags:
- Maya API
---

For many of my artist colleagues who want to learn scripting in Maya, one of their motivations is mostly to automate repetitive or time-consuming manual tasks. Since Maya can show all command history of our user interactions in Script Editor. Therefore, if we know how to perform target instructions manually, we could write a script of all corresponding commands. But finding the equivalent commands is not straightforward in some situations. Besides looking up documentation, we need other strategies.
<!--more-->

# MEL or Python?

![Maya programming interfaces](/images/maya/maya_api.png)

There are two interfaces can be used to manipulate Maya internal state: *Maya Commands* and *Maya API*. On the top of that, Maya provides three programming language options: [MEL (Maya Embedded Language)](http://help.autodesk.com/view/MAYAUL/2018/ENU/?guid=GUID-E151A15C-BA1D-4E60-8DB6-9D92C6202170), [Python](https://www.python.org/) and [C++](https://isocpp.org/tour).

MEL is Maya's native scripting language. Everything we do via Maya GUI can be automated by using MEL commands. By contrast, Python is a general purpose programming language. Python scripts are executed by a Python interpreter. The reason we can execute a Python script in Maya is because there is an embedded Python interpreter in Maya, and this specialized interpreter is capable of interpreting Maya Commands (<code>maya.cmds</code>) and Maya API (<code>maya.OpenMaya</code> or <code>maya.api.OpenMaya</code>).

That's why we can't execute Maya Python script within Houdini's Python interpreter and vice versa. (Since they don't know the dialects used by one another, but they can talk with their common languages though.) Moreover, there is another high-level library called [PyMEL](http://help.autodesk.com/view/MAYAUL/2018/ENU/?guid=__PyMel_index_html) which is built upon Maya Python binding. Its interfaces are object-oriented and most intuitive to use among all these choices. 

Since <span class="blue">Maya shows command history in Script Editor with MEL syntax</span>, thus we have to learn how to read MEL expressions at least. Fortunately, if you are already familiar with any other scripting languages, MEL is quite handy to pick up.

In my personal workflow, I use MEL for interactive command testing and use Python for scripting. PyMEL is pretty intuitive to use but at the cost of API abstraction (e.g. wrappers). I am used to using PyMEL at first, because it has clearer syntax with high readability. If I ran into performance issues (e.g. the lag caused by creating several thousands of nodes and making massive connections), I would rewrite those code with regular Maya commands or Maya Python API. As for tasks with intensive computation, I always develop plug-ins with Maya C++ API for better execution performance.

## Recommend Readings for Beginners

A few years ago, I have taught several artist friends of mine to learn programming from the ground up. The most important thing we have learned is to <span class="blue">learn how to **play** (yeah, coding is fun!) the command first.</span> For those people without any programming experience, here are some video lectures and books that might be a good starting point:

* [Beginning Python for Maya](http://zurbrigg.com/tutorials/beginning-python-for-maya), by Chris Zurbrigg.
* [Maya Python for Games and Film: A Complete Reference for Maya Python and the Maya Python API](https://www.amazon.com/Maya-Python-Games-Film-Reference/dp/0123785782/ref=sr_1_1), by Adam Mechtley and Ryan Trowbridge.
* [Practical Maya Programming with Python](https://www.amazon.com/Practical-Programming-Python-Robert-Galanakis/dp/1849694729/ref=sr_1_2), by Robert Galanakis.

If you want to take a different learning path, here is my suggested one:

1. Learn basic [MEL](http://help.autodesk.com/view/MAYAUL/2018/ENU/?guid=GUID-E151A15C-BA1D-4E60-8DB6-9D92C6202170) programming (from sec. '*MEL Overview*' to '*Creating interfaces*').
2. Learn Python itself (not in Maya); either read [Python tutorial](https://docs.python.org/2/tutorial/index.html) Sec. 1 to Sec. 8 or watch [Python tutorial for beginners](https://www.youtube.com/playlist?list=PL-osiE80TeTskrapNbzXhwoFUiLCjGgY7). Make yourself familiar with the concepts of:
    * data type, variable (mutable or immutable), operators, expression
    * data structures: tuple, list, dictionary, set
    * flow control: if, else, while, for, continue, break
    * functions, arguments and the scope of variables
    * modules
    * errors and exceptions
3. Learn [Maya Python commands](http://help.autodesk.com/view/MAYAUL/2018/ENU/?guid=__CommandsPython_index_html) and translate your MEL script done in step 1 by using corresponding functions in module <code>maya.cmds</code>.
4. Learn the concepts of *class* and *data model* in Python
    * watch [Python OOP Tutorial](https://www.youtube.com/playlist?list=PL-osiE80TeTsqhIuOqKhwlXsIBIdSeYtc) or
    * read [Python tutorial](https://docs.python.org/2/tutorial/index.html) Sec. 9
5. Learn [PyMEL](http://help.autodesk.com/view/MAYAUL/2018/ENU/?guid=__PyMel_index_html) and refactor the Python script written in step 3.

As for studying of Python, I personally suggest learn Python outside the Maya first. Since Python is a general purpose programming language, it can do many things outside 3D content creation software. Additionally, Python has also been integrated into many software like Maya, Houdini, Nuke, etc. Starting from the core Python language has *the benefit to clearly see the boundaries between native language features and the extension modules* provided by other embedding software like Maya or Houdini.

<!-- Programming is a practices skill, there is no shortcut. The more practice, -->

# Essential Concepts of Computation in Maya

Maya is a **node-based** system, all computations are composed together as a Dependency Graph (DG). <span class="blue">Any action in Maya can be seen as a process of *manipulating computation graph*.</span> In other words, it's all about creating/deleting nodes and connecting/disconnecting plugs.

The *'history'* of a node means its upstream nodes. When we *delete history* of a certain node, all its upstream nodes are deleted and the computation results are cached in [data blocks](http://help.autodesk.com/view/MAYAUL/2018/ENU/?guid=__files_Dependency_graph_plugins_Data_blocks_htm) of that node. To see a computation graph, we could open Node Editor:

![Menu of Node Editor](/images/maya/maya_menu_node_editor.png)

For instance, the figure below shows a graph that produces two sphere meshes, and one of them is triangularized. The history of polySurfaceShape1 is polySphere1, while the ones of pSphereShape1 contains polyTriangulate1 and polySphere1. (polySphere1 is a geometry generator which generates mesh data according to radius and subdivision parameters.)

![Poly sphere triangularization](/images/maya/maya_dg_poly_sphere_triangularization.png)

Generally speaking, if we want to automate certain manual routines in Maya, our goal is trying to write a script to setup such computation graph on demand.

# Convert Manual Instructions to Script

Suppose we want to write a script to [create and assign a lambert shader to a shape](https://knowledge.autodesk.com/support/maya/downloads/caas/CloudHelp/cloudhelp/2016/ENU/Maya/files/GUID-F9B3EE68-94F8-4C25-88C6-712F9C9D2B50-htm.html). After searching keyword '*shading*' or '*hyperShade*' in [documentation](http://help.autodesk.com/view/MAYAUL/2018/ENU/?guid=__CommandsPython_index_html), our first attempt might look like this along with <code>cmds.shadingNode</code> and <code>cmds.hyperShade</code>:
<pre class="prettyprint linenums lang-python">
import maya.cmds as cmds
xform, shape = cmds.polySphere()
shader = cmds.shadingNode('lambert', asShader=True)
cmds.select(shape)
cmds.hyperShade(shader, assign=True)
</pre>

and the results are errors about missing shader, eh?
<samp class="error">\# Error: No shader is selected to be assigned #
\# Error: No renderable object is selected for assignment #</samp>

> It is because Maya uses <code>shadingEngine</code> node to record the information about shader assignment, thus we can't just assign <code>shader</code> node to selected object directly. Here are the [details]({{< ref "2015-04-21-assign-custom-shapes-components-to-sets-in-maya.md" >}}) about shader assignment of custom shape.

Actually, it indeed created a lambert shader successfully. To clarify its root cause, we dig into the command history (NOT the node's history in Node Editor) of our manual instructions to find out the differences from our script. First, we have to enable <code>Echo All Commands</code> of Script Editor,
![Echo all commands](/images/maya/maya_menu_echo_all_commands.png)
and clear command history in Script Editor before starting our manual instructions.
![Clear command history](/images/maya/maya_toolbox_clear_history.png)

Execute the manual instruction which creates a Lambert shader in Hypershade, and search for the corresponding commands at very beginning of Script Editor:
![Command](/images/maya/maya_cmd_createRenderNodeCB.png)

Now we know the equivalent MEL expression is <code>createRenderNodeCB -asShader "surfaceShader" lambert "";</code>, but there is no command named createRenderNodeCB in documentation. ***What is*** createRenderNodeCB and how do we use it?

## The Usage of MEL Command *'WhatIs'*

To find the definition of certain function, we have to use the MEL command <code>whatIs</code>:
<samp>whatIs createRenderNodeCB;
// Result: Mel procedure found in: C:/adsk/Maya2017/scripts/others/createRenderNode.mel</samp>
It tells us <code>createRenderNodeCB</code> is a MEL procedure defined in createRenderNode.mel. Thus we could just open that MEL script, search function by name and continuously trace into the source code:

<pre class="prettyprint linenums:79 lang-mel">
global proc string createRenderNodeCB ( string $as, string $flag,
                                 string $type, string $postCommand )
{
    int $projection = (`optionVar -query create2dTextureType` == "projection");
    int $stencil = (`optionVar -query create2dTextureType` == "stencil");
    int $placement = `optionVar -query createTexturesWithPlacement`;
    int $shadingGroup = `optionVar -query createMaterialsWithShadingGroup`;
    int $createAndDrop = 0;
    string $editor = "";

    return renderCreateNode(
        $as,
        $flag,
        $type,
        $postCommand,
        $projection,
        $stencil,
        $placement,
        $shadingGroup,
        $createAndDrop,
        $editor);
}
</pre>

It's more than that, <code>whatIs</code> can also be used to query *command* or *variable*:
<samp>whatIs "sphere";
// Result: Command //<br>
whatIs "initToolBox";
// Result: Mel procedure found in: /u/mayauser/maya/scripts/initToolBox.mel //<br>
whatIs "kaosFieldTest";
// Result: Script found in: /u/mayauser/maya/scripts/kaosFieldTest.mel //<br>
whatIs "fdsafda";
// Result: Unknown //<br>
int $abc[42];
whatIs "$abc";
// Result: int[] variable //<br>
$s=\`pwd\`;
whatIs "$s";
// Result: string variable //
</samp>

## Reveal Details of Runtime Command

Now here is a quick quiz, please find out the corresponding command for '*new scene*'.

Based on what we have learned so far, we can easily get the answer is <code>NewScene</code>. But after applying <code>whatIs</code> to it, we get a new type of command named <code>Run Time Command</code>. Conceptually, a [runtime command](http://help.autodesk.com/view/MAYAUL/2018/ENU/?guid=GUID-2B2BE446-D622-4274-80FC-2A5EF7D87C5C) is a wrapper of other commands or scripts. To reveal its actual commands underneath, we need to use <code>runtimeCommand</code> with flag -c:
<samp>whatIs NewScene;
// Result: Run Time Command //<br>
runTimeCommand -q -c NewScene;
// Result: performNewScene 0; //<br>
whatIs performNewScene;
// Result: Mel procedure found in: C:/adsk/Maya2017/scripts/startup/performNewScene.mel</samp>

By using two MEL commands <code>whatIs</code> and <code>runtimeCommand</code>, we have figured out the equivalent function for 'new scene' is performNewScene. 

> To wrap it up, the entire workflow is:
> 
> 1. Enable <code>Echo All Commands</code> and clear command history of Script Editor.
> 2. Execute a target instruction manually.
> 3. Search the relevant command at the beginning of Script Editor.
> 4. Query with <code>whatIs</code>, if the result is
>     1. command -> look it up in [documentation](http://help.autodesk.com/view/MAYAUL/2018/ENU/?guid=__Commands_index_html)
>     2. MEL procedure -> trace the source script
>     3. runtime command -> query with <code>runtimeCommand -c</code>

<!-- # Remote Python Debugging with VS Code

1. Download [ptvsd.4.x.zip](https://pypi.org/project/ptvsd/#files) and extract the folder name '*ptvsd*' (without version number) to %USERPROFILE%/documents/maya/scripts
2. Open launch.json in VS code: press <code>Ctrl + Shift + P</code> -> <code>Debug: Open launch.json</code>
3. Add *pathMappings* to launch config [seems optional]:
    [figure]
4. Enable attachment in Maya. Execute following commands in script editor:
    <pre class="prettyprint lang-python">
    import ptvsd
    ptvsd.enable_attach(('localhost', 3000), redirect_output=True)
    </pre>
5. Back to VS Code, press <code>F5</code> to attach debugger to Maya -->

# Conclusion

Writing scripts to automate repetitive and time-consuming manual tasks can greatly improve our productivity at work. To build an efficient and robust pipeline, we have to reduce those kinds of manual tasks as much as possible. Now we know how to convert any manual instructions into scripts in Maya. Let's start scripting and enjoy automation!

# References

1. Bryan Ewert. 2006. [How do you find out all these things about MEL? I can't find some things in the documentation!](http://ewertb.soundlinker.com/mel/mel.001.php)
2. Kristine Middlemiss. 2009. [Maya API: Where to start!?!??!?](https://mayastation.typepad.com/maya-station/2009/12/maya-api-where-to-start.html)
3. [Maya programming resources.]({{< ref "maya.md" >}})

<!-- 1. [Remote Debugging with ptvsd](https://donjayamanne.iimigithub.io/pythonVSCodeDocs/docs/debugging_remote-debugging/)
2. [VSCodeからMaya上をリモートデバッグする](https://qiita.com/takumi_akashiro/items/5e18dd96b7af942cefbc) -->