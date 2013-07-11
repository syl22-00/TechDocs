What's up with audio in the web browser?
========================================

For anyone interested in audio software, it is an exciting time. A bunch of new technologies and standards are being developed and published that allow creating software with powerful audio features for web browsers. Developers do not have to worry anymore about operating systems, audio hardware, software installation and updates, and the availability of high level languages such as JavaScript (and also HTML and CSS) is making it possible to create complex and robust applications very quickly.

However, with many of these technologies being new, still evolving and often complex, it is not straightforward to understand what each of them does, how mature they are, which OSes and browsers support them, and how they can be linked together.

In this document, we try to overview and describe what they are, what they can and cannot do and the bridges between them. So we're going to talk about the [Web Audio API](http://chimera.labs.oreilly.com/books/1234000001552), [WebRTC](http://www.webrtc.org/), getUserMedia and the [HTML5 audio element](http://www.whatwg.org/specs/web-apps/current-work/#media-elements). We will also talk about recent web technologies are are not technically about audio but are very useful for building audio web applications, such as [Web workers](http://en.wikipedia.org/wiki/Web_worker) and [WebSockets](http://en.wikipedia.org/wiki/WebSocket).

# 1. Overview

A good starting point is to list out the major problems or use cases that these technology try to address:

* Web browsers should be able to play music or other types of audio (for instance for a web radio, or a cloud-based music player).
* You should be able to play games in your web browsers, with music and sound effects triggered in real-time.
* You should be able to create music in your browser, which implies things like synthesizers, mixing tables, samplers, etc.
* You should be able to record audio in your browser, and do anything you want with that recorded audio, for instance process and analyze it or send a voice message over the web.
* You should be able to communicate with other via voice, in your browser, and maybe even play music together, just like when you use Skype or Google Hangouts.

These are actually very diverse problems and more complex than just having access to the speakers or the microphone. You might notice that some of these would also apply for video, but for now I only focus on audio.

So a few of the technical issues to deal with are:

* Accessing speakers and audio hardware.
* Audio encoding and decoding from and to different formats.
* Real-time signal processing.
* Streaming audio through Internet.

Most of these topics have Wikipedia entries, and I'll give references along the way. However, one resource stands out, the [Web Audio API](http://chimera.labs.oreilly.com/books/1234000001552) book by Boris Smus.

# 2. Prehistory of audio in the browser

Before the web became a civilized place, there were things like Java applets, the `<embed>` tag, obscure things such as the quicktime and realplayer plug-ins, and Flash. With these you could play, and sometimes record audio, at the cost of security and stability issues and messages to users such as "This page can only be viewed on Internet Explorer" (on Windows of course). As just writing about these brings back painful memories, we'll just stop there.

# 3. The HTML5 `<audio>` tag

At the time people started boasting about HTML5, one of the most awaited features was the `<audio>` (and the similar `<video>`) tag. In short, the `<audio>` tag is for audio what the `<img>` tag is for images. An audio file can be included in a web page, either with the URL of an actual audio file or as a data URI (in which case, the binary data are embedded inside the HTML, encoded in base64), or even a binary [blob](https://developer.mozilla.org/en-US/docs/Web/API/Blob). There are optional displayable controls for playback, and there is a simple JavaScript API to control it, such as play, pause, preload and seek to approximate locations inside the file. One great thing with the audio tag is that it deals with audio streaming: you can start playing back a file as it is still being downloaded. There are a few JavaScript events you can register so that the application can keep track of the downloading and playing status.

Although it finally became available in most common web browsers (even including Internet Explorer), as with images in the previous century, the audio tag comes with his share of problems around supported data formats. For obscure reasons, even today, there is not one audio format that can be played on all browsers. Even well-established, non-patent-encumbered, high-quality, with-liberally-licensed-reference-implementations formats such as [OGG/Vorbis](http://www.vorbis.com/), [Speex](http://www.speex.org/), or [Opus](http://www.opus-codec.org/) are ignored by some major browsers. But at least a web developer can build a music player that works on the latest versions of all browsers, as long as he provides resources in several formats (MP3 and OGG/Vorbis should work). Even mobile browsers now support it.

There are also quite a lot of JavaScript libraries that solve this encoding issue by providing a flash fallback, like [jPlayer](http://jplayer.org/) and [MediaElement.js](http://mediaelementjs.com/).

The limitations of the audio tag, which are addressed by combining it with the Web Audio API (see later section) are:

* There is no access for the programmer to the actual audio data (the audio samples), just like you can't access pixel values of an image in an `<img>` tag. So no way to filter audio in any way.
* There is little control of audio playback level, so no way to carefully mix several audio tracks.
* Control of playback timing, either using the seeking capability, or starting playback at given time using `setTimeout` or `setInterval` is not accurate enough for games and music applications, due the the one-thread constrain of JavaScript.


# 4. The stillborn Audio Data API

As web developers were eager to access audio data in JavaScript and access the user's microphone, Mozilla proposed and implemented a JavaScript API that extended the API around the `<audio>` element. It was called the [Audio Data API](https://wiki.mozilla.org/Audio_Data_API). Browsers other than Firefox have not implemented it, and it has been deprecated in favor of more general solutions such as getUserMedia and the Web Audio API.

# 5. The Web Audio API

The Web Audio API is pretty new, mainly available on Chrome, but support in Safari and Firefox is coming very soon. It aims to make it easier for developers of modern, web-based, games and music applications where processing audio in real time is a must-have. It can:

* Very accurately trigger audio playback at specific time slots,
* Mix several audio inputs with different gains,
* Apply many types of filters to audio inputs,
* Expose audio samples to the programmer.

The Web Audio API is used by building audio graphs, where audio data go from the input to the output, going through different nodes to process them. As the output is usually the speakers, input can be either audio samples, or retrieved from components outside the Web Audio API, such as the audio tag or getUserMedia. So for instance, the Web Audio API itself does not stream audio files (that's what the audio tag does), and does not access the microphone (that's what getUserMedia does). But it can process data, either through its own API, which includes many types of filters, or through JavaScript functions made by web developers. If possible, the available functions in the API should be preferred as they run much faster than JavaScript code.

The entry point for the Web Audio API is the audio context, which is used to create nodes that are then connected to each other.

## 5.1 Bridging the Web Audio API and the `<audio>` tag.

A node can be created from a media element with the `createMediaElementSource` function:

    var context = new webkitAudioContext();
    var audio = new Audio();
    source = context.createMediaElementSource(audio);

## 5.2 Bridging the Web Audio API and the audio input (such as microphone).

As we'll see in the next section, the API to access the microphone is getUserMedia. This API is based on the concept of media streams. So in order to get the actual audio samples from the stream, one must create a node whose input is a stream created by getUserMedia:

    var context = new webkitAudioContext();
    navigator.webkitGetUserMedia({audio: true}, 
        function(stream) {
            var input = context.createMediaStreamSource(stream);
            input.connect(context.destination);
        },
        function() {
            alert("Error trying to access user media");
        }
    );

Then, this input node can be connected to a processing node created by `createScriptProcessor`, which exposes the audio samples in its `onprocess` callback. An example can be seen in [Recorderjs](https://github.com/mattdiamond/Recorderjs).

# 6. WebRTC and getUserMedia

WebRTC is THE popular new web technology this year. It is closely related to the latest-and-greatest Google app: Hangouts. WebRTC is for creating real-time, peer-to peer communication applications in the browser. Just like Skype in the browser. It is the successor of Google Talk, and, with WebRTC being a standard, open-source, and available in several browsers, Hangout now works without browser plug-ins. WebRTC now being usable in both Chrome and Firefox, and open-source, developers can now create distributed communication applications without relying on any commercial or proprietary technology provider, which opens great business opportunities.

WebRTC is actually the combination of several components. The one that interests us is getUserMedia, which allows developers to access the user's microphone. What getUserMedia provides is MediaStream objects, which is yet another concept than what we have seen with the audio tag and the Web Audio API. However, as we saw previously, the MediaStream objects can also be used to create audio nodes. So here, the Grail is `createMediaStreamSource`, which is for now only available on Chrome.

# 7. Web workers, WebSockets and Typed Arrays

As we saw, with the Web Audio API, one can access the audio samples as they are being produced, inside a node created with `createScriptProcessor`. It is usually preferred to use Web Audio API functions to process them, as the browser uses native code, but you can also process the samples in JavaScript. In this case, you can process them inside a Web worker, leaving the UI thread free from that kind of heavy work.

Meanwhile, as digital audio data is large but takes time to be produced, one can make use of WebSockets and stream them as they are recorded. That can be used for instance to send audio messages to other people, or process audio on a server, as a web service. Using WebSockets, the binary nature of audio streams is not an issue, and any size of data can be sent. You would have harder time putting a large blob in a POST request.

Supporting a lot of these technologies are the new Typed Arrays which offer much better performance than the original JavaScript arrays that can contain any type of data. Audio is usually stored in arrays of short integers (for the most common audio encodeing, on 16 bits), so our friend here is `Int16Array`:

    var buffer = new ArrayBuffer(2 * 16000); // One second at 16kHz
    var array = new Int16Array(buffer);

# 8. Browser support

Chrome is by far the most cutting edge browser for audio features and supports all the technologies described here. However, some of these features are buggy, for instance live audio capture using `getUserMedia`, `createMediaStreamSource` and `createScriptProcessor` often produces silent audio. The status and possible fixes for this issue can be tracked in [this](http://code.google.com/p/chromium/issues/detail?id=112367) and [this](https://code.google.com/p/chromium/issues/detail?id=170384) bug reports.

Firefox is catching up, it is now WebRTC ready and is adding the Web Audio API in the coming releases (23 and 24). Mozilla maintains a nice [Web Audio API Rollout Status](https://wiki.mozilla.org/WebAudio_API_Rollout_Status) page. Mozilla has just announced that they are [pretty much ready](https://hacks.mozilla.org/2013/07/web-audio-api-comes-to-firefox/), although the very important bridges `MediaStreamAudioSourceNode` and `MediaElementAudioSourceNode` are not done yet.

The site <http://caniuse.com/> summarizes the support for web technologies by web browsers, here are the links to the points we discussed:

* [Audio tag](http://caniuse.com/audio)
* [Web Audio API](http://caniuse.com/audio-api)
* [getUserMedia and Media streams](http://caniuse.com/#feat=stream)
* [WebSockets](http://caniuse.com/websockets)
* [Web workers](http://caniuse.com/webworkers)
* [Data URIs](http://caniuse.com/datauri)
* [Blob URLs](http://caniuse.com/bloburls)
* [Typed Arrays](http://caniuse.com/typedarrays)