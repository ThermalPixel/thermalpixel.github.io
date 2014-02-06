---
layout: post
title:  "Using GL_KHR_debug in OpenSceneGraph"
date:   2014-02-06 09:59:38
categories: opengl osg
---

The [OpenGL debug output extension][gl-khr-debug] is very useful to find errors and performance issues. However, OpenSceneGraph still uses `glGetError` for error reporting. In this post, I will show you how to enable the `GL_KHR_debug` extension with OpenSceneGraph without the use of external libraries. So, let's get started.

Setting up the Viewer
---------------------
In order to setup an OpenGL extension, we need an active OpenGL context on the current thread. A way to do this with OpenSceneGraph is to create a subclass of `osg::GraphicsOperation` and register it as a realize operation on the viewer.

{% highlight c++ %}
osg::Viewer viewer;
viewer.setRealizeOperation(new EnableGLDebugOperation());
{% endhighlight %}

The graphics operation is simple, it only calls extension setup routine.

{% highlight c++ %}
class EnableGLDebugOperation : public osg::GraphicsOperation 
{
public:
    EnableGLDebugOperation()
        : osg::GraphicsOperation("EnableGLDebugOperation", false) {
    }
    virtual void operator ()(osg::GraphicsContext* gc) {
        OpenThreads::ScopedLock<OpenThreads::Mutex> lock(_mutex);
        int context_id = gc->getState()->getContextID();
        enableGLDebugExtension(context_id);
    }
    OpenThreads::Mutex _mutex;
};
{% endhighlight %}

Setting up the Extension
------------------------
We need to define the three entry points of the extensions as well as the OpenGL defines. I'll skip some of the ugly code here and leave the rest in the [repository][code] for the brave.

{% highlight c++ %}
typedef void (GL_APIENTRY *GLDEBUGPROC)(GLenum, GLenum, GLuint, GLenum, GLsizei,
const GLchar *, const void *);
typedef void (GL_APIENTRY *GLDebugMessageControlPROC)(GLenum, GLenum, GLenum, GLsizei,
 const GLuint *, GLboolean);
typedef void (GL_APIENTRY *GLDebugMessageCallbackPROC)(GLDEBUGPROC , const void *) ;
{% endhighlight %}

Now, we can set up the extension. OpenSceneGraph provides with `setGLExtensionFuncPtr` a function that maps the address of the extension to our function instance:
{% highlight c++ %}
void enableGLDebugExtension(int context_id)
{
    GLDebugMessageControlPROC glDebugMessageControl = nullptr;
    GLDebugMessageCallbackPROC glDebugMessageCallback = nullptr;
    //test the possible debug extensions
    if(osg::isGLExtensionSupported(context_id, "GL_KHR_debug")){
        osg::setGLExtensionFuncPtr(glDebugMessageCallback, "glDebugMessageCallback");
        osg::setGLExtensionFuncPtr(glDebugMessageControl, "glDebugMessageControl");
    }else if(osg::isGLExtensionSupported(context_id, "GL_ARB_debug_output")){
        osg::setGLExtensionFuncPtr(glDebugMessageCallback, "glDebugMessageCallbackARB");
        osg::setGLExtensionFuncPtr(glDebugMessageControl, "glDebugMessageControlARB");
    }else if(osg::isGLExtensionSupported(context_id, "GL_AMD_debug_output")){
        osg::setGLExtensionFuncPtr(glDebugMessageCallback, "glDebugMessageCallbackAMD");
        osg::setGLExtensionFuncPtr(glDebugMessageControl, "glDebugMessageControlAMD");
    }

    if(glDebugMessageCallback != nullptr && glDebugMessageControl != nullptr){
        glEnable(GL_DEBUG_OUTPUT);
        glEnable(GL_DEBUG_OUTPUT_SYNCHRONOUS);
        glDebugMessageControl(GL_DONT_CARE, GL_DONT_CARE, GL_DEBUG_SEVERITY_MEDIUM, 0, NULL,
        GL_TRUE);
        glDebugMessageCallback(debugCallback, nullptr);
    }
}
{% endhighlight %}

Finally, one just need to define the callback `debugCallback` that will accept the messages. An example implementation, which logs the messages using `osg::notify`, can be found in the [repository][code].

Some Notes
----------
In order to guarantee a debug output based on the `GL_KHR_debug` extension specification, the `CONTEXT_FLAG_DEBUG_BIT` must be set during OpenGL context creation. However, the OpenSceneGraph API currently doesn't provide access to the context creation. Therefore, this approach is depending on how the graphics vendors implemented the debug output extension. For me, the debug output in an OpenSceneGraph application works fine on a NVIDIA graphics card. However, I don't receive any callbacks on an AMD card. I assume that the debug output is disabled when the `CONTEXT_FLAG_DEBUG_BIT` is not set.

Get the code [here][code].

[gl-khr-debug]: http://www.opengl.org/registry/specs/KHR/debug.txt
[code]: https://github.com/ThermalPixel/osgdemos/tree/master