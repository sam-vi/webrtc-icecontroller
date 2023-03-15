## Summary

Peer-to-peer connections are used in a variety of applications such as audio/video conferencing, live broadcasts, cloud gaming, screen sharing, and many more. This relies on the Interactive Connectivity Establishment (ICE) protocol to discover, evaluate, and use network paths suitable for communication between the peers. When an application creates a peer-to-peer connection using `RTCPeerConnection`, the user agent performs ICE to manage the connection, with limited input feedback from the application.

`RTCIceController` will allow an application to observe the user agent performing ICE in greater detail than before, and take over some aspects of ICE. This will permit the application to have more control over the network path used for a peer-to-peer connection.

## Motivation

- Improve connection reliability for users by allowing the application to use its knowledge of the exact usage scenario to better evaluate trade-offs and manage the network connection actively.
- Allow applications to experiment with what works for their specific use case independent of the standards to improve the speed of innovation and iteration.

## Example

```javascript
const iceController = new RTCIceController();

// Application implements these methods for candidate pair selection.
function selectBestPairToPing() { /* TODO... */ }
function selectBestPairToUse() { /* TODO... */ }
function selectPairsToPrune() { /* TODO... */ }

// Application implements these methods to determine if a candidate pair
// ping or switch should be allowed to happen.
function shouldPingPair(pair) { /* TODO... */ }
function shouldSwitchToPair(pair) { /* TODO... */ }

// Initial intervals for sending periodic connectivity checks and picking
// the most suitable candidate pair for transport.
let pingRecheckIntervalMs = 2.5 * 1000;
let switchRecheckIntervalMs = 5 * 1000;

// Observe events that indicate when the browser is about to perform an ICE
// action. Cancel the action by calling preventDefault if the application has
// other plans.
iceController.addEventListener('icepingproposed', event => {
  if (!shouldPingPair(event.candidatePair)) {
    event.preventDefault();
  }
});
iceController.addEventListener('iceswitchproposed', event => {
  if (event.cancelable &&
      !shouldSwitchToPair(event.reason, event.candidatePair)) {
    event.preventDefault();
  }
});

// If the listener will not call preventDefault, it can be marked passive to
// allow optimizations.
iceController.addEventListener('icepruneproposed', event => {
  console.log(JSON.stringify(event.getCandidatePairs()));
}, {passive: true});

// Check connectivity of various candidate pairs.
function doPing() {
  iceController.sendIcePing(selectBestPairToPing());
  setTimeout(() => doPing(), pingRecheckIntervalMs);
}
iceController.addEventListener('candidatepairadded', () => {
  doPing();
}, {once: true});

// Select the most suitable candidate pair and use that for transport. Prune
// any unneeded candidate pairs.
function doSelectAndPrune() {
  iceController.switchToCandidatePair(selectBestPairToUse());
  iceController.pruneCandidatePairs(selectPairsToPrune());
  setTimeout(() => doSelectAndPrune(), switchRecheckIntervalMs);
}
iceController.addEventListener('candidatepairupdated', () => {
  doSelectAndPrune();
}, {once: true});

// Clamp a number between lower and upper bounds.
function clamp(num, min, max) {
  return Math.min(Math.max(num, min), max);
}

// Adjust the recheck intervals based on RTT measurements.
function adjustRecheckIntervals(report) {
  const rtts = report.getRoundTripTimeSamples().map(s => s.value);
  const rtt_p75 = quantile(rtts, 0.75);

  const MIN_PING_RECHECK_INTERVAL_MS = 500;
  const MAX_PING_RECHECK_INTERVAL_MS = 10 * 1000;
  const MIN_SWITCH_RECHECK_INTERVAL_MS = 1000;
  const MAX_SWITCH_RECHECK_INTERVAL_MS = 30 * 1000;

  const LOW_RTT_THRESHOLD_MS = 250;
  const HIGH_RTT_THRESHOLD_MS = 500;

  if (rtt_p75 < LOW_RTT_THRESHOLD_MS) {
    // Selected candidate pair has good RTT, recheck less frequently.
    pingRecheckIntervalMs *= 2;
    switchRecheckIntervalMs *= 2;
  }
  else if (rtt_p75 > HIGH_RTT_THRESHOLD_MS) {
    // Selected candidate pair has poor RTT, recheck more frequently.
    pingRecheckIntervalMs = Math.floor(pingRecheckIntervalMs / 2);
    switchRecheckIntervalMs = Math.floor(switchRecheckIntervalMs / 2);
  }

  pingRecheckIntervalMs = clamp(pingRecheckIntervalMs,
    MIN_PING_RECHECK_INTERVAL_MS, MAX_PING_RECHECK_INTERVAL_MS);
  switchRecheckIntervalMs = clamp(switchRecheckIntervalMs,
    MIN_SWITCH_RECHECK_INTERVAL_MS, MAX_SWITCH_RECHECK_INTERVAL_MS);
}

// If the selected candidate pair is worsening, increase ping and select
// intervals more frequently. Otherwise decrease the recheck frequency.
iceController.addEventListener('candidatepairupdated', event => {
  if (event.candidatePair == iceController.getSelectedCandidatePair()) {
    adjustRecheckIntervals(event.report);
  }
});

// Set the ICE controller in RTCConfiguration.
const pc = new RTCPeerConnection({ iceController: iceController });
```

## Relevant Links

- Public API draft: https://sam-vi.github.io/webrtc-icecontroller/
