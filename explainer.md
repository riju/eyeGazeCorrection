# Eye Gaze Correction

## Authors:

- Rijubrata Bhaumik, Intel Corporation
- Eero Häkkinen, Intel Corporation
- Youenn Fablet, Apple Inc.

## Participate
- github.com/riju/eyeGazeCorrection/issues/

## Introduction

Eye Contact / Gaze Redirection is a new technique designed to help ensure that your eyes are looking at the camera when presenting in a video call.

Camera placement can determine how we are looking in the video but now novel techniques can shift our eyes and create a sense that we are always making eye contact, which is a powerful social stimulus. That helps to create a sense of connection and engagement with the viewer. Many studies have shown that perceiving other individuals’ direct gaze has robust effects on various attentional and cognitive processes.

In a typical video conferencing setup, it is hard to maintain eye contact during a call since it requires looking into the camera rather than the display. Studies show increase in the amount of eye contact made by a speaker can significantly increase their perceived credibility. However, a typical video conferencing setup creates a gaze disparity that breaks the eye contact, resulting in unnatural interactions. Many videoconferencing capable devices have a display and camera that are not aligned with each other. During video conferences, users tend to look at the other person on display or even a preview of themselves rather than looking into the camera. The gap between the camera and where the users typically look at makes it hard to maintain eye contact and have a natural, face to face, conversation.

This Eye Gaze Correction API gives developers a **choice** to use the native platform's API. This would ensure conformance to the corresponding native apps.


## Use Cases

Not everybody can talk through a presentation or a lecture without any notes. A telepromter allows you to appear natural and speak off the cuff for your audience. In the video conference world, that should translate to reading through a script, yet keeping an eye contact with the audience.

Content creators can now record themselves while reading their notes or a script without having to look directly at a camera.


## Goals

Right now Apple had offers a boolean ON/OFF to control the Eye Gaze Correction feature. NVIDIA Broadcast 1.4 also today offers the same boolean to control this feature. Windows 11 22H2 offers an additional **Stare** mode akin to a teleprompter. Taking these data into consideration and the aim to provide similar knobs as offered to native platform, if it makes sense, we plan to make the Eye Gaze Correction use an enum. We think content creation on the Web is going to grow and content creators who use the web platform for broadcasting and connecting with their audience would enjoy the **stare** mode enhancement along with the normal eye gaze correction.

## Non-goals


## Performance

FaceTime has this feature called [Attention Correction](https://techcrunch.com/2019/07/03/apples-ios-13-update-will-make-facetime-eye-contact-way-easier/) from 2018.

Nvidia Broadcast 1.4 added the [“Eye Contact feature"](https://www.nvidia.com/en-us/geforce/news/jan-2023-nvidia-broadcast-update/) in 2023.

On Windows and Mac platforms, Eye Gaze Corretion would leverage NPU (Neural Processing Unit). Windows 11 and a explicit NPU/ VPU is a  requirement. 

Today, only selected models of Windows 11, having NPU will have the Eye Gaze Correction feature, but within this year, NPUs on PCs are going to be widespread and hence Eye Gaze Correction should work on all Meteor Lake systems and beyond and also on AMD systems with NPUs and Spandragon SQ3.

The performance will vary based on the models used in the platform.

## User research

* TEAMS : 

* MEET : 

* Zoom : 

## Eye Gaze Correction API

This is about bringing platform automatic eye gaze correction to the web so constrainable media track capabilities fit naturally to the purpose. Because the aye gaze correction is (or should be) implemented by the platform media pipeline, it is enough for an (web) application to control the eye gaze correction through constrainable properties. On Apple devices, Eye gaze is controlled globally from Command Center and as such any app cannot independently switch on/off the feature. On Windows 11 platform, version 22H2, apart from the normal Eye Gaze correction which solves for the geometrical problem of camera-display offset, there's another option called the **Stare** mode which is useful while reading a presentation/ document in a call, like a teleprompter. **Stare** is a more aggressive form of Eye Contact that continually shifts the pixels of the eyes to make it look like you are speaking with your audience even though you might be reading off a script and moving the eyeball rapidly.


```js
  enum EyeGazeCorrectionMode {
    "off",
    "normal",
    "stare" // telepromter mode.
  }

  partial dictionary MediaTrackSupportedConstraints {
    boolean eyeGazeCorrection = true;
  };

  partial dictionary MediaTrackCapabilities {
    sequence<DOMString> eyeGazeCorrection;
  };

  partial dictionary MediaTrackConstraintSet {
    ConstrainDOMString eyeGazeCorrection;
  };

  partial dictionary MediaTrackSettings {
    DOMString eyeGazeCorrection;
  };
  ```

[Spec Discussions](https://github.com/w3c/mediacapture-extensions/pull/56)


## Exposing change of MediaStreamTrack configuration

The configuration (capabilities, constraints or settings) of a MediaStreamTrack may be changed dynamically outside the control of web applications.
One example is when a user decides to switch on eye gaze correction through the operating system.
Web applications might want to know that the configuration of a particular MediaStreamTrack has changed.
For that purpose, a new event is defined below.

```js
partial interface MediaStreamTrack {
  attribute EventHandler onconfigurationchange;
};
```

[PR under discussion](https://github.com/w3c/mediacapture-extensions/pull/56)

## Example

 * main.js:
   ```js
   // main.js:
   // Open camera.
   const stream = await navigator.mediaDevices.getUserMedia({video: true});
   const [videoTrack] = stream.getVideoTracks();

   // Use a video worker and show to user.
   const videoElement = document.querySelector('video');
   const videoWorker = new Worker('video-worker.js');
   videoWorker.postMessage({videoTrack}, [videoTrack]);
   const {data} = await new Promise(r => videoWorker.onmessage);
   videoElement.srcObject = new MediaStream([data.videoTrack]);
   ```
 * video-worker.js:
   ```js
   self.onmessage = async ({data: {videoTrack}}) => {
     const processor = new MediaStreamTrackProcessor({track: videoTrack});
     let readable = processor.readable;

     const videoCapabilities = videoTrack.getCapabilities();
     if ((videoCapabilities.eyeGazeCorrection || []).includes(true)) {
       // The platform supports eyeGazeCorrection and
       // allows it to be enabled.
       // Let's try use platform eyeGazeCorrection.
       await track.applyConstraints({eyeGazeCorrection: true});
     }
     const videoSettings = videoTrack.getSettings();
     if (videoSettings.eyeGazeCorrection) {
       // eyeGazeCorrection is enabled.
       // No transformer streams are needed.
       // Pass the original track back to the main.
       parent.postMessage({videoTrack}, [videoTrack]);
     } else {
       // The platform does not support eyeGazeCorrection or
       // does not allow it to be enabled.
       // Let's use custom face detection to aid custom eyeGazeCorrection.
       importScripts('custom-face-detection.js', 'custom-eyeGazeCorrection.js');
       const transformer = new TransformStream({
         async transform(frame, controller) {
           // Use a custom face detection.
           const detectedFaces = await detectFaces(frame);
           // Use a custom eyeGazeCorrection.
           const newFrame = await eyeGazeCorrection(frame, detectedFaces);
           frame.close();
           controller.enqueue(newFrame);
         }
       });
       // Transformer streams are needed.
       // Use a generator to generate a new video track and pass it to the main.
       const generator = new VideoTrackGenerator();
       parent.postMessage({videoTrack: generator.track}, [generator.track]);
       // Pipe through a custom transformer.
       await readable.pipeThrough(transformer).pipeTo(generator.writable);
     }
   };
   ```

## Security considerations

The eyeGazeCorrection capability is provided by the User-Agent and cannot be modified by web applications. It may however change for instance if the user uses operating system controls to enforce eyeGazeCorrection or to remove such an enforcement.

The eyeGazeCorrection constraints are provided by web applications by passing them to `navigator.mediaDevices.getUserMedia()` or to `MediaStreamTrack.applyConstraints()`. Constraints allow web applications to change settings within the bounds of capabilities.

## Privacy considerations

When a user uses eyeGazeCorrection, the system would use novel techniques in AI to trick audience that we are looking at them by synthesizing images of our eyes, so essentially the system is faking eye contact.

### Fingerprinting

If a site does not have [permissions](https://w3c.github.io/permissions/), eyeGazeCorrection provides practically no fingerprinting posibilities.
The only provided information is `navigator.mediaDevices.getSupportedConstraints().eyeGazeCorrection` which either is true (if the User-Agent supports eyeGazeCorrection in general irrespective of whether the device has a camera or a platform version which support eyeGazeCorrection) or does not exist.
That same information can most probably obtained also by other means like from the User-Agent string.

If a site utilizes `navigator.mediaDevices.getUserMedia({video: {}})` which resolves only after the user has [granted](https://w3c.github.io/permissions/#dfn-granted) the ["camera"](https://www.w3.org/TR/mediacapture-streams/#dfn-camera) permission, the returned video tracks may have `eyeGazeCorrection` capabilities and settings.

Based on the capability, it is possible to determine if the platform is one which allows application only to observe eyeGazeCorrection setting changes or one which allows applications also to set the eyeGazeCorrection setting.
In essence, this splits operating systems to two groups but does not differentiate between platform versions.

All the frames for which eyeGazeCorrection happens, originate from cameras.
No methods are provided for sites to insert frames for eyeGazeCorrection.
As such, sites cannot fingerprint the eyeGazeCorrection implementation as the sites have no access to original frames and have access to frames with feye gaze corrected only if the user has the user has [granted](https://w3c.github.io/permissions/#dfn-granted) the ["camera"](https://www.w3.org/TR/mediacapture-streams/#dfn-camera) permission.

## Stakeholder Feedback / Opposition

[Implementors and other stakeholders may already have publicly stated positions on this work. If you can, list them here with links to evidence as appropriate.]

[If appropriate, explain the reasons given by other implementors for their concerns.]

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- [Bernard Aboba]
- [Harald Alvestrand]
- [Jan-Ivar Bruaroey]
- [Dominique Hazael-Massieux]
- [François Beaufort]


## Disclaimer

Intel is committed to respecting human rights and avoiding complicity in human rights abuses. See Intel's Global Human Rights Principles. Intel's products and software are intended only to be used in applications that do not cause or contribute to a violation of an internationally recognized human right.

Intel technologies may require enabled hardware, software or service activation.

No product or component can be absolutely secure.

Your costs and results may vary.

© Intel Corporation
