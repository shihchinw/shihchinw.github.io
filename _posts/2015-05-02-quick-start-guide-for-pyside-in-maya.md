---
layout: post 
title: Quick Start Guide for PySide in Maya
date: '2015-05-02T18:34:00.000+08:00'
categories: []
tags: [Maya, Python, PySide]
published: True
cover-img: "images/pyside/pyside_maya_workflow.png"

---

The goal of tool-making is to make users do certain works more efficiently. A functional tool with poor designed interface would not be fully utilized by users, because users aren't willing to use that tool if the interaction makes them feel frustrated. 

# Developing is a Iterative Process

Although there are a few [principles](http://bokardo.com/principles-of-user-interface-design/) about user interface design, but it doesn't mean that we can simply apply those principles and get our jobs done. In day-to-day situations, tool-development is a iterative process. As a developer, we'd better continuously communicate with our target users, and do some modifications according to the feedback, and repeat those processes back and forth until users'  workflows are truly improved. (In my personal experience, the much users like our tool, the more feedback they will give us.)

# Functionalities and User-Interface

In early development stage, we might change the interface design frequently. In order to make our life easier, we'd better avoid putting all these codes into a single Python script. We can decompose the source codes to two primary parts: one is the underlying functionalities, the other is for layout design. This loose-coupling idea make us possible to redesign the layout without amending any code for underlying functionalities. Besides, the code is easier to maintain, because we can focus on each part respectively without worrying about other interferences.

> Furthermore, when we develop a tool with more complicated functionalities and interactions, it would be better to adopt the Model-View-Controller pattern. It separates the logic into data accessing model, view presentation and the interaction between them.
> 
> More Information about MVC
> * [Model-View-Controller](http://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller) @wiki
> * [Model-View programming](http://doc.qt.io/qt-4.8/model-view-programming.html)

# Workflow

![PySide Workflow](/images/pyside/pyside_maya_workflow.png)<br>

1. Design the custom UI layout with QtDesigner
2. Compile ui file to py script
3. Combine the design and functionality
4. Invoke the UI from Script Editor
5. Update the Layout via reload()
The example files can be downloaded [here](https://drive.google.com/file/d/0BzMAAU0MqqtQVVBGU2ZyRU1EbFk/view?usp=sharing).

## Design the custom UI layout with QtDesigner

* The designer is shipped with Maya, we can find it in __$(MAYA_ROOT)\bin\designer.exe__
* We can simply design the layout via the drag-and-drop interface. (Learn [more](http://doc.qt.io/qt-4.8/designer-manual.html) about QtDesigner)

![QtDesigner](/images/pyside/qtRobotMayaDesigner.png) <br>

* We could also setup the signal/slot relationship in designer.
    * Here we make the rightDial synced with leftDial

![Singal-Slot Editor](/images/pyside/signal-slot-editor.png)

## Compile ui file to py script

{% highlight powershell %}
$(MAYA_BIN)\mayapy $(MAYA_BIN)\pyside-uic design.ui -o uiDesign.py
{% endhighlight %}

![pyside-uic](/images/pyside/pyside-uic-helps.png)

## Combine the design and functionality

Create a new file mainwin.py and

* Import `uiDesign.py` which is compiled from `design.ui` previously. _line#7_
* Inherit the mixin class (e.g. Ui_RobotWindow) and invoke `self.setupUi()` to create the layout. _line#13-16_
* Connect signal to proper slots which are responsible for each specific task respectively. _line#19_

{% highlight python linenos %}
#!/usr/bin/env python
import sys
 
from PySide.QtCore import *
from PySide.QtGui import *
from shiboken import wrapInstance
import uiDesign
reload(uiDesign)
 
import maya.cmds as cmds
import maya.OpenMayaUI as omui
 
class RobotWindow(QMainWindow, uiDesign.Ui_RobotWindow):
    def __init__(self, parent=None):
        super(RobotWindow, self).__init__(parent)
        self.setupUi(self)
 
        # Setup signal/slot connections...
        self.horizontalSlider.valueChanged.connect(self.setCurrentTime)
 
    def setCurrentTime(self, value):
        minTime = cmds.playbackOptions(q=True, min=True)
        maxTime = cmds.playbackOptions(q=True, max=True)
        time = (maxTime - minTime) * value / 100
        cmds.currentTime(time, e=True)
 
def initUi():
    mayaMainWindowPtr = omui.MQtUtil.mainWindow()
    mayaMainWindow = wrapInstance(long(mayaMainWindowPtr), QWidget)
    # Create a instance of our RobotWindow and set its parent 
    # to the main window of Maya.
    myWin = RobotWindow(mayaMainWindow)
    myWin.show()
{% endhighlight %}

Make sure to parent our widget under an existing Maya widget (such as Maya's main window). Otherwise, if the widget is un-parented, it may be destroyed by the Python interpreter's garbage collector when a reference to it is not maintained.

## Invoke the UI from Script Editor

{% highlight python linenos %}
import sys
sys.path.append(r'DIR_PATH_TO_THE_SCRIPTS')
import mainwin as MyMainWin
MyMainWin.initUi()
{% endhighlight %}

If everything works fine, we should see a window with robot face with following features:

![QtRobot](/images/pyside/qtRobotMaya.png)

* Left dial (eye) will trigger the movement of right dial
* The bottom slider (mouth) controls the current time in the range of Maya's playback slider

## Update the Layout via reload()

When we want to update the latest layout from design.ui, the first thing is to compile it to uiDesign.py via pyside-uic. Then, we've to reload the module uiDesign __AND__ mainwin to make sure all definitions are up-to-date. Why do we need to reload both modules? Let's check the [reload()](https://docs.python.org/2.7/library/functions.html?highlight=reload#reload) in docs:

> When reload(module) is executed, other references to the old objects (such as names external to the module) are __not__ rebound to refer to the new objects and must be updated in each namespace where they occur if that is desired.
>
> If a module instantiates instances of a class, reloading the module that defines the class does __not__ affect the method definitions of the instances — they continue to use the old class definition. The same is true for derived classes.

In other words, if we only reload mainwin, it would __not__ update the uiDesign <span class="orange">because the module uiDesign is imported within the mainwin.</span> In addition, if we only reload uiDesign, <span class="orange">the instance of RobotWindow is still using the __old__ definition of uiDesign.</span> That's why we need to reload them both, however, in order to save manual steps for updating, I simply put `reload(uiDesign)` in _line#8_. Thus we could get the up-to-date result every time we reload the mainwin.

Few discussions about __reload()__ @stackoverflow:

* [reloading dependent modules in Python](http://stackoverflow.com/questions/16675066/reloading-dependent-modules-in-python)
* [how to find list of modules which depend upon a specific module in python](http://stackoverflow.com/questions/1827629/how-to-find-list-of-modules-which-depend-upon-a-specific-module-in-python/)
* [reload component Y imported with 'from X import Y'?](http://stackoverflow.com/questions/1739924/python-reload-component-y-imported-with-from-x-import-y)

# References

* [PySide](http://wiki.qt.io/Pyside)
* [Rapid programming from PyQt](http://www.qtrac.eu/pyqtbook.html)
* [Qt Designer Manual](http://qt-project.org/doc/qt-4.8/designer-manual.html)
* Maya doc
    * [Using LoadUI command](http://help.autodesk.com/view/MAYAUL/2015/ENU/?guid=__files_GUID_CEC1D76B_7568_4DCA_B80B_1DE49362492C_htm)
    * [Working with PySide in Maya](http://help.autodesk.com/view/MAYAUL/2015/ENU/?guid=__files_GUID_3F96AF53_A47E_4351_A86A_396E7BFD6665_htm)
