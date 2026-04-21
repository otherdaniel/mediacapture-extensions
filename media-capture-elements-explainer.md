# Media Capture Elements: `<usermedia>`, `<camera>`, and `<microphone>`

## Authors:

- Daniel Vogelheim (Google LLC)
- Minh Le (Google LLC)
- Thomas Nguyen (Google LLC)

## Introduction

The web permission model is shifting from passive permission control to active Capability Elements. Building on the foundations of the Page-Embedded Permission Control (PEPC) initiative, these specialized Capability elements evolve the user experience from a simple "Allow/Deny" toggle into a direct controller for web capabilities.

Rather than serving as a static status indicator, a Capability Element acts as a functional action broker. It streamlines the entire lifecycle: establishing a trusted signal of intent, managing the permission flow, and delivering the resulting data or media stream directly to the application. This shift reduces developer friction, eliminates "permission holes," and provides users with a consistent, browser-controlled interface for hardware like cameras and microphones.

This document proposes a suite of targeted Media Capture Elements (`<usermedia>`, `<camera>`, and `<microphone>`). They move beyond acting merely as permission gatekeepers to serving as data mediators: handling consent, fetching the `MediaStream`, delivering it to the site, and persisting as a stateful, trusted UI control for toggling media tracks.

## Goals

*   Data Mediator and Action Broker: The elements handle the entire acquisition flow: obtaining user consent, executing the underlying `getUserMedia` request, and exposing the resulting `MediaStream` directly, often eliminating the need for separate JavaScript API boilerplate.
*   Stateful Media Capture Control: Elements act as integrated UI components. Following a permission grant, the element transitions from a "request" state to a "control" state. For `<camera>` and `<microphone>`, this provides a browser-enforced, trusted UI toggle for muting and unmuting the stream. All elements act as strict monitors of their assigned track's states, natively reverting to the "request" state if the underlying tracks are fully terminated.
*   Reduced Friction & Trusted Intent: Establish a "trusted signal of intent" driven by explicit User Gestures on a browser-controlled element, allowing developers to offer users an option to use the grant and use the capability in context.
*   Seamless In-Page Recovery: Similar to and inheriting concepts from the `<permission>` element, this resolves the "permission hole" by providing an in-page flow to re-enable access if the user previously denied it, avoiding the need for users to navigate complex, UA-specific browser settings.
*   Specific vs. Generic Tags: Replace the generic `<permission>` element with targeted, semantic elements (`<camera>`, `<microphone>`, `<usermedia>`). This allows for UA-tailored UI, specific IDL attributes, and distinct security/privacy models tailored to the specific hardware media capture capability.

## Non-goals

*   Replacing `getUserMedia`: The `mediaDevices.getUserMedia` API remains the primary programmatic primitive. Media Capture Elements are designed to complement it by handling common UI-driven flows. In the future, we may add a parameter to indicate whether the `getUserMedia` request originated from a programmatic JS call or directly from a media capture element.
*   Complete UI Customization: The shadow DOM/internal UI of these elements is strictly controlled by the UA to prevent clickjacking and ensure a trusted state representation. Complete visual overriding by the site is a non-goal for security reasons.

## The `<camera>` HTML element

The `<camera>` element provides a dedicated UI for requesting and controlling access to a camera's video track. It renders as a camera text button or icon. After a successful grant, the element transitions to a trusted UI toggle for muting and unmuting the video stream.

### WebIDL

```webidl
[Exposed=Window]
interface HTMLCameraElement : HTMLElement {
  [HTMLConstructor] constructor();

  readonly attribute MediaStreamTrack? track;
  undefined setConstraints(optional MediaTrackConstraintSet constraints = {});
  readonly attribute any error;

  attribute boolean autostart;

  attribute EventHandler onstream;
  attribute EventHandler ontrackchange;
};
HTMLCameraElement includes ActivationBlockersMixin;
```

### Attributes and Properties

*   `track`: A read-only property that holds the associated `MediaStreamTrack` object (video). It is populated automatically upon successful user interaction.
*   `autostart`: A boolean attribute. If present, the UA will attempt to start the stream automatically as the element is added to the DOM and rendered (provided permissions are already granted).
*   `error`: A read-only property that holds a `DOMException` or `null`. Populated if the stream acquisition fails.
*   `onstream`: An `EventHandler` that fires when a `getUserMedia` attempt finishes (either populating `track` or `error`).
*   `ontrackchange`: An `EventHandler` that fires when the associated track's active/muted/ended state changes.
*   `isValid`, `invalidReason`, `onvalidationstatuschange`: Provided by the `ActivationBlockersMixin` to protect against programmatic activation and clickjacking.

### Constraints Configuration

Developers can specify a `MediaTrackConstraintSet` to dictate the parameters of the underlying `getUserMedia` call. This is done imperatively via the `setConstraints()` method. If constraints are not set before the user interacts with the element, an empty default (`{}`) is used.

**Constraint Filtering**: The UA applies a "constraint filter" before executing the `getUserMedia` call. Because the element's `setConstraints()` takes a `MediaTrackConstraintSet` directly (which contains no `audio` or `video` keys), the UA constructs a `MediaStreamConstraints` object where `audio` is forced to `false`, and `video` is set to the provided `MediaTrackConstraintSet`. Additionally, any required constraints (like `exact`, `min`, or `max`) are stripped to prevent the element from failing silently with an `OverconstrainedError` when the user interacts with it.

### Example

```html
<!-- A camera element that will be configured via JavaScript -->
<camera id="my-camera"></camera>
<video id="camera-playback" autoplay playsinline></video>

<script>
  const cameraEl = document.getElementById('my-camera');
  const videoEl = document.getElementById('camera-playback');

  // The developer configures high-resolution video constraints for the camera element
  cameraEl.setConstraints({
    width: { ideal: 1920 }, height: { ideal: 1080 }
  });

  // Listen for the stream event which fires when acquisition finishes
  cameraEl.addEventListener("stream", () => {
    if (cameraEl.track) {
      // Create a MediaStream from the track to attach to the video element
      const stream = new MediaStream([cameraEl.track]);
      videoEl.srcObject = stream;
    } else if (cameraEl.error) {
      console.error("Stream acquisition failed:", cameraEl.error);
    }
  });

  // Observe the trackchange event to react to mute/unmute state changes
  cameraEl.addEventListener("trackchange", () => {
    console.log("Track active/muted/ended state changed");
  });
</script>
```

## The `<microphone>` HTML element

The `<microphone>` element provides a dedicated UI for requesting and controlling access to a microphone's audio track. It renders as a microphone text button or icon. After a successful grant, the element transitions to a trusted UI toggle for muting and unmuting the audio stream.

### WebIDL

```webidl
[Exposed=Window]
interface HTMLMicrophoneElement : HTMLElement {
  [HTMLConstructor] constructor();

  readonly attribute MediaStreamTrack? track;
  undefined setConstraints(optional MediaTrackConstraintSet constraints = {});
  readonly attribute any error;

  attribute boolean autostart;

  attribute EventHandler onstream;
  attribute EventHandler ontrackchange;
};
HTMLMicrophoneElement includes ActivationBlockersMixin;
```

### Attributes and Properties

*   `track`: A read-only property that holds the associated `MediaStreamTrack` object (audio). It is populated automatically upon successful user interaction.
*   `autostart`: A boolean attribute. If present, the UA will attempt to start the stream automatically as the element is added to the DOM and rendered (provided permissions are already granted).
*   `error`: A read-only property that holds a `DOMException` or `null`. Populated if the stream acquisition fails.
*   `onstream`: An `EventHandler` that fires when a `getUserMedia` attempt finishes (either populating `track` or `error`).
*   `ontrackchange`: An `EventHandler` that fires when the associated track's active/muted/ended state changes.
*   `isValid`, `invalidReason`, `onvalidationstatuschange`: Provided by the `ActivationBlockersMixin` to protect against programmatic activation and clickjacking.

### Constraints Configuration

Developers can specify a `MediaTrackConstraintSet` to dictate the parameters of the underlying `getUserMedia` call. This is done imperatively via the `setConstraints()` method. If constraints are not set before the user interacts with the element, an empty default (`{}`) is used.

**Constraint Filtering**: The UA applies a "constraint filter" before executing the `getUserMedia` call. Because the element's `setConstraints()` takes a `MediaTrackConstraintSet` directly (which contains no `audio` or `video` keys), the UA constructs a `MediaStreamConstraints` object where `video` is forced to `false`, and `audio` is set to the provided `MediaTrackConstraintSet`. Additionally, any required constraints are stripped to prevent the element from failing silently with an `OverconstrainedError`.

### Example

```html
<!-- A microphone element that will be configured via JavaScript -->
<microphone id="my-microphone"></microphone>

<script>
  const micEl = document.getElementById('my-microphone');

  // The developer configures audio constraints for the microphone element
  micEl.setConstraints({
    echoCancellation: true,
    noiseSuppression: true
  });

  micEl.addEventListener("stream", () => {
    if (micEl.track) {
      console.log("Audio track acquired!");
    } else if (micEl.error) {
      console.error("Audio acquisition failed:", micEl.error);
    }
  });
</script>
```

## The `<usermedia>` HTML element

The `<usermedia>` element controls combined audio and video capture. Unlike `<camera>` and `<microphone>`, it does not natively transition into a per-track mute/unmute control upon grant due to the complexity of managing potentially independent active/ended states for multiple audio and video tracks. It primarily functions as a unified access broker.

### WebIDL

```webidl
dictionary HTMLMediaStreamConstraints {
  MediaTrackConstraintSet video;
  MediaTrackConstraintSet audio;
};

[Exposed=Window]
interface HTMLUserMediaElement : HTMLElement {
  [HTMLConstructor] constructor();

  readonly attribute MediaStream? stream;
  undefined setConstraints(optional HTMLMediaStreamConstraints constraints = {});
  readonly attribute any error;

  attribute boolean autostart;

  attribute EventHandler onstream;
  attribute EventHandler ontrackchange;
};
HTMLUserMediaElement includes ActivationBlockersMixin;
```

### Attributes and Properties

*   `stream`: A read-only property that holds the associated `MediaStream` object. It is populated automatically upon successful user interaction.
*   `autostart`: A boolean attribute. If present, the UA will attempt to start the stream automatically as the element is added to the DOM and rendered (provided permissions are already granted).
*   `error`: A read-only property that holds a `DOMException` or `null`. Populated if the stream acquisition fails.
*   `onstream`: An `EventHandler` that fires when a `getUserMedia` attempt finishes (either populating `stream` or `error`).
*   `ontrackchange`: An `EventHandler` that fires when the associated tracks' active/muted/ended states change.
*   `isValid`, `invalidReason`, `onvalidationstatuschange`: Provided by the `ActivationBlockersMixin` to protect against programmatic activation and clickjacking.

### Constraints Configuration

Developers can specify `HTMLMediaStreamConstraints` to dictate the parameters of the underlying `getUserMedia` call. This is done imperatively via the `setConstraints()` method. If constraints are not set before the user interacts with the element, a secure default (`{audio:{}, video:{}}`) is used.

**Constraint Filtering**: The UA applies a "constraint filter" before executing the `getUserMedia` call. For `<usermedia>`, the UA constructs a `MediaStreamConstraints` object by extracting the `audio` and `video` `MediaTrackConstraintSet`s from the provided `HTMLMediaStreamConstraints`. If either is omitted, an empty constraint set (`{}`) is substituted, ensuring both audio and video are always requested. Additionally, any required constraints are stripped to prevent silent `OverconstrainedError` failures.

### Example

```html
<!-- A usermedia element that will be configured via JavaScript -->
<usermedia id="my-usermedia"></usermedia>
<video id="stream-playback" autoplay playsinline></video>

<script>
  const umElement = document.getElementById("my-usermedia");
  const videoEl = document.getElementById("stream-playback");

  // Configure specific video bounds using setConstraints()
  umElement.setConstraints({
    video: { width: { ideal: 1280 } },
    audio: {}
  });

  // Listen for the stream event which fires when acquisition finishes
  umElement.addEventListener("stream", () => {
    if (umElement.stream) {
      videoEl.srcObject = umElement.stream;
    } else if (umElement.error) {
      console.error("Stream acquisition failed:", umElement.error);
    }
  });

  // Observe the trackchange event to react to mute/unmute state changes
  umElement.addEventListener("trackchange", () => {
    console.log("Track active/muted/ended state changed");
  });
</script>
```

## User Journey & Interaction Model

1.  Request State (Inactive): An element with no active associated `MediaStream` is in a request state. A User Gesture (click) triggers a permission prompt (if not already granted). Alternatively, if the `autostart` attribute is present when the element is rendered in the DOM, the UA will attempt to bypass the click requirement and automatically start the stream (provided permissions are already granted).
2.  Acquisition: Upon grant, the UA implicitly executes a `getUserMedia` request using the element's configured constraints. The resulting `MediaStream` is assigned to the `stream` property. If acquisition fails, the `error` property is populated. The element fires the `stream` event and begins monitoring the resulting `MediaStreamTrack`s.
3.  Control State (Active for `<camera>` and `<microphone>` only): The element updates its internal UI (e.g., from "Use Camera" to "Mute Camera"). Subsequent clicks on `<camera>` or `<microphone>` elements will trigger the element's secondary activation steps, natively toggling the `enabled` property of all associated `MediaStreamTrack`s. The `<usermedia>` element skips this second-click behavior due to the complexity of managing potentially independent active/ended states for multiple audio and video tracks.
4.  Track Monitoring: The element acts as an active monitor for its internal `stream`, observing the `enabled`, `muted`, and `ended` states (including manual `track.stop()` calls) of the associated tracks to ensure its UI remains securely synchronized with the underlying hardware state. Any state changes will update the element's UI and fire the `trackchange` event. If all tracks terminate (i.e. the stream becomes inactive), the element resets to its initial request state, and a subsequent click will fetch a new stream.

## Key Scenarios

*   Contextual Permission Recovery: A user who previously denied microphone access sees a `<microphone>` element with a warning badge. Clicking it opens a UA-provided dialogue explaining the blocked state and allowing immediate recovery within the page context, without digging through browser settings.
*   Seamless Re-entry (Autostart): A user accesses a known meeting room where they previously granted camera and microphone access. The web page renders `<usermedia autostart></usermedia>`. The UA immediately acquires the stream and fires the `stream` event without waiting for the user to click the element, allowing immediate video playback.
*   Conferencing entry: A user clicks a `<usermedia>` button in a meeting lobby. The browser handles the prompt and the stream acquisition. The developer observes the `stream` property and attaches it to a `<video>` tag, all without needing to write complex `getUserMedia` logic.

## Privacy & Security Considerations

*   Implicit Stream Activation: Unlike the legacy `<permission>` element which merely gated access, Media Capture Elements automatically trigger device hardware and indicator lights (e.g., the camera LED) upon grant. Future iterations of this specification may need to introduce additional features to support initializing streams in an explicitly disabled state to mitigate privacy concerns when immediate activation is unexpected. The exact feasibility and shape of such additions are currently under investigation. 
*   Constraint Fingerprinting: Allowing sites to pass highly specific `MediaStreamConstraints` via the element exposes the same device fingerprinting vectors as the raw `getUserMedia` API. UAs must apply the same fingerprinting mitigations (e.g., fuzzying exact constraints, restricting access to `deviceId` until permission is granted) to the declarative constraints as they do to imperative JS calls.
*   Trusted UI vs. Clickjacking: Because these elements control sensitive hardware states (mute/unmute), their rendering must be strictly limited from the host page's CSS layout interference to prevent clickjacking or "fake element" attacks. The elements must be implemented using secure, closed UA Shadow DOMs, and pointer events must be rigorously validated as trusted User Gestures.

## Stakeholder Feedback / Opposition

- Chromium: Positive. Implementing the MVP; previous Origin Trials showed significant improvements for user experience and the permission granting flow.
- Mozilla and WebKit (Apple): Engaged in discussions. Feedback provided via standard positions ([#1245](https://github.com/mozilla/standards-positions/issues/1245)) and WICG issues ([#62](https://github.com/WICG/PEPC/issues/62)).

