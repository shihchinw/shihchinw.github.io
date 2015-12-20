---
layout: post
title: Assign Custom Shapes' Components to Sets in Maya
date: '2015-04-21T23:01:00.001+08:00'
categories: []
tags: [MayaAPI]
published: True

---

Recently, I tried to extend the functionalities of GPU cache in Maya with dag item selection and shader assignment. In order to achieve the seamless integration, the idea is to use face component to represent each dag item within the Alembic cache.  Therefore, users can select the dag item from the cache in component mode and assign those components to a shader.

![Custom Shape Components](/images/maya/maya_api_custom_shape_comp_to_sets.jpg)

This idea is pretty simple, but the implementation is not as easy as it sounds. It took me few days to figure out all the relevant interactions among the classes in Maya API. This post is about the minimum implementation steps to support component assignment and restoration of a custom shape. In order to achieve our goal, there are two main tasks needed to be done: one is __selection routine__ the other is __dummy geometry data__.  Before diving into the implementation details, let's look at the big picture first.

# Checklist: The Minimum Requirements

* A custom geometry data inherited from __MPxGeometryData__
    * Only needs to implement the pure virtual functions derived from MPxGeometryData
* Inherit __MPxSurfaceShape/MPxComponentShape__ and __MPxSurfaceUI__ and override the following functions
    * <span class="blue">___for regular set assignments___</span>
        * geometryData()
        * matchComponent()
        * select()
    * <span class="blue">___for shader (render set) assignments___</span>
        * renderGroupComponentType()
        * createFullRenderGroup()

# Selection and Draw

The select procedural is implemented in __MPxSurfaceUI::select()__. In select(), we have to determine the selection mode and create a component object with one of these type:  

* Vertex: MFn::kMeshVertComponent, kSubVertexComponent, kCurveCVComponent
* Edge: MFn::kMeshVertComponent, etc
* Face: MFn::kMeshPolygonComponent, etc

I found that using component type MFn::kSetGroupComponent would <span class="red">__NOT__</span> trigger __MPxSurface::componentToPlugs()__, and this would make the 
__MPxSurfaceShape::activeComponents()__ always return null MObject. It means that we can't retrieve the activeComponents() from MPxSurfaceShape in MPxSurfaceShapeUI::draw() to render the corresponding components with desire style (ex. wireframe). 

## Few More Experiments

* At first, I tried to maintain the activeComponents myself, but I found it's almost impossible to reproduce the selection behavior with modifier (i.e. ctrl or shift key).
* The other interesting thing I found is that, when we do selection in the component mode, the MPxSurfaceUI::select() would be invoked when the mouse cursor is within the shape's bounding box __regardless we press the mouse button or not!__
* The select() is also used for the Maya's hidden command `dagObjectHit`. When we right click the mouse at our custom shape, it should invoke the select() for object hit query. If it doesn't, we should double check the selection mask to make it works, otherwise our custom `dagMenuProc` would not work as expected.

If everything works properly at this stage, we should be able to select the component from mouse event or from a command like `select -r myShape.f[0]`.

> Since I use face component to represent the dag-id within the Alembic cache, there is no actual data need to be stored in the plug. Therefore, I simply use the default implementation of componentToPlugs() without any overriding in my custom shape class. This configuration works for me so far.

# Support Per-Face Shader Assignment

In order to support faces assignment for a custom shape, we need two extra function overrides:

* renderGroupComponentType()
    * should always return one of these types:
        * MFn::kMeshPolygonComponent
        * MFn::kSubdivFaceComponent
        * MFn::kSurfaceFaceComponent
* createFullRenderGroup()
    * create a component object whose type is <span class="orange">__identical__</span> to renderGroupComponentType()

Now we can assign face components of our custom shape to a shader in the same fashion as Maya does for its native geometry types. However, although we can see the component information stored in the shape's attribute in Maya ASCII file, but the assignments can't be restored from the file yet!

![Components in ma](/images/maya/maya_api_componts_in_ma.jpg)

# Where Is Component Info Stored?

The place where component data is stored depends on __whether there is a history or not.__ When there is a history of the shape, the component is stored in the `groupParts` node. `groupParts` is used for appending the component data to a group in geometry data whose id comes the from a `groupId` node.

After we delete the history of the shape, the component data would be transferred to the shape node, but the `groupId` nodes still keep their connections. From the devkit/apiMesh sample, we found that it can restore the component data __ONLY if its history presents__. Now, our question is, how do we transfer the component data to our custom shape?

# Restore the Components in Sets without History

Here is the most subtle part, it took me two days to hack this out. We don't need to maintain the component data in our custom shape ourselves. All these things are done by Maya automatically if we provide it the appropriate geometry data via the __MPxSurfaceShape::geometryData().__ If we return a null Mobject, we won't be able to restore the shapes' components in sets when we reopen the file, even if the component data are correctly saved in the `myShape.objectGroups[*].objectGrpCompList`. It seems that Maya always query the component data from geometryData(). Therefore, the last thing we need to do is to override the __geometryData()__ properly, take apiMesh for example:

{% highlight cpp linenos %}
MObject apiMesh::geometryData() const
//
// Description
//
//    Returns the data object for the surface. This gets
//    called internally for grouping (set) information.
//
{
    apiMesh* nonConstThis = (apiMesh*)this;
    MDataBlock datablock = nonConstThis->forceCache();
    MObject geomAttr = hasHistory() ? inputSurface : outputSurface;
    MDataHandle handle = datablock.inputValue(geomAttr);
    return handle.data();
}
{% endhighlight %}

Finally, we have a custom shape supporting face assignment to a shader as Maya does for its geometry shapes. Writing a custom shape is never an easy task, we need to override many derived virtual functions for seamless integration. Hope this post would be helpful for anyone who wants to implement their own custom shape. Please feel free to drop me a note if there is any missing information. ;)

# References

* [Maya Developer Help - Shapes](http://help.autodesk.com/view/MAYAUL/2015/ENU/?guid=__files_Shapes_htm)
* [Custom Shapes in the Maya API](http://around-the-corner.typepad.com/adn/2012/09/custom-shapes-in-the-maya-api-1.html) [[plug-ins examples](http://around-the-corner.typepad.com/files/customShapeSamples.zip)] @Around the Corner
    * Step by step examples about custom shape. Extremely helpful!
* [Maya’s Deformer Architecture](http://around-the-corner.typepad.com/adn/2012/09/maya-deformer-architecture.html) @Around the Corner
    * Clear explanation for the concepts of groupParts and groupId
