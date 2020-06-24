# Kraken

🐙 High performance WebRTC SFU implemented with pure Go.

## Architecture

Kraken SFU only supports simple group audio conferencing, more features may be added easily.

### monitor [WIP]

This is the daemon that load balance all engine instances according to their system load, and it will direct all peers in a room to the same engine instance.

### engine 

The engine handles rooms, all peers in a room should connect to the same engine instance. No need to create rooms, a room is just an ID to distribute streams.

Access the engine with HTTP JSON-RPC, some pseudocode to demonstrate the full procedure.

```javascript
var roomId = getUrlQueryParameter('room');
var userId = uuidv4();
var trackId;

var pc = new RTCPeerConnection(configuration);

// send ICE candidate to engine
pc.onicecandidate = ({candidate}) => {
  rpc('trickle', [roomId, userId, trackId, JSON.stringify(candidate)]);
};

// play the audio stream when available
pc.ontrack = (event) => { 
  el = document.createElement(event.track.kind)
  el.id = aid;
  el.srcObject = stream;
  el.autoplay = true;
  document.getElementById('peers').appendChild(el)
};

// setup local audio stream from microphone
const stream = await navigator.mediaDevices.getUserMedia(constraints);
stream.getTracks().forEach((track) => {
  pc.addTrack(track, stream);
});
await pc.setLocalDescription(await pc.createOffer());

// RPC publish to roomId, with SDP offer
var res = await rpc('publish', [roomId, userId, JSON.stringify(pc.localDescription)]);
// publish should respond an SDP answer
if (res.data && res.data.sdp.type === 'answer') {
  await pc.setRemoteDescription(res.data.sdp);
  trackId = res.data.track;
  subscribe(pc);
}

// RPC subscribe to roomId periodically  
async function subscribe(pc) {
  var res = await rpc('subscribe', [roomId, userId, trackId]);

  if (res.data && res.data.type === 'offer') {
    await pc.setRemoteDescription(res.data);
    var sdp = await pc.createAnswer();
    await pc.setLocalDescription(sdp);
    await rpc('answer', [roomId, userId, trackId, JSON.stringify(sdp)]);
  }
  setTimeout(function () {
    subscribe(pc);
  }, 3000);
}

async function rpc(method, params = []) {
  const response = await fetch('http://localhost:7000', {
    method: 'POST',
    mode: 'cors', 
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({id: uuidv4(), method: method, params: params})
  });
  return response.json(); 
}
``` 

## Quick Start

Setup Golang development environment at first.

```
git clone github.com/MixinNetwork/kraken
cd kraken && go build

cp config/engine.example.toml config/engine.toml
ip address # get your network interface name, edit config/engine.toml

./kraken -c config/engine.toml -s engine
```

Get the source code of Mornin(https://github.com/fox-one/mornin.fm) and configure it to use your local kraken API.
