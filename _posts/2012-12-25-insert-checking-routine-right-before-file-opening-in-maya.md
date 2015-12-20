---
layout: post
title: Insert checking routine right before file opening in Maya
date: '2012-12-25T22:58:00.000+08:00'
categories: []
tags: [MayaAPI, Python]
published: True

---

ScriptJob is a very convenient approach for us to register a routine for certain event. However, when we want to inject a check routine for an opened file and to be capable of canceling the open operation of the invalid file. We will find that ScriptJob only contains <span class="orange">_NewSceneOpened, PostSceneRead and SceneOpened, ..._</span>, and none of these events can abort the current operation within our registered routine. Fortunately, the `MSceneMessage` from API could help us achieve our purpose:

{% highlight python linenos %}
from maya.OpenMaya import MSceneMessage, MScriptUtil

def isValidFile(fileObj):
    print "Check Opened File %s" % fileObj.rawName()
    return True

def foo(retCodePtr, fileObj, clientData):
    # The type of retCodePtr is bool*
    # We need to use MScriptUtil as a bridge
    MScriptUtil.setBool(retCodePtr, isValidFile(fileObj))

cbId = MSceneMessage.addCheckFileCallback(MSceneMessage.kBeforeOpenCheck, foo)

# Remove the callback binding
MSceneMessage.removeCallback(cbId)
{% endhighlight %}

Here we register a callback function '_foo_' for the event `MSceneMessage.kBeforeOpenCheck`. After the registration, the foo would be invoked right before the file opening and the value pointed by retCodePtr is used to indicate whether to open that file afterwards.

Although we could directly use the function foo to perform the validation process, we should notice that if we redefine/update the foo without rebinding, the `kBeforeOpenCheck` event is still bound to the previous function object. Therefore, during the prototyping stage, the preferred way is to use another function (ex. isValidFile in our example) to do the actual checking routine. It could make us easier to modify the detail of checking process without resetting the callback registry each time.
