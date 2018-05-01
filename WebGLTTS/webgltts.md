---
layout: page
permalink: webgltts.html
---
# WebGLTTS Support
## Downloading and using WebGLTTS 

You can fork the project on GitHub or download it to your Unity3D project from the <a href="files/WebGLTTS.unitypackage">asset store</a>.

# How it Works
This section explains how the project works and the technologies that make it possible.

## Screen readers and DOM
Screen readers can represent a wide variety of hardware and software solutions that make it possible for users to read the graphical contents presented on screens. These readers may be simple assistive technologies that are often built into operating systems like Mac OS VoiceOver or specialized braille devices. Visual content on the screen can be thought of as pixels, this information is completely visual and cannot be processed by non-visual processes.

A way to translate these visual representations is to provide an equivalent textual representation. A webpage, is constructed from a `Document Object Model (DOM)`, which is a essentially a textual representation of the visual webpage. Screen readers have been used with web browsers for a long time and the technology landscape in this area is quiet mature today.

The use of `DOM` as an alternative source of consuming the visual webpage is not much useful for `WebGL` games. This is because the game is rendered in a `canvas` element. For the screen reader, parsing this `canvas` is impossible since it is just a collection of pixels.

WebGLTTS support provides a way to send text content to the `DOM` tree outside the `canvas` element, allowing developers to provide textual feeback for in-game interactions. This content is routed into an `ARIA Live` region which is picked up by the screen readers. This mechanism is the basis of this project.

## ARIA and ARIA Live Regions

`Accessible Rich Internet Applications (ARIA)` is an accesibility standard for mark-up languages - specifically `HTML`. `ARIA` provides a set of attributes that provide accessibility hints and directives to assistive software. Individual implementations of assistive software should ideally confirm to the standardized definitions of how to process these attributes. The Mozilla documentation on `ARIA` is a great starting point for understanding the role of this standard in enabling accessibility on the internet <a href="#ref1">[1]</a>.

## Talking with the Browser

To write to the `DOM` content tree from within the game, the Unity game code needs to talk to the world outside its `WebGL` `canvas`. This is done by calling JavaScript functions from within Unity scripts. The Unity script calls a JavaScript function to update the `ARIA Live` region with the text content supplied from the game. The interactions between web browser and Unity scripts are detailed in Unity3D documentation <a href="#ref2">[2]</a>.

# Understanding the Demo

This section explains the demo scene included with the WebGLTTS package.

<img class="img-responsive" src="images/DemoImage.png" />

## WebGL and Custom Templates

Available Unity3D WebGL templates do not have regions marked with special `ARIA` attributes to help screen readers. To be able to help screen readers identify content that it must *speak*, we need to add a special `DOM element` with the `aria-live` attribute that marks it as a live region. Any changes to the text content inside a live region alert the assistive software, depending on the `aria-live` attribute's politeness settings the assistive software may *speak* the updated contents to the user.

To make this work, we need to add an element specifically meant to provide `ARIA Live` region accessibility. The demo uses a customized WebGL template. The `index.html` for it looks as follows:

```
<!DOCTYPE html>
<html lang="en-us">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
    <title>%UNITY_WEB_NAME%</title>
    <link rel="shortcut icon" href="TemplateData/favicon.ico">
    <link rel="stylesheet" href="TemplateData/style.css">
    <style type="text/css">
    .hidden 
        {position:absolute;
        left:-10000px;
        top:auto;
        width:1px;
        height:1px;
        overflow:hidden;}
    *
        {margin: 0;
        padding: 0;}
    canvas
        {width: 100vw;
        height: 100vh;
        position: absolute;}
    </style>
    <script src="TemplateData/UnityProgress.js"></script>  
    <script src="%UNITY_WEBGL_LOADER_URL%"></script>
    <script>
      var gameInstance = UnityLoader.instantiate("gameContainer", "%UNITY_WEBGL_BUILD_URL%", {onProgress: UnityProgress});
    </script>
  </head>
  <body id="game">
    <div id="game-screenreader-content" aria-live="assertive" role="alert" class="hidden"></div>
    <div id="gameContainer"></div>
  </body>
</html>
```
You see the `div` with id `game-screenreader-content` which comes just before the game comtainer is the `ARIA Live` region. The `aria-live` attribute is set to `assertive` and its `role` is set to `alert`, this ensures that the content updates to this element are picked up by the screen reader at a higher priority.

The positioning of this element before the game container is intentional. This ensures that the live region is the first element that the screen reader will try to process. You will have to include such a customized element to serve as an `ARIA Live` region and note its `id` attribute which will be used to match this region with a SpeechManager object. More information about making custom WebGL templates is avaialbel on Unity3D manual <a href="#ref3">[3]</a>.

## Mapping SpeechManagers to ARIA Live Regions

Every SpeechManager instance needs to be mapped to one and only one `ARIA Live` region. This helps the SpeechManager identify the `DOM element` that it must route the text content to. SpeechManagers are connected to these `DOM elements` by specifying the `id` of the `DOM element` in the public member variable `domIdLiveRegion` of SpeechManager. 

<img class="img-responsive" src="images/SpeechManager.png" />

Once a SpeechManager instance has been initialized with a `domIdLiveRegion`, it cannot be reset or changed. It is **advised that the same `domIdLiveRegion` should not be assigned to multiple SpeechManagers, this may lead to confusion.**

## Mapping TTSContentBehavior to SpeechManager

TTSContentBehavior allows selectable components to describe themselves when under focus. You can think of every `GameObject` with TTSContentBehavior using a SpeechManager as a channel to communicate with the user. The SpeechManager simply facilitate the channeling of the description provided by the TTSContentBehavior of the `GameObject` in focus.

<img class="img-responsive" src="images/TTSContentBehavior.png" />

We must specify for every TTSContentBehavior, what must be the channel of delivery. In other words, which SpeechManager must the TTSContentBehaviour target. This is done by specifying the `speechManager` public member variable of TTSContentBehavior.

## Context and Commentary

The main role of any accessible experience is to effectively convery the context and provide meaningful commentary for the user. The user experience of browsing the internet using the `DOM` tree and assistive technologies is very well established, gaming does not provide such well established patterns. A game cannot be accessible to users who use assistive technologies like screen readers unless the user experience is meaningful. The use of WebGLTTS is not a solution in itself, it is tool that lets game developers use to craft rich experiences for their screen reader users.

## Extending for localization

WebGLTTS allows you to interface with localization software by assigning value to the delegate method member variable `ContentMethod`. The signature for the delegate method returns a string and takes a string parameter, the parameter is the `contentText` member variable of the TTSContentBehavior. 

Localization during runtime for web distributions is not possible because of limitations in HTML and how browsers and screen readers process pages. The first hint for the browser and the screen reader about the language of the webpage is the `<HTML>` tag's `lang` attribute which can be assigned an ISO 639-1 Language Code. Once the screen reader has read this tag, which may default to the users local language setting unless mentioned with the `lang` tag; you cannot change the language of the webpage without you reload it. **Care must be taken, that switching language attributes in webpages after a page load will not change the way the browser or the screen reader see the webpage, the browser must be reloaded**.

## Multiple Live Regions

You can have multiple `ARIA Live` regions inside the WebGL Template. Each of these regions will need to be addressed by their own SpeechManagers. Multiple regions may be used to ensure different text prompts of varying severity are channeled property. It is quiet unlikely to require multiple `ARIA Live` regions, but this is possible so long as each region has its own SpeechManager.

# WebGLTTSSupport Documentation

WebGLTTSSupport helps send out text from Unity 3D's C# or Unity Script code to the ARIA live regions of the WebGL build. This provides game developers the ability to interact with users who use screen readers.
<hr>
## SpeechManager.cs

SpeechManager is the central object for interacting with the screen reader compatiable `ARIA live regions` on the WebGL template. Every individual `ARIA live region` on the target WebGL template must have an associated SpeechManager. 

This association is specified by a public member variable `domIdLiveRegion` which is the `ID` for the `HTML DOM` item with the `aria-live` attribute, this value is specified via the inspector in Unity 3D.

Every SpeechManager has a private, thread-safe SpeechPipeline object as a member variable. This is created on the `Start()` method of the SpeechManager. The SpeechPipeline represents the textual content that will be piped out to the `DOM` element specified by 'domIdLiveRegion' member variable. 

### `public string domIdLiveRegion`

**This public property is the most important property for the SpeechManager.** It specifies the `id` of the `DOM element` that serves as the `ARIA Live` region. The value of the property depends on how your `WebGL Template` is constructed, this property is set when the SpeechManager object is initialized.

**This property cannot be manipulated at runtime.** It is strongly advised that every individual `ARIA Live` region must be represented by its own SpeechManager object.

### `public void Speak ( string content )`

Adds `content` to the speech pipeline. The speech pipeline will empty its contents into the `ARIA Live` region associated with the SpeechManager.

{:.table}
| field | type | description |
| ----- | ----- |----------- |
| `content` | parameter | The string contents that need to be written out to the `ARIA Live` region |

### `public void ClearSpeech ( )`

Clears the speech pipeline. All contents of the pipeline will be lost.

### `public void RepeatLastSaid ( )`

Repeats the contents last applied to the speech pipeline. Effectively, appends the string contents supplied to the latest call of `Speak (string content)` method of SpeechManager. **Should be used with caution in multi-threaded access to SpeechManager.** In multi-threaded access to SpeechManager, it may not be obvious what was last said. It is reasonable to assume that the order of `Speak (string content)` calls may have been different, in such cases depending on `RepeatLastSaid ()` is not advised. 

### `public void MuteSpeechManager ( bool mute )`

Mutes or un-mutes the `ARIA Live` region associated with the SpeechManager. This is achieved by setting the `aria-live` attribute of the target `ARIA Live` region to `off`, it is unmuted by resetting the same `aria-live` attribute to 'assertive'.

{:.table}
| field | type | description |
| ----- | ----- |----------- |
| `mute` | parameter | True, to mute the SpeechManager. False, to unmute the SpeechManager |

<hr>

## SpeechPipeline.cs

The SpeechPipeline is a private member of every SpeechManager. The SpeechPipeline represents the pipeline of text content to be written out to the associated `ARIA Live` region by the SpeechManager. **The SpeechPipeline provides a thread safe implementation of a simple queue.** This is useful when the SpeechManager may be accessed by multiple threads, it ensures the atomicity and integrity of the speech pipeline.

### `public string GetAndClearPipelineContent ()`

This method returns the consolidated contents of the speech pipeline, concatenating all of the pipeline contents into a single string. The method also clears the pipeline. It is used in the `Update ()` cycle of the SpeechManager where the latest contents of the pipeline are flushed into the `ARIA Live` region.

{:.table}
| field | type | description |
| ----- | ----- |----------- |
|  | return | Returns concatenated contents of the speech pipeline |

### `public void AppendSpeechPipeline ( string content )`

Appends the string supplied in `content` parameter to the speech pipeline.

{:.table}
| field | type | description |
| ----- | ----- |----------- |
| `content` | parameter | String content to be added to the speech pipeline |

### `public string GetLastSaid ( )`

Returns the content that was last added to the pipeline. **This will return the last added content, even after the speech pipeline has been cleared.**

This will return an empty string if nothing was added to the speech pipeline.

{:.table}
| field | type | description |
| ----- | ----- |----------- |
|  | return | Returns concatenated contents of the speech pipeline. |

<hr>

## TTSContentBehavior.cs

TTSContentBehavior implements the most common case of game interactivity, that of providing descriptions of `GameObjects` in focus to the SpeechManager. It is one way in which SpeechManager's APIs can be used to provide text descriptions for the user's interactions.

The class implements the `ISelectHandler` interface that works by intercepting select events on objects inheriting or attached with `Selectable` behavior. The class also has a provision to provide a text description or a dynamic value by way of a delegate method that uses the `contentText` public member variable as a parameter.

Most importantly, TTSContentBehavior serves as an example of what can be done with SpeechManager's APIs.

### `public string contentText`

This member variable represents the description of the `GameObject` that this is attached to. This also serves as parameter if a delegate method is supplied in `ContentMethod` member variable.

### `public ContentDelegate ContentMethod`

TTSContentBehavior gives developers two ways to provide descriptions for `GameObject`, first method being supplying a text definition in the public member variable `contentText` or using a public delegate method. If sucha delegate method is assigned, the `contentText` value is passed as a parameter to the method. This allows us to interface with localization code which will supply a different description depending on the language it is set on.

The signature for the delegate method is as follow:

`delegate string ContentDelegate( string param )`

The method is expected to return a string, this string will be supplied as a decription instead of the value in `contentText`.

### `public string GetContentText ( )`

Returns the string description of the `GameObject` that the TTSContentBehavior is attached to. It first checks if any `ContentDelegate` method is assigned - it returns the value of the delegate method. If no `ContentDelegate` is assigned, it returns the contents of the `contentText`.

{:.table}
| field | type | description |
| ----- | ----- |----------- |
|  | return | Returns the description of the `GameObject`. |

<hr>

# Frequently Asked Questions

### 1. Will adding this package to my project make it screen-reader accessible?
Yes, but there is more work involved. WebGLTTS is not a turn key solution and significant effort is needed to make any game screen reader friendly. The packahge provides you with an API to route the text from the game, out to an `ARIA Live` region.
### 2. How do I contribute to this project?
You can contribute by sending us a pull request, creating a new issue on GitHub or commenting on an issue. We would love to onboard new volunteers.
### 3. There is a bug in this code, where do I raise an issue for it?
Please file a new bug in the project's GitHub page.
### 4. What browsers does this support?
Most modern browsers support ARIA attributes. However minor variations between how the screen readers and browsers interpret these do exist. <a href="#ref3">[4]</a>.
### 5. Does this only work for Unity3D games?
Yes, it only supports Unity3D environment.
### 6. Does this only work for WebGL builds of Unity3D games?
Yes, this can only help with WebGL builds.

# References

<a id="ref1"></a>[1] Mozilla, ARIA Documentation. URL: <https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA>

<a id="ref2"></a>[2] Unity3D, WebGL: Interacting with browser scripting. URL: <https://docs.unity3d.com/Manual/webgl-interactingwithbrowserscripting.html>

<a id="ref3"></a>[3] Unity3D, Using WebGL Templates. URL: <https://docs.unity3d.com/Manual/webgl-templates.html>

<a id="ref4"></a>[4] Mozilla, Web applications and ARIA FAQ. URL: <https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Web_applications_and_ARIA_FAQ>
