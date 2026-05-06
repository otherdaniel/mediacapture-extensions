# Media Capture Elements: `<camera>`, `<microphone>`, and `<usermedia>`

## Authors:

- Daniel Vogelheim (Google LLC)
- Minh Le (Google LLC)
- Thomas Nguyen (Google LLC)

## Introduction

The web permission model is shifting from passive permission control to active Capability Elements. Building on the foundations of the Page-Embedded Permission Control (PEPC) initiative, these specialized Capability elements evolve the user experience from a simple "Allow/Deny" toggle into a direct controller for web capabilities.

Rather than serving as a static status indicator, a Capability Element acts as a functional action broker. It streamlines the entire lifecycle: establishing a trusted signal of intent, managing the permission flow, and delivering the resulting data or media stream directly to the application. This shift reduces developer friction, eliminates "permission holes," and provides users with a consistent, browser-controlled interface for hardware like cameras and microphones.

This document proposes a suite of targeted Media Capture Elements (`<camera>`, `<microphone>`, and `<usermedia>`). They move beyond acting merely as permission gatekeepers to serving as data mediators: handling consent, fetching the `MediaStream`, delivering it to the site, and persisting as a stateful, trusted UI control for toggling media tracks.

## Goals

*   Data Mediator and Action Broker: The elements handle the entire acquisition flow: obtaining user consent, executing the underlying `getUserMedia` request, and exposing the resulting `MediaStream` directly, often eliminating the need for separate JavaScript API boilerplate.
*   Stateful Media Capture Control: Elements act as integrated UI components. Following a permission grant, the element transitions from a _request_ state to a _control_ state. For `<camera>` and `<microphone>`, this provides a browser-enforced, UI toggle for disabling and enabling the underlying track. All elements reflect their track's enabled state, and revert to the _request_ state once the track has ended.
*   Reduced Friction & Trusted Intent: Establish a "trusted signal of intent" driven by explicit User Gestures on a browser-controlled element, allowing developers to offer users the option to use the capability in context without having to deal with permission.
*   Seamless In-Page Recovery: Similar to and inheriting concepts from the `<permission>` element, this resolves the "permission hole" by providing an in-page flow to re-enable access if the user previously blocked it in the user agent, avoiding the need for users to navigate complex, UA-specific browser settings.
*   Specific vs. Generic Tags: Replace the generic `<permission>` element with targeted, semantic elements (`<camera>`, `<microphone>`, `<usermedia>`). This allows for UA-tailored UI, specific IDL attributes, and distinct security/privacy models tailored to the specific hardware media capture capability.

## Non-goals

*   Replacing `getUserMedia`: The `mediaDevices.getUserMedia` API remains the primary programmatic primitive. Media Capture Elements are designed to complement it by handling common UI-driven flows.
*   The `permissions` API will remain to support the use of `getUserMedia` alongside the new elements.
*   Complete UI Customization: The shadow DOM/internal UI of these elements is strictly controlled by the UA to prevent clickjacking and ensure a trusted state representation. Complete visual overriding by the site is a non-goal for security reasons.

## The `<camera>` HTML element

The `<camera>` element provides a dedicated UI for requesting and controlling access to a camera's video track. It renders as a camera text button or icon. After a successful grant, the element transitions to a trusted UI toggle for disabling and enabling the video track.



### Attributes and Properties

*   `track`: A read-only property that holds the associated `MediaStreamTrack` object (video). It is populated automatically upon successful user interaction.
*   `autostart`: A boolean attribute. If present, the UA will attempt to start the stream automatically as the element is added to the DOM and rendered (provided permissions are already granted).
*   `onstream`: An `EventHandler` that fires when a hardware acquisition attempt completes successfully and populates the `track` property.
*   `onerror`: An `EventHandler` that fires when a stream acquisition attempt fails. To allow the element to carry detailed diagnostic info without bloating the main element interface, it is proposed that this event object carries a dedicated error payload containing, for example, a DOMException along with optional platform-specific diagnostic parameters.

### Constraints Configuration

Developers can specify a `MediaTrackConstraintSet` to dictate the parameters of the underlying `getUserMedia` call. This is done imperatively via the `setConstraints()` method. If constraints are not set before the user interacts with the element, an empty default (`{}`) is used.

**Constraint Filtering**: Sets the active constraints for the underlying camera track. Note that advanced and required constraints are ignored by the User Agent to prevent the element from failing silently with an `OverconstrainedError`.

### Example

```javascript
const cameraEl = document.querySelector('camera');
const videoEl = document.querySelector('video');

// Configure high-resolution constraints directly
cameraEl.setConstraints({ width: 1920, height: 1080 });

// Handle successful stream acquisition
cameraEl.addEventListener("stream", () => {
  const stream = new MediaStream([cameraEl.track]);
  videoEl.srcObject = stream;
});

// Handle stream acquisition failure with detailed error event diagnostics
cameraEl.addEventListener("error", (event) => {
  const exception = event.detail?.exception; // Proposed error dictionary payload
  console.error(`Camera acquisition failed: ${exception.name}`);
});
```

## The `<microphone>` HTML element

The `<microphone>` element provides a dedicated UI for requesting and controlling access to a microphone's audio track. It renders as a microphone text button or icon. After a successful grant, the element transitions to a trusted UI toggle for muting and unmuting the audio stream.



### Attributes and Properties

*   `track`: A read-only property that holds the associated `MediaStreamTrack` object (audio). It is populated automatically upon successful user interaction.
*   `autostart`: A boolean attribute. If present, the UA will attempt to start the stream automatically as the element is added to the DOM and rendered (provided permissions are already granted).
*   `onstream`: An `EventHandler` that fires when a hardware acquisition attempt completes successfully and populates the `track` property.
*   `onerror`: An `EventHandler` that fires when a stream acquisition attempt fails. To allow the element to carry detailed diagnostic info without bloating the main element interface, it is proposed that this event object carries a dedicated error payload containing, for example, a DOMException along with optional platform-specific diagnostic parameters.

### Constraints Configuration

Developers can specify a `MediaTrackConstraintSet` to dictate the parameters of the underlying `getUserMedia` call. This is done imperatively via the `setConstraints()` method. If constraints are not set before the user interacts with the element, an empty default (`{}`) is used.

**Constraint Filtering**: Sets the active constraints for the underlying microphone track. Note that advanced and required constraints are ignored by the User Agent to prevent the element from failing silently with an `OverconstrainedError`.

### Example

```javascript
const micEl = document.querySelector('microphone');

// Configure echo cancellation and noise suppression
micEl.setConstraints({ echoCancellation: true, noiseSuppression: true });

// Handle successful microphone track acquisition
micEl.addEventListener("stream", () => {
  console.log("Audio track successfully acquired:", micEl.track);
});

// Handle acquisition failure
micEl.addEventListener("error", (event) => {
  console.error("Audio acquisition failed:", event.detail?.exception.name);
});
```

## The `<usermedia>` HTML element

The `<usermedia>` element controls combined audio and video capture. Unlike `<camera>` and `<microphone>`, it does not natively transition into a per-track mute/unmute control upon grant due to the complexity of managing potentially independent active/ended states for multiple audio and video tracks. It primarily functions as a unified access broker.



### Attributes and Properties

*   `stream`: A read-only property that holds the associated `MediaStream` object. It is populated automatically upon successful user interaction.
*   `autostart`: A boolean attribute. If present, the UA will attempt to start the stream automatically as the element is added to the DOM and rendered (provided permissions are already granted).
*   `onstream`: An `EventHandler` that fires when a hardware acquisition attempt completes successfully and populates the `stream` property.
*   `onerror`: An `EventHandler` that fires when a stream acquisition attempt fails. To allow the element to carry detailed diagnostic info without bloating the main element interface, it is proposed that this event object carries a dedicated error payload containing, for example, a DOMException along with optional platform-specific diagnostic parameters.

### Constraints Configuration

Developers can specify `HTMLMediaStreamConstraints` to dictate the parameters of the underlying `getUserMedia` call. This is done imperatively via the `setConstraints()` method. If constraints are not set before the user interacts with the element, a secure default (`{audio:{}, video:{}}`) is used.

**Constraint Filtering**: Sets the active constraints for both the underlying audio and video tracks. Note that advanced and required constraints are ignored by the User Agent to prevent the element from failing silently with an `OverconstrainedError`.

### Example

```javascript
const umElement = document.querySelector('usermedia');
const videoEl = document.querySelector('video');

// Configure constraints for combined audio/video capture
umElement.setConstraints({
  video: { width: 1280 },
  audio: {}
});

// Handle successful stream acquisition
umElement.addEventListener("stream", () => {
  videoEl.srcObject = umElement.stream;
});

// Handle stream acquisition failure
umElement.addEventListener("error", (event) => {
  console.error("Combined stream acquisition failed:", event.detail?.exception.name);
});
```

## User Journey & Interaction Model

1.  Request State (Inactive): An element with no active associated `MediaStream` is in a request state. A User Gesture (click) triggers a permission prompt (if not already granted). Alternatively, if the `autostart` attribute is present when the element is rendered in the DOM, the UA will attempt to bypass the click requirement and automatically start the stream (provided permissions are already granted).
2.  Acquisition: Upon grant, the UA implicitly executes a `getUserMedia` request using the element's configured constraints. The resulting track or stream is assigned to the `track` or `stream` property. If acquisition succeeds, the element fires the `stream` event and begins monitoring the resulting `MediaStreamTrack`s. If acquisition fails, the element fires the `error` event carrying detailed payload diagnostics.
3.  Control State (Active for `<camera>` and `<microphone>` only): The element updates its internal UI (e.g., from "Use Camera" to "Mute Camera"). Subsequent clicks on `<camera>` or `<microphone>` elements will trigger the element's secondary activation steps, natively toggling the `enabled` property of all associated `MediaStreamTrack`s. The `<usermedia>` element skips this second-click behavior due to the complexity of managing potentially independent active/ended states for multiple audio and video tracks.
4.  Track Monitoring: The element acts as an active monitor for its internal tracks or streams, observing their `enabled`, `muted`, and `ended` states (including manual `track.stop()` calls) to ensure its UI remains securely synchronized with the underlying hardware state. Any state changes will update the element's UI. If all tracks terminate, the element resets to its initial request state, and a subsequent click will fetch a new stream.

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

