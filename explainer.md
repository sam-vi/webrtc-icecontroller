## Summary

Peer-to-peer connections are used in a variety of applications such as audio/video conferencing, live broadcasts, cloud gaming, screen sharing, and many more. This relies on the Interactive Connectivity Establishment (ICE) protocol to discover, evaluate, and use network paths suitable for communication between the peers. When an application creates a peer-to-peer connection using `RTCPeerConnection`, the user agent performs ICE to manage the connection, with limited input feedback from the application.

`RTCIceController` will allow an application to observe the user agent performing ICE in greater detail than before, and take over some aspects of ICE. This will permit the application to have more control over the network path used for a peer-to-peer connection.

## Motivation

- Improve connection reliability for users by allowing the application to use its knowledge of the exact usage scenario to better evaluate trade-offs and manage the network connection actively.
- Allow applications to experiment with what works for their specific use case independent of the standards to improve the speed of innovation and iteration.

## Example

```javascript
const iceController = new RTCIceController();

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
  makeNoteOfPruneProposal(event.getCandidatePairs());
}, {passive: true});

// Check connectivity of various candidate pairs.
function doPing() {
  iceController.sendIcePing(selectBestPairToPing());
  setTimeout(() => doPing(), getPingRecheckInterval());
}
iceController.addEventListener('candidatepairadded', () => {
  doPing();
}, {once: true});

// Select the most suitable candidate pair and use that for transport. Prune
// any unneeded candidate pairs.
function doSelectAndPrune() {
  iceController.switchToCandidatePair(selectBestPairToUse());
  iceController.pruneCandidatePairs(selectPairsToPrune());
  setTimeout(() => doSelectAndPrune(), getSelectRecheckInterval());
}
iceController.addEventListener('candidatepairupdated', () => {
  doSelectAndPrune();
}, {once: true});

// If the selected candidate pair is worsening, increase ping and select
// intervals more frequently.
iceController.addEventListener('candidatepairupdated', event => {
  if (event.candidatePair == iceController.getSelectedCandidatePair()) {
    adjustRecheckIntervals(event.report);
  }
});

// Set the ICE controller in RTCConfiguration.
const configuration = {
  iceController: iceController
};
const pc = new RTCPeerConnection(configuration);
```

## Relevant Links

- Public API draft: https://sam-vi.github.io/webrtc-icecontroller/
