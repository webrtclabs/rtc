# rtc

The `rtc` module does most of the heavy lifting within the
[rtc.io](http://rtc.io) suite.  Primarily it handles the logic of coupling
a local `RTCPeerConnection` with it's remote counterpart via an
[rtc-signaller](https://github.com/rtc-io/rtc-signaller) signalling
channel.

In most cases, it is recommended that you use one of the higher-level
modules that uses the `rtc` module under the hood.  Such as:

- [rtc-quickconnect](https://github.com/rtc-io/rtc-quickconnect)
- [rtc-glue](https://github.com/rtc-io/rtc-glue)


[![NPM](https://nodei.co/npm/rtc.png)](https://nodei.co/npm/rtc/)

[![unstable](http://hughsk.github.io/stability-badges/dist/unstable.svg)](http://github.com/hughsk/stability-badges)

## Getting Started

If you decide that the `rtc` module is a better fit for you than either
[rtc-quickconnect](https://github.com/rtc-io/rtc-quickconnect) or
[rtc-glue](https://github.com/rtc-io/rtc-glue) then the code snippet below
will provide you a guide on how to get started using it in conjunction with
the [rtc-signaller](https://github.com/rtc-io/rtc-signaller) and
[rtc-media](https://github.com/rtc-io/rtc-media) modules:

```js
var signaller = require('rtc-signaller')('http://rtc.io/switchboard/');
var rtc = require('rtc');
var media = require('rtc-media');
var localMedia = media();

// render the local media to the document body
localMedia.render(document.body);

// capture local media first as firefox
// will want a local stream and doesn't support onnegotiationneeded event
localMedia.once('capture', function(localStream) {
  // look for friends
  signaller.on('peer:announce', function(data) {
    // create a peer connection for our new friend
    var pc = rtc.createConnection();

    // couple our connection via the signalling channel
    var monitor = rtc.couple(pc, data.id, signaller);

    // add the stream to the connection
    pc.addStream(localStream);

    // once the connection is active, log a console message
    monitor.once('active', function() {
      console.log('connection active to: ' + data.id);
    });

    // when the peer connection receives a remote stream render it to the
    // screen
    pc.onaddstream = function(remoteStream) {
      media(remoteStream).render(document.body);
    };
  });

  // announce ourself in the rtc-getting-started room
  signaller.announce({ room: 'rtc-getting-started' });
});


```

This code definitely doesn't cover all the cases that you need to consider
(i.e. peers leaving, etc) but it should demonstrate how to:

1. Capture video and add it to a peer connection
2. Couple a local peer connection with a remote peer connection
3. Deal with the remote steam being discovered and how to render
   that to the local interface.

## Factories

### createConnection(opts?, constraints?)

Create a new `RTCPeerConnection` auto generating default opts as required.

```js
var conn;

// this is ok
conn = rtc.createConnection();

// and so is this
conn = rtc.createConnection({
  iceServers: []
});
```

## rtc/couple

### couple(pc, targetId, signaller, opts?)

Couple a WebRTC connection with another webrtc connection identified by
`targetId` via the signaller.

The following options can be provided in the `opts` argument:

- `sdpfilter` (default: null)

  A simple function for filtering SDP as part of the peer
  connection handshake (see the Using Filters details below).

- `maxAttempts` (default: 1)

  How many times should negotiation be attempted.  This is
  **experimental** functionality for attempting connection negotiation
  if it fails.

- `attemptDelay` (default: 3000)

  The amount of ms to wait between connection negotiation attempts.

#### Example Usage

```js
var couple = require('rtc/couple');

couple(pc, '54879965-ce43-426e-a8ef-09ac1e39a16d', signaller);
```

#### Using Filters

In certain instances you may wish to modify the raw SDP that is provided
by the `createOffer` and `createAnswer` calls.  This can be done by passing
a `sdpfilter` function (or array) in the options.  For example:

```js
// run the sdp from through a local tweakSdp function.
couple(pc, '54879965-ce43-426e-a8ef-09ac1e39a16d', signaller, {
  sdpfilter: tweakSdp
});
```

## rtc/detect

Provide the [rtc-core/detect](https://github.com/rtc-io/rtc-core#detect) 
functionality.

## rtc/generators

The generators package provides some utility methods for generating
constraint objects and similar constructs.

```js
var generators = require('rtc/generators');
```

### generators.config(config)

Generate a configuration object suitable for passing into an W3C
RTCPeerConnection constructor first argument, based on our custom config.

### generators.connectionConstraints(flags, constraints)

This is a helper function that will generate appropriate connection
constraints for a new `RTCPeerConnection` object which is constructed
in the following way:

```js
var conn = new RTCPeerConnection(flags, constraints);
```

In most cases the constraints object can be left empty, but when creating
data channels some additional options are required.  This function
can generate those additional options and intelligently combine any
user defined constraints (in `constraints`) with shorthand flags that
might be passed while using the `rtc.createConnection` helper.

### parseFlags(opts)

This is a helper function that will extract known flags from a generic
options object.

## rtc/monitor

In most current implementations of `RTCPeerConnection` it is quite
difficult to determine whether a peer connection is active and ready
for use or not.  The monitor provides some assistance here by providing
a simple function that provides an `EventEmitter` which gives updates
on a connections state.

### monitor(pc) -> EventEmitter

```js
var monitor = require('rtc/monitor');
var pc = new RTCPeerConnection(config);

// watch pc and when active do something
monitor(pc).once('active', function() {
  // active and ready to go
});
```

Events provided by the monitor are as follows:

- `active`: triggered when the connection is active and ready for use
- `stable`: triggered when the connection is in a stable signalling state
- `unstable`: trigger when the connection is renegotiating.

It should be noted, that the monitor does a check when it is first passed
an `RTCPeerConnection` object to see if the `active` state passes checks.
If so, the `active` event will be fired in the next tick.

If you require a synchronous check of a connection's "openness" then
use the `monitor.isActive` test outlined below.

### monitor.getState(pc)

Provides a unified state definition for the RTCPeerConnection based
on a few checks.

In emerging versions of the spec we have various properties such as
`readyState` that provide a definitive answer on the state of the 
connection.  In older versions we need to look at things like
`signalingState` and `iceGatheringState` to make an educated guess 
as to the connection state.

### monitor.isActive(pc) -> Boolean

Test an `RTCPeerConnection` to see if it's currently open.  The test for
"openness" looks at a combination of current `signalingState` and
`iceGatheringState`.

## Internal RTC Helper Libraries

The RTC library uses a number of helper modules that are contained within
the `lib/` folder of the `rtc-io/rtc` repository.  While these are designed
primarily for internal use, they can be accessed by directly requiring
the modules, e.g. `require('rtc/lib/helpermodule')`

## rtc/lib/listen

```js
var listen = require('rtc/lib/listen');

// listen for negotiation needed events
listen(pc).on('negotiationneeded', function(evt) {

});
```

The `listen` helper provides an event emitter for a peer connection object
that will bind to each of the core events WebRTC events (unless overriden
by providing the listen function additional arguments).

## License(s)

### Apache 2.0

Copyright 2014 National ICT Australia Limited (NICTA)

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
