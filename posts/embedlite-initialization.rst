.. title: EmbedLite Initialization
.. slug: embedlite-initialization
.. date: 2014/03/02 15:08:04
.. tags: gecko,firefox,mozilla,embedlite
.. link: 
.. description: 
.. type: text

I guess some people may find useful the couple of words below on how
EmbedLite embedding is initialized. I've written them mostly because I'm trying
to wrap my head around the topic myself. So please let me know if you see non-sense.


Initialization procedure
========================

First of all a toolkit specific embedding (e.g. qtmozembed) must pre-configure
embedlite with the function `LoadEmbedLite()`. Then we should instantiate
`EmbedLiteApp` class. This is done with the function `XRE_GetEmbedLite()`.
Only one instance of `EmbedLiteApp` class can be created. Let's call this
singleton "embedLiteApp".

Next step is to set up callbacks to the embedding into "embedLiteApp". Currently
the callbacks should be implemented as a class inheriting to
`EmbedLiteAppListener`. The callbacks are called by "embedLiteApp" in order
either to notify our embedding about application-wide events or to configure
native threads. Only one instance of this callback collection makes sense.

Then the embedding (qtmozembed) should register all needed manifests of XPCOM
components with `EmbedLiteApp::AddComponentManifest()`.

At this stage a separate thread with embedlite can be started with either
`EmbedLiteApp::StartWithCustomPump()` or `EmbedLiteApp::Start()` methods.
The former method does an asynchronous call and returns immediately, the latter
one returns only after embedlite has stopped. After that the web engine starts
its initialization procedures.

Internally "embedLiteApp" schedules a call to its `EmbedLiteApp::StartChild()`
which is supposed to create a thread for embedlite either itself or with help
of a toolkit specific embedding (through the
`EmbedLiteAppListener::ExecuteChildThread()` callback) and inside the thread
it calls `EmbedLiteApp::StartChildThread()`. This function
loads manifests of XPCOM components, loads libxul.so, initializes the gecko
webengine, creates a new message loop for the thread and schedules creation of
"App Thread" actors or communication end points in other words. One end point
(an instance of `EmbeLiteAppThreadParent`) is a parent actor used to deliver
messages from the just created thread to the parent thread where "embedLiteApp"
lives. The other end point (an instance of `EmbedLiteAppThreadChild`) is a
child actor used to communicate with objects living in the child thread. At the
moment of creation the `EmbedLiteAppThreadChild` instance opens a communication
channel to `EmbedLiteAppThreadParent`. The communication protocol is defined
in the file `PEmbedLiteApp.ipdl`. Direct method calls from one thread to another
must be avoided since the objects can be placed into different processes actually,
not threads.

Basically the instance of `EmbedLiteApp` represents a chrome (or main UI) thread.
It communicates with a toolkit specific embedding through installed callbacks
(see `EmbedLiteAppListner`) and with Gecko webengine through the actor
`EmbedLiteAppThreadParent`.

Also the initialization procedures includes creation of `EmbedLiteAppService`
implementing the interface `nsIEmbedAppService`. This service keeps track of
created web views and is used by XPCOM components to communicate with the veiws.

After the implementation of `nsIEmbedAppService` is up and running the web
engine is considered to be fully functional. This event gets propagated to
the main UI thread (the parent actor receives the `async Initialized()`
message). Also the "embedliteInitialized" message is broadcasted with
`nsIObserverService`. From now on we can create actual web views.

WebView creation
================

For every native widget representing a web view there should exist a corresponding
instance of `EmbedLiteView` class. This class exposes public API for web views
to the toolkit specific embedding (i.e. qtmozembed). Just like in the case of
`EmbedLiteApp` the embedding is supposed to register its callbacks to the
instance of `EmbedLiteView`. The callbacks are organized as virtual methods
of a class inheriting to `EmbedLiteViewListener`. A native widget is supposed
to instantiate the class and to register it with `EmbedLiteView::SetListener()`.

The instance of `EmbedLiteView` can be created by a native widget with the
method `EmbedLiteApp::CreateView()`. Internally in the method "embedliteApp"
generates an unique ID, then instantiates `EmbedLiteView` identified by the ID
and puts it into a local hash of views. After that it sends a message
`async CreateView(viewId, parentViewId)` to the gecko thread/process and returns
the just created `EmbedLiteView` instance to the native widget. The native
widget installs its callbacks and that's it. At this moment the `EmbedLiteView`
instance still cannot be used to communicate with the actual web view because it
doesn't exist yet.

When the gecko thread receives the message `async CreateView()` (via
`EmbedLiteAppThreadChild::RecvCreateView()`) it creates a pair of subprotocol
actors `EmbedLiteViewThreadParent` and `EmbedLiteViewThreadChild`. The former
lives in the same thread as `EmbedLiteAppThreadParent` does (the main UI
thread). And the latter lives in the gecko thread together with
`EmbedLiteAppThreadChild`. The parent end point for the view serves as
a communication channel to the corresponding web view which gets actually
created by the child end point. The act of web view creation happens in the
method `EmbedLiteViewThreadChild::InitGeckoWindow()`. Instances of
`EmbedLiteViewThreadChild` keep handles to the created "browser windows".
When a new "browser window" is created and properly initialized the child
end point sends a `async Initialized()` message to the corresponding parent.
The parent end point directly calls the callback `ViewInitialized()` registered
by the toolkit specific embedding. Now the native widget is notified that
its web view is fully functional.

Web view initialization
=======================

So, what actually happens inside `EmbedLIteViewThreadChild::InitGeckoWindow()`?
First of all we create an object representing web browser, that is the object must
implemenent the interface `nsIWebBrowser`. The reference to this object is kept
in the private member `EmbedLiteViewThreadChild::mWebBrowser`.

Then we create an interface instance for the `nsIBaseWindow` interface out of the
web browser object.

.. note::
   Remember that interface instances of different types can refer to the same physical
   object implementing more than one interfaces.

Also we create a fake browser widget `EmbedLitePuppetWidget` inheriting to
`PuppetWidget` and implementing the interface `nsIWidget`. This is how this
abstraction is described in Gecko code::

   This "puppet widget" isn't really a platform widget.  It's intended
   to be used in widgetless rendering contexts, such as sandboxed
   content processes.  If any "real" widgetry is needed, the request
   is forwarded to and/or data received from elsewhere.

Then we initialize the base window with the widget::

   rv = baseWindow->InitWindow(0, mWidget, 0, 0, mViewSize.width, mViewSize.height);
   if (NS_FAILED(rv)) {
     return;
   }

The important part is that we initialize the window which hasn't been created
yet, because as said in the documentation for the property
`nsIWebBrowser.containerWindow`::

   The embedder must create one chrome object for each browser object that is
   instantiated. The embedder must associate the two by setting this property
   to point to the chrome object before creating the browser window via the
   browser's nsIBaseWindow interface.

After that we

  1. create and initialize a chrome object (`nsIWebBrowserChrome`),
  2. associate it with the web browser object,
  3. finally create the base window (`nsIBaseWindow`),
  4. request an interface object for `nsIDOMWindow` from the web browser object,
  5. register the view ID in `nsIEmbedAppService`,
  6. broadcast the event "embedliteviewcreated" on behalf of the `nsIDOMWindow`
     instance to interested observers,
  7. instantiate an interface object for `nsIWebNavigation` out of the base
     window,
  8. associate the web browser with the chrome object (so now they know each
     other),
  9. mark the base window visible,
  10. create a `TabChildHelper` instance,
  11. send `async Initialized()` message to the main UI thread.

`TabChildHelper` is a private object handling various tasks for `EmbedLiteViewThreadChild`
such as viewport calculations and handling scroll events originating from
content.

.. note::
   The current goal is to make TabChildHelper share functionality with upstream dom/ipc/TabChild
   class, in order to reduce maintenance burden.

On compositing
==============

The code of `EmbedLitePuppetWidget` basically is a copy-paste from mozilla's
`PuppetWidget` class. Would be nice to refactor it to avoid code duplication.
Mainly the code differs in how compositor objects are created. In fact the base
class `PuppetWidget` doesn't create any compositor objects since it's a
responsibility of a native widget, but embedlite does create a compositor
inside this fake widget by calling the static method
`gfxPlatform::GetPlatform()` (see `EmbedLitePuppetWidget::CreateCompositor`).
This method

  1. initializes a font rasterizer library,
  2. initializes Qt's graphic platform (looks like there is no much Qt specific
     stuff left there),
  3. creates a Cairo surface,
  4. creates the compositor thread and the global compositor map if they haven't
     been created before (see `static void CompositorParent::StartUp()`).

     .. note::
        Only one compositor thread per gecko proccess is created.

  5. creates the image bridge thread connected to the compositor thread via
     the pair of actors `ImageBridgeParent` and `ImageBridgeChild`.

     .. note::
        The `PImageBridge` protocol is used to allow isolated threads or processes
        to push frames directly to the compositor thread/process (from the content
        thread) without relying on the main thread which might be too busy dealing
        with content script. Again only one image bridge thread per gecko process
        can be created.

In addition to that the fake widget creates

  1. an instance of `LayerManager` class;
  2. an instance of `EmbedLiteCompositorParent` class which is a subclass of the
     `CompositorParent` actor class and a `CompositorChild` end point. This
     `CompositorChild` instance serves as a communication channel to the
     compositor thread for the `LayerManager` object;
  3. a shadow manager (a child end point of the `LayerTransaction` subprotocol);
  4. a shadow forwarder connected to the shadow manager

and registers the shadow manager in the image bridge. Images drawn in the content
thread by the layer manager get forwarded through the image bridge to the
compositor thread which is supposed to render the images into a GL context.
See `this page <https://wiki.mozilla.org/Gecko:Overview#Graphics>`_ for a better
explanation of compositing.

.. warning::
   Currently the `EmbedLiteCompositorParent` class implements methods that are
   called from the main UI thread. But the object is supposed to live in the
   compositor thread. This may become a problem if UI and gecko get moved to
   separate processes.
