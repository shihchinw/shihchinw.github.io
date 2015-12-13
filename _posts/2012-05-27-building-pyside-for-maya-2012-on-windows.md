---
layout: post 
title: Building PySide for Maya 2012 on Windows
date: '2012-05-27T00:26:00.001+08:00'
categories: []
tags: [Maya, PySide, CMake]
published: True

---

One of my friend received some tools developed in PySide for Maya recently. He tried to copy the latest pre-built binaries of PySide into Maya's site-packages but got an import error. Therefore, we have to build the PySide for Maya from the ground up and this post is about the entire building process we've been through. Hope this note might be helpful for people who want to use PySide in Maya. :)

BTW, the PySide for Maya 2012 built by vc9 can be downloaded [here](https://docs.google.com/open?id=0BwFItFJF_4j3Wm1kMDVXcWIxbW8), enjoy it!

# Prerequisites

* [git](http://git-scm.com/download/win)
* [cmake](http://www.cmake.org/cmake/resources/software.html)
* [jom](http://qt-project.org/wiki/jom)
* visual c++

# Build autodesk modified Qt

1. download [QT4.7.1 modified for Maya 2012 Service Pack 2](http://images.autodesk.com/adsk/files/qt-4.7.1-modified_for_maya_2012_service_pack_2.zip)
2. extract files into C:\Qt\qt-adsk-4.7.1
3. cd C:\qt\qt-adsk-4.7.1
4. configure -platform win32-msvc2008 -release -no-qt3support
5. jom -j N (_N=how many cores do you want to use for compiling_)
6. set environment variable __QTDIR=C:\qt-adsk-4.7.1__
7. append __%QTDIR%\bin__ to your PATH environment variable

OK, let's go for the packages of PySide. Assume we have following directory layout:

+ D:\pyside_for_maya
  + src\
  + deploy\

# Build Shiboken

1. cd D:\pyside_for_maya\src
1. git clone git://gitorious.org/pyside/shiboken.git
1. cd shiboken
1. apply the [patch](https://github.com/PySide/Shiboken/issues/65) [optional]
1. mkdir build
1. cd build
1. create a text file named `init_cache.txt` for cmake pre-load cache:
    <pre class="prettyprint lang=Cmake">
SET(CMAKE_BUILD_TYPE "Release" CACHE STRING "");
SET(CMAKE_INSTALL_PREFIX "d:/pyside_for_maya/deploy/shiboken" CACHE PATH "");
SET(BUILD_TESTS OFF CACHE BOOL ""); SET(MAYA_ROOT "path-to-maya" CACHE PATH "");
SET(PYTHON_EXECUTABLE "${MAYA_ROOT}/bin/mayapy.exe" CACHE FILEPATH "");
SET(PYTHON_INCLUDE_DIR "${MAYA_ROOT}/include/python2.6" CACHE PATH ""); 
SET(PYTHON_LIBRARY "${MAYA_ROOT}/lib/python26.lib" CACHE FILEPATH "");</pre>
1. cmake -G "NMake Makefiles JOM" -C "init_cache.txt" ..
2. jom -j N 
ps. if you have the following error:
    <samp class="error">[ 98%] Running generator for 'shiboken'...
jom: D:\pyside_for_maya\src\shiboken\build\shibokenmodule\CMakeFiles\shibokenmod ule.dir\build.make [shibokenmodule\shiboken\shiboken_module_wrapper.cpp] Error 2
jom: D:\pyside_for_maya\src\shiboken\build\CMakeFiles\Makefile2 [shibokenmodule\CMakeFiles\shibokenmodule.dir\all] Error 2
jom: D:\pyside_for_maya\src\shiboken\build\Makefile [all] Error 2</samp> please copy QtCore4.dll from C:\Qt\qt-adsk-4.7.1\bin to
D:\pyside_for_maya\src\shiboken\build\generator and execute jom again
3. jom install

# Build PySide

1. copy all dlls from C:\Qt\qt-adsk-4.7.1\bin to D:\pyside_for_maya\deploy\shiboken\bin [optional]
2. cd d:\pyside_for_maya\src
3. git clone git://gitorious.org/pyside/pyside.git
4. cd pyside
5. mkdir build
6. create `init_cache.txt`:
    <pre class="prettyprint lang=CMake">
SET(CMAKE_BUILD_TYPE Release CACHE STRING "")
SET(CMAKE_INSTALL_PREFIX D:/pyside_for_maya/deploy/pyside CACHE PATH "")
SET(BUILD_TESTS OFF CACHE BOOL "")
SET(Shiboken_DIR D:/pyside_for_maya/deploy/shiboken/lib/cmake/Shiboken-1.1.2 CACHE PATH "")</pre>
7. make -G "NMake Makefiles JOM" -C "init_cache.txt" ..
8. jom -j N
9. jom install

Troubleshooting for compile error in step 8:
<samp class="error">[ 47%] Building CXX object PySide/QtGui/CMakeFiles/QtGui.dir/PySide/QtGui/qmessagebox_wrapper.cpp.obj D:\pyside_for_maya\src\pyside\build\PySide\QtGui\PySide\QtGui\qmenu_wrapper.cpp(3720) : error C2589: 'constant' : illegal token on right side of '::'
D:\pyside_for_maya\src\pyside\build\PySide\QtGui\PySide\QtGui\qmenu_wrapper.cpp(3720) : error C2059: syntax error : '::'
jom: D:\pyside_for_maya\src\pyside\build\PySide\QtGui\CMakeFiles\QtGui.dir\build.make [PySide\QtGui\CMakeFiles\QtGui.dir\PySide\QtGui\qmenu_wrapper.cpp.obj] Error 2 qmessagebox_wrapper.cpp
jom: D:\pyside_for_maya\src\pyside\build\CMakeFiles\Makefile2 [PySide\QtGui\CMakeFiles\QtGui.dir\all] Error 2
jom: D:\pyside_for_maya\src\pyside\build\Makefile [all] Error 2
</samp>just fix the `line 3720` in
D:\pyside_for_maya\src\pyside\build\PySide\QtGui\PySide\QtGui\qmenu_wrapper.cpp to <span style="background-color: yellow;">::QPoint* cppArg0 = <del>QPoint::</del>NULL;</span>

# Deploy PySide packages

Yeah, we are almost done! The last thing to do is to copy files from our deploy folder to path-to-maya\Python\lib\site-packages: 

1. shiboken\lib\site-packages\shiboken.pyd
2. pyside\lib\site-packages\PySide

Besides, we also need to copy two dlls to path-to-maya\Python\lib\site\packages\\__PySide__

1. pyside\bin\pyside-python2.6.dll
2. shiboken\bin\shiboken-python2.6.dll

# Simple "Hello World!"

1. extract [hello_pyside_for_maya.rar](https://docs.google.com/open?id=0BwFItFJF_4j3aFdidG9kRHdYM2c) to D:\
2. execute the following commands in Maya's script editor
    <pre class="prettyprint">
    import sys
    sys.path.append(r'D:\hello_pyside_for_maya')

    import mainwin as mw
    frame = mw.MainWindow()
    frame.show()</pre>
3. If it executes successfully, you would see this simple window:
    <img src="/images/maya/hello_pyside_for_maya.PNG" width="60%">

Congrats and enjoy the PySide in Maya. ;)
