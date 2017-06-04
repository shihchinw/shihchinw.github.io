---
categories: []
date: 2011-10-23T13:02:00Z
published: true
tags:
- Maya API
- Python
title: How to select hard edges from a mesh in Maya?
---

![feature edge](/images/maya/maya_how_to_hardedge.jpg)

A hard edge (aka crease edge or feature edge) is:

1. shared by two polygons.
2. and the dihedral angle (i.e the angle between two polygons) is above user-specified threshold.

To find these edges in Maya, the intuitive approach is to traverse all the edges by `MItMeshEdge` and examining the dot product of two adjacent faces' normals.

Suppose we have MObject(s) of target mesh and edge components (optional) selected by user, and we want to replace current selection with the hard edges from these input. We could first create a MItMeshEdge with the mesh and edge components, then for each visited edge:

1. Get ids of adjacent faces by `MItMeshEdge.getConnectedFaces`.
2. Use these ids to get face normals with `MFnMesh.getPolygonNormals`.
3. Determine if the dot product of two face normals is less than the threshold given by user.
4. Append these edges to a selection list.
5. Finally, set the selection list as an active one with `MGlobal.setActiveSelectionList`.

<pre class="prettyprint linenums lang-python">
def selectHardEdge(angleInDegree, dpItem, moComp=om.cvar.MObject_kNullObj):
    """
    \param dpItem The dag Path of given object.
    \param moComp The MObject of the components.
    """

    angle = math.radians(angleInDegree)
    cosThreshold = math.cos(angle)

    fnMesh = om.MFnMesh(dpItem)
    itEdge = om.MItMeshEdge(dpItem, moComp)

    faceIds = om.MIntArray()
    faceNormal1 = om.MVector()
    faceNormal2 = om.MVector()

    selResult = om.MSelectionList()

    while not itEdge.isDone():

        itEdge.getConnectedFaces(faceIds)
        assert(len(faceIds) == 2)

        fnMesh.getPolygonNormal(faceIds[0], faceNormal1)
        fnMesh.getPolygonNormal(faceIds[1], faceNormal2)

        if cosThreshold > faceNormal1 * faceNormal2:
            selResult.add(dpItem, itEdge.currentItem(), True)

        itEdge.next()

    om.MGlobal.setActiveSelectionList(selResult)
</pre>

<span class="red">__Note:__</span> If we use OpenMaya.MObject.kNullObj as the default value of moComp, we would get following error message:
<samp class="error"># Error: NotImplementedError: Wrong number of arguments for overloaded function ‘new_MItMeshEdge’.
Possible C/C++ prototypes are:
MItMeshEdge(MObject &,MStatus *)
MItMeshEdge(MObject &,MObject &,MStatus *)
MItMeshEdge(MDagPath const &,MObject &,MStatus *) #</samp>

It is because Maya binds these static variables with cvar in each module, thus we should use OpenMaya.__cvar__.MObject_kNullObj instead!
