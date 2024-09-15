# The `<model>` element

## Authors:

- [Antoine Quint](https://github.com/graouts)
- [Dean Jackson](https://github.com/grorg)
- [Theresa O'Connor](https://github.com/hober)
- [Marcos Cáceres](https://github.com/marcoscaceres)
- [Brandel Zachernuk](https://github.com/zachernuk)

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [tl;dr](#tldr)
- [Introduction](#introduction)
- [Non-goals](#non-goals)
- [The HTMLModelElement](#the-htmlmodelelement)
- [DOM API](#dom-api)
- [JavaScript API](#javascript-api)
- [DOM Events](#dom-events)
- [Visual presentation control](#visual-presentation-control)
  - [Scale units](#scale-units)
  - [Stage interaction mode](#stage-interaction-mode)
    - [Orbit scale](#orbit-scale)
  - [Lighting](#lighting)
  - [Audio and stateful interaction](#audio-and-stateful-interaction)
- [Playback and accessibility considerations](#playback-and-accessibility-considerations)
- [Privacy considerations](#privacy-considerations)
- [Security considerations](#security-considerations)
- [Detailed design discussion](#detailed-design-discussion)
  - [Why add a new element?](#why-add-a-new-element)
  - [Rendering](#rendering)
- [Considered alternatives](#considered-alternatives)
- [Additional reading](#additional-reading)
- [Acknowledgements](#acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## tl;dr

We propose adding a `<model>` element to HTML that displays 3D content using
a renderer built-in to the browser.

## Introduction

HTML allows the display of many media types through elements such as `<img>`,
`<picture>`, or `<video>`, but it does not provide a declarative manner to directly
consume 3D content. Embedding 3D content within a page is comparatively
cumbersome and relies on scripting the `<canvas>` element. We believe it is
time to put 3D models on equal footing with other, already supported, media
types.

There is a long history of presenting 3D content on the Web: For example,
[three.js](https://threejs.org/) and [Babylon JS](https://babylonjs.com/)
are WebGL frameworks that can process many different formats. Then there is
the [model-viewer](https://modelviewer.dev) project which shows models
inline in a web page, and also allows users on some devices to see the 3D
object in augmented reality. And iOS Safari has the ability to navigate
directly to an augmented reality view with its 
[AR Quick Look feature](https://webkit.org/blog/8421/viewing-augmented-reality-assets-in-safari-for-ios/).

However, there are cases where these current options cannot render content.
This is due to security restrictions or to technical limitations of `<canvas>`
(see below for [more details on motivation](#detailed-design-discussion)).

The HTML `<model>` element aims to allow a website to embed interactive 3D models as
conveniently as any other visual media. Models are expected to be created by 3D authoring
tools or generated dynamically, but served as a standalone resource by the server.

Additionally, besides the simple display of a 3D model, the `<model>` element should have
support for manipulation and animation playback while presented within the page, and also support
more immersive experiences, such as augmented reality.

## Non-goals

This proposal does *not* aim to define a mechanism that allows the creation of a 3D scene
within a browser using declarative primitives or a programmatic API. 

While some popular model formats have the ability to encode and relate audio tracks, <model>
is expected to present silently and will not honor audio components.

Many model formats include the ability to blend animation tracks, <model> is expected to play
only the first animation track discovered in a valid source asset.

Many model formats include the ability to encode stateful interaction with the scene, for example
with selection targets linked to the selective showing and hiding of content. <model> is expected
to only manage a single, linear animation timeline, and the manipulation of the `entityTransform`
that dictates the apparent scale, orientation, and position offset of the entire scene as an
atomic element.

## The `HTMLModelElement`

The `<model>` element is a new
[replaced](https://drafts.csswg.org/css-display/#replaced-element) HTML
element similar to `<video>` in that it is replaced visually by the content
of an external resource referenced via a `<source>` element. Like other
HTML elements, it can be styled using CSS.

The resource is resolved by selecting the first, most appropriate `<source>`
element with supported `type` and `media` attributes, allowing different versions of the same
resource in different formats to be specified. See the HTML specification for
the definition of
[type](https://html.spec.whatwg.org/multipage/embedded-content.html#attr-source-type) and
[media](https://html.spec.whatwg.org/multipage/embedded-content.html#attr-source-media).

This is an example showing a 400px by 300px model, allowing the browser to choose between a
[USDZ](https://graphics.pixar.com/usd/docs/Usdz-File-Format-Specification.html)
file and a [glTF](https://www.khronos.org/gltf/) file, depending on what the browser supports.

```html
<model style="width: 400px; height: 300px">
    <source src="assets/example.usdz" type="model/vnd.usdz+zip">
    <source src="assets/example.glb" type="model/gltf-binary">
</model>
```

Browsers may support direct manipulation of the `<model>` element while presented in the page. For example, a browser
may allow the model to be rotated or zoomed within the element's bounds without affecting the scrolling
position or zoom level of the page. To opt into this behavior, the author may set the `stagemode`
HTML attribute to `orbit`.

The previous example can be augmented to allow free orbiting of the model, provided by the browser:

```html
<model style="width: 400px; height: 300px" stagemode="orbit">
    <source src="assets/example.usdz" type="model/vnd.usdz+zip">
    <source src="assets/example.glb" type="model/gltf-binary">
</model>
```

It is also possible that browsers support an animated presentation of the model, by running
animations defined in the source data. Such animations are not enabled by default, but can
be triggered on load by using the `autoplay` HTML attribute.

The original example can be augmented to allow for such animation:

```html
<model style="width: 400px; height: 300px" autoplay>
    <source src="assets/example.usdz" type="model/vnd.usd+zip">
    <source src="assets/example.glb" type="model/gltf-binary">
</model>
```

The `stagemode="orbit"` and `autoplay` conditions are not mutually exclusive and may be combined. A browser can
run a default animation while the user interacts with the model.

As such, the original example can be augmented to allow for both animations and free orbit:

```html
<model style="width: 400px; height: 300px" autoplay stagemode="orbit">
    <source src="assets/example.usdz" type="model/vnd.usd+zip">
    <source src="assets/example.glb" type="model/gltf-binary">
</model>
```

Like the `<video>`, the `<model>` element has an optional `poster` attribute that references
an image to be shown while the content is being loaded, or if the content fails to load.

Here is [an example](examples/index.html) of the `<model>` element. On a browser that has implemented
the element, it should appear as in the image below.

![Ha-Ha iMessage tap-back bubble](https://github.com/WebKit/explainers/raw/main/model/assets/haha.png)

### Fallback content

In the case where `<model>` can not display any of its `<source>` children, it
will fall-back to showing its last non-`<source>` child that is a replaced
element. In the example below, this would mean the contents of the `<picture>`
element would be displayed.

```html
<model>
    <source src="fake.typ1" type="imaginary/type-1">
    <source src="fake.typ2" type="imaginary/type-2">
    <picture>
        <source src="animated-version.mp4" type="video/mp4">
        <source src="animated-version.webp" type="image/webp">
        <img src="animated-version.gif"/>
    </picture>
</model>
```

## DOM API

Each `<model>` element is represented in the DOM as `HTMLModelElement` instances.

The following properties allow easy access to information otherwise represented by HTML
attributes and elements:

* `currentSrc`: read-only string returning the URL to the loaded resource. To change the loaded
resource, the author should use existing DOM APIs to add, remove or modify `<source>` children
elements to the `<model>` element.
* `autoplay`: read-write boolean indicating whether the model will automatically start playback.
Setting this property to `false` removes the `autoplay` HTML attribute if present, while setting it to `true`
adds the `autoplay` HTML attribute if absent.
* `stagemode`: read-write string indicating whether user input automatically 
results in changing the display orientation of the model. Setting this
property to anything but `orbit` removes the automatic orbit behavior, while setting it to `orbit`
sets the stage mode to orbit.
* `environmentmap`: read-write string indicating the URL of an environment map, also known as an Image-Based Light (IBL). Supplied as an equirectangular image, frequently in a High-Dynamic Range
(HDR) image format.
* `loop`: read-write boolean indicating whether the model animation, if present, will automatically loop.
* `loading`: behaves in the same manner as the
[`img` attribute of the same name](https://html.spec.whatwg.org/multipage/embedded-content.html#attr-img-loading).
* `poster`: behaves in the same manner as the [`video` attribute of the same name](https://html.spec.whatwg.org/multipage/media.html#attr-video-poster)

Similar to other elements with sub-resources, the `HTMLModelElement` will provide
APIs to observe the loading and decoding of data.

While HTML supports the notion of taking an element fullscreen, browsers may want to offer yet more
immersive experiences that require going beyond the page itself, one example would be to present the
model in augmented reality to allow the user to visualize it at real scale in the user's immediate
surroundings. To support this, new DOM APIs may be added or the existing HTML fullscreen API extended
via more [FullscreenOptions](https://fullscreen.spec.whatwg.org/#dictdef-fullscreenoptions) properties.

## JavaScript API

In addition to the DOM API relating to the source, animation, and environment map, the JavaScript API has
additional capabilities relating to the animation timing and view parameters. 

* `entityTransform`: a read-write `DOMMatrixReadOnly` that expresses the current mapping of the view of
the model contents to the view displayed in the browser. 
* `boundingBoxCenter`: a read-only `DOMPoint` that indicates the center of the axis-aligned bounding box (AABB)
of the model contents. If there is an animation present, the bounding box is computed for the first frame of 
the animation and remains static for the lifetime of the model. It does not update based on a change of
the `entityTransform`.
* `boundingBoxExtent`: a read-only `DOMPoint` that indicates the extent of the bounding box of the model
contents. 
* `boundingSphereCenter`: a read-only `DOMPoint` that indicates the center of the bounding sphere of the model contents, as it may differ from the bounding _box_ center. 
* `boundingSphereRadius`: a read-only `double` that indicates the radius of the bounding sphere of the model contents. 
* `duration`: a read-only `double` reflecting the un-scaled total duration of the animation, if present.
If there is no animation on this model, the value is 0.
* `currentTime`: a read-write `double` reflecting the un-scaled playback time of the model animation, if present.
It is clamped to the duration of the animation, so for an animation with no animation, the value is always 0.
* `playbackRate`: a read-write `double` reflecting the time scaling for animations, if present. For example,
a model with a ten-second animation and a `playbackRate` of 0.5 will take 20 seconds to complete.
* `paused`: A read-only `Boolean` value indicating whether the element has an animation that is currently playing.
* `play()`: A method that attempts to play a model's animation, if present. It returns a `Promise` that resolves
when playback has been successfully started.
* `pause()`: A method that attempts to pause the playback of a model's animation. If the model is already paused
this method will have no effect.

## DOM Events

While the author may prevent any built-in interactive behavior for a `<model>` by ommitting the `stagemode`
attribute, it might be desirable for the decision to allow custom control of the model behavior at runtime.
To that end, when a user initiates a gesture over a `<model>` element, the author may call the `preventDefault()`
method when handling the `pointerdown` event. If this method is not called for the
[`pointerdown`](https://www.w3.org/TR/pointerevents/#the-pointerdown-event) event for the
[primary pointer](https://www.w3.org/TR/pointerevents/#the-primary-pointer) of a gesture, calling
`preventDefault()` for any additional pointer event will have no effect.

The `mousedown` and `touchstart` compatibility events may also be used for this purpose.

* `load`: Dispatched when the model's source file has been loaded and processed, such that the
bounding box information is available and the animation duration, if present, is known.
* `error`: Dispatched if the the model's source file is unable to be fetched, or if the file
cannot be interpreted as a valid <model> asset.
* `iblload`: Dispatched when a model's selected environmentmap has been loaded and is ready to
contribute to the visual appearance of the model.
* `iblerror`: Dispatched if there has been an issue with the model's selected environmentmap,
which will prevent its ability to act as the lighting environment.

### DOM actions

In addition to being a standard DOM element, special behaviors may be desirable on spatial platforms,
relating to the `Element.requestFullscreen()` and the `document.pictureInPictureElement` properties.

* `document.pictureInPictureElement`: By requesting the model to be the picture-in-picture element,
the model becomes separated from the rest of the active window, and positionable within the user's space.
its real-world location is not made available to the page context. In addition to a JavaScript-based
request of this feature, a user gesture may be used to both invoke the action and position the element.

* `Element.requestFullscreen()`: A `<model>` that is part of a DOM fragmented and has been granted 
fullscreen access on a spatial platform may be presented with a transparent background.

## Visual presentation control

In a spatial or stereoscopic environment, A model element presents its three-dimensional content
 as though it exists inside a portal in a page, with content clipped at the z-boundary of the model 
 element's front face, so that no content has the ability to protrude beyond the appropriate surface.

The visual presentation of a model is primarily managed through its `entityTransform` attribute, 
specified as a `DOMMatrixReadOnly`. The model scene is a right-handed, Y-up coordinate system, with the
center of the view-plane at (0,0,0). 
<!-- Diagram of the coordinate space -->

The initial `entityTransform` is set to center the view on the `boundingBoxCenter`, set back by
`boundingBoxExtent.z/2` so that the element resides entirely within the portal, and to fit the
`boundingingBoxExtent` within the visible view. That is, to set a uniform scale such that the
`boundingBoxExtent.y` fits within the model's portal height, and the `boundingBoxExtent.x` fits within
the model's portal width.
<!-- Diagram of the default/autofit scale -->

### Scale units
The page dimensions are understood to be reflected as points, such that a 1000-pixel model element
with its `entityTransform` set to the identity will depict a size of about 32cm, and a 500-pixel
model will likewise show a viewport covering 16cm. For privacy reasons, user agents may obscure the
true spatial scale of the window, so that 1000 pixels may be much larger or smaller - a fixed conversion
assuming 72DPI as a default should be understood as standard. This only reflects a mapping to the 
perceptual scale, and doesn't reflect the _rasterization_ scale of the element, which occurs at the
discretion of the user agent.
<!-- Diagram of the visual scale with real-world units -->

### Stage interaction mode
Setting the `stagemode` attribute to `orbit` results in an _orbit_ interaction mode, where the 
`entityTransform` becomes read-only (or attempts to write directly to it are ignored), and the view is
updated exclusively based on input events from the user. Dragging on the model horizontally
results in a rotation on the Y axis of the model, and vertical dragging alters the pitch of the 
element:

<!-- Diagram of the input events in orbit mode -->

#### Orbit scale

Because the orbit mode may result in presenting any arbitrary orientation of the model contents,
the effective scale of the model is reduced to accommodate the bounding _sphere_, rather than the
bounding box, and the setback in the z-dimension is also set to this bounding sphere radius.
<!-- Diagram of the scale and setback in and out of orbit mode -->

### Lighting
The model may be illuminated by a system-default environment map, or by an environment map as specified
by a valid, author-specified `environmentmap` attribute. On a platform with the ability to estimate
the user's lighting environment, this may be applied as a default map. 
A physically-based material model like [OpenPBR] or [MaterialX] is assumed.
<!-- screenshot of two different IBLs on the same asset or assets -->


## Audio and stateful interaction

While Some model source formats have the ability to specify audio sources, it
is recommended that an initial implementation exclude audio playback. This is
because audio playback is achievable through separate APIs, and the relationship 
between arbitrary seeking inside a model animation and the audio sources therein
is not well-established.

## Playback and accessibility considerations

Model resources may contain animations and as such should be
considered like other media and animated content by browsers. This means
that browser behaviors around loading, autoplay and accessibility should be
honored for the `<model>` element as well, for instance:

- a static poster image may be displayed prior to loading the full `<model>` resource,
- playback may be disabled if the user has set a preference to reduce animations.

Like other timed media, the `<model>` element will provide a DOM API for playing, 
pausing, etc.

The `<model>` element has an `alt` attribute to provide a textual description of the
content. Also, the 3D content itself might expose some features to the accessibility engine.

## Privacy considerations

Rendered `<model>` data is not exposed to / extractable by the page in this
proposal, so no tainting is required. We do expect this would require
extensions to Fetch (a new destination type), Content Security Policy (a new
policy directive), and likely a few others.

## Security considerations

As always, introducing support for parsing and processing new formats raises the surface area
of attach posibilities in a browser.

However, some existing browsers already process such formats in a non-inline manner
(such as iOS's AR Quick Look and Android's Scene Viewer).

## Detailed design discussion

### Why add a new element?

We believe it is time for files representing 3D geometric data to become a first class
citizen on the Web.

Adding a new element to HTML requires significant justification. At first glace, the `<model>` element
does not appear necessary since HTML already provides a mechanism to load arbitrary data and
render it: `<canvas>` and its rendering contexts.

So why add a new element?

Firstly, we believe that content such as this is important enough that it should not require
a third-party library. Like raster images, vector images, audio and video, three-dimensional
geometric data should be a data type that can be directly embedded in HTML content.

Secondly, while we are not proposing a DOM for the data at the moment, we expect to in the
future. It would be of benefit to the Web developer to learn a single common API for 3D
geometry rather than learn the API of various third-party libraries. Furthermore, different
file types would then conform to the same API.

Thirdly, there are cases where a JavaScript library cannot render content. This might be due to
security restrictions or to the limitations of `<canvas>`, which is bound to a flat two-dimensional
surface in the web page.

Consider a browser or web-view being displayed in Augmented Reality. The developer wants to
show a 3D model in the page. In order for the model to look accurate, it must be rendered
from the viewpoint of the user - otherwise it is a flat rendering of a three-dimensional
image with incorrect perspective.

A solution to this would be to allow the Web page, in particular the WebGL
showing the 3D model, to render from the perspective of the user. This would
involve granting too much private information to the page, possibly including
the camera feed, some scene understanding, and very accurate position data on
the user. It should not be a requirement that every Web page has to request
permission to show a 3D model in this manner. The user should not have to
provide access to this sensitive information to have the experience.

Furthermore, there is more information needed to produce a realistic rendering, such as
the ability to cast and receive shadows and reflections from other content in the scene, that
might not be on the same Web page.

This means that rendering in Augmented Reality is currently limited to a system
component, different from the in-page component, leading to inconsistent results.


### Rendering

Unfortunately it is impractical to define a pixel accurate rendering approach for the `<model>` element. If such
an attempt was made, it would likely pose too many restrictions on the browser engines, which have to work
on a number of operating systems, hardware and environments.

Instead we suggest adopting a Physically-Based Rendering approach, probably referencing an existing
shading model such as [MaterialX](https://materialx.org/). Browsers would be free to
implement the system as they wish, with a goal of producing the most accurate rendering possible.
We do not expect pixel-accurate results between browsers.

While this is a clear problem, it also comes with some large advantages.

- Improvements in hardware should see improvements in rendering quality.
- The quality of the rendered content may improve without requiring a change in the source.
- The browser can use the environment to make a more realistic display. For example, reflections or shadows
  cast by other elements in the AR scene (another thing that would be impossible for page content to have access to).

For reference, the Model Viewer project has a [rendering engine fidelity comparison](https://modelviewer.dev/fidelity/).


## Considered alternatives

1. *Reuse `<embed>` or `<object>` instead of adding a new element*

   It would be possible to reuse one of the generic embedding elements, such as `<embed>` or `<object>`,
   for this purpose. However, we think that 3D content should behave similar to other media types.

2. *Reuse `<img>`, `<picture>` or `<video>` instead of adding a new element*

   One can consider a 3D rendering to be an image or movie, but we expect there to be differences in
   interactivity.

3. *A simple `src=""` attribute instead of `<source>` children*

   Like `<audio>` and `<video>`, there are several widely-used formats that authors might wish to use,
   and browser support for these formats may vary. Given this, providing multiple `<source>`s seems desirable.

4. *Do not add a new element. Pass enough data to WebGL to render accurately*

    As noted above, this would require any site that wants to use an AR
    experience to request and have the user trust that site enough to allow
    them access to the camera stream as well as other information. A new
    element allows this use case without requiring the user to make that decision.

## Additional reading

For additional insight into the history and how we see the potential evolution
of the `<model>` element going, please see the ["`<model>` Evolution"](https://github.com/WebKit/explainers/blob/main/model/HistoryAndEvolution.md)
companion document.

## Acknowledgements

Many thanks for valuable feedback and advice from:

- Sam Sneddon
- Sam Weinig
- Simon Fraser
