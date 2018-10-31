---
categories: []
date: 2013-06-01T10:25:00Z
published: true
tags:
- CMake
- Maya
title: Building AbcExport/AbcImport on Windows
---

[Alembic](http://code.google.com/p/alembic/) is a popular data format for geometry cache nowadays, however, there are no simple workflow to build the AbcImport/AbcExport on windows. In this post, I tried to demonstrate a building flow as simple as possible.

[Here](https://docs.google.com/file/d/0B3FnMqIdd-iLN0czVE1mTl9oWUE/edit?usp=sharing) is the pre-built binaries for Maya 2013 x64.
<!--more-->

# Prerequisites

* Boost
    1. open vc10 command prompt
    2. cd c:\boost\boost_1_xx_00
    3. bootstrap.bat
    4. b2 toolset=msvc-10.0 address-model=64 link=static threading=multi --with-program_options --with-thread
* [HDF5](http://www.hdfgroup.org/HDF5/release/obtain5.html)
* [Ilmbase](https://github.com/openexr/openexr)
* Zlib
* glut (optional)

Since I usually build maya plugin with /MD, thus all the static third-party libraries I used are built with `/MD`.

# Pre-Configuration

Now, suppose we have following folder structure:

* alembic_source_root (hg clone https://code.google.com/p/alembic/)
    * houdini
    * lib
    * maya
    * ...
    * CMakeLists.txt
    * init_cache.cmake

# Create init_cache.cmake

In order to make configuration easier, we could create a simple text file name `init_cache.cmake` as initial cmake cache:
<pre class="prettyprint lang-CMake">
SET(BOOST_ROOT C:/boost/boost_1_53_0 CACHE PATH "Boost Root")
SET(ENV{HDF5_ROOT} D:/coding/packages/HDF5 CACHE PATH "HDF5 Root")
SET(ILMBASE_ROOT D:/coding/packages/ilmbase CACHE PATH "Ilmbase Root")
SET(ZLIB_ROOT D:/coding/packages/zlib CACHE PATH "Zlib Root")
SET(GLUT_ROOT_PATH D:/coding/packages/glut-3.7 CACHE PATH "Glut Root")
SET(MAYA_ROOT C:/adsk/Maya2013 CACHE PATH "Maya Root")<br>
SET(USE_PYALEMBIC OFF CACHE BOOL "Compile Python Binding")
SET(USE_ARNOLD OFF CACHE BOOL "Use Arnold")
</pre>

Ps. Please replace the path of each 3rd-party library in your local machine.

# Modify CMakeLists.txt (optional)

If we want to link static libraries of Ilmbase, we have to do little modification of CMakeLists.txt in `line 80`:

<pre class="prettyprint lang=CMake">
#ADD_DEFINITIONS ( -DOPENEXR_DLL -DHALF_EXPORTS )
ADD_DEFINITIONS ( -DPLATFORM_BUILD_STATIC )
</pre>

# Create VC Project

1. open vc10 command prompt
2. cd /d alembic_source_root
3. mkdir vc_proj
4. cd vc_proj
5. cmake -G "Visual Studio 10 Win64" -C ..\init_cache.cmake ..

If all goes well, we could simply open the ALEMBIC.sln and compile the libraries as usual.

# Python-Binding?

Actually, I have not successfully built the python binding yet. The primary [problem](https://groups.google.com/d/topic/alembic-discussion/OMOvhg0Nv7o/discussion) I ran into is about the `PyIlmbase`.
