<script>
  import { onMount } from 'svelte';
  import io from 'socket.io-client';

  /**
   * @type {string | undefined}
   */
  let myId;
  let rtcPeerConnections = {};
  let dataChannels = {};
  const iceServers = {
    'iceServers': [
      { 'urls': 'stun:stun.l.google.com:19302' }
    ]
  };

  /**
   * @param {string} myRole
   */
  function startSignaling(myRole) {
    const socket = io('https://10.68.0.129:3000', {
      query: {
        role: myRole
      },
      secure: true,
      reconnection: true,
      rejectUnauthorized: false
    });

    socket.on('connect', () => {
      console.log('My id is:', socket.id);
      myId = socket.id;
    });

    socket.on('event', evt => {
      switch (evt.type) {
        case 'join':
          console.log(evt.from + ' has joined');
          // @ts-ignore
          rtcPeerConnections[evt.from] = createRTCPeerConnection(evt.from);
          if (evt.role === 'p') {
            // @ts-ignore
            rtcPeerConnections[evt.from].addTransceiver('video', { direction: 'recvonly' });
          }

          // @ts-ignore
          dataChannels[evt.from] = rtcPeerConnections[evt.from].createDataChannel('chat');
          // @ts-ignore
          dataChannels[evt.from].onmessage = (/** @type {{ data: any; }} */ e) => addChatMessage(e.data);

          // @ts-ignore
          rtcPeerConnections[evt.from].createOffer().then((/** @type {any} */ sdp) => onOfferAnswer('offer', sdp, evt.from));
          break;
        case 'offer':
          console.log(evt.from + ' has sent an offer:', evt.sdp);
          // @ts-ignore
          rtcPeerConnections[evt.from] = createRTCPeerConnection(evt.from);
          // @ts-ignore
          rtcPeerConnections[evt.from].setRemoteDescription(new RTCSessionDescription(evt.sdp));

          // @ts-ignore
          rtcPeerConnections[evt.from].ondatachannel = (/** @type {any} */ e) => receiveChannelCallback(e, evt.from);
          
          // @ts-ignore
          rtcPeerConnections[evt.from].createAnswer().then((/** @type {any} */ sdp) => onOfferAnswer('answer', sdp, evt.from));
          break;
        case 'answer':
          console.log(evt.from + ' has sent an answer:', evt.sdp);
          // @ts-ignore
          rtcPeerConnections[evt.from].setRemoteDescription(new RTCSessionDescription(evt.sdp));
          break;
        case 'candidate':
          console.log(evt.from, 'sent a candidate:', evt.candidate);
          // @ts-ignore
          rtcPeerConnections[evt.from].addIceCandidate(new RTCIceCandidate({
            sdpMLineIndex: evt.label,
            candidate: evt.candidate
          }));
          break;
        case 'bye':
          console.log(evt.from + ' has left');
          const videoElement = document.getElementById('remote_' + evt.from);
          if (videoElement) {
            // @ts-ignore
            videoElement.pause();
            videoElement.removeAttribute('srcObject'); // empty source
            // @ts-ignore
            videoElement.load();
            videoElement.remove();
            styleVideos();
          }
          // @ts-ignore
          if (rtcPeerConnections[evt.from]) {
            // @ts-ignore
            rtcPeerConnections[evt.from].close();
            // @ts-ignore
            delete rtcPeerConnections[evt.from];
          }
          // @ts-ignore
          if (dataChannels[evt.from]) {
            // @ts-ignore
            dataChannels[evt.from].close();
            // @ts-ignore
            delete dataChannels[evt.from];
          }
          break;
      }
    });

    /**
     * @param {{ to: any; from: any; type: any; sdp?: any; label?: any; id?: any; candidate?: any; }} data
     */
    function sendSignaling(data) {
      socket.emit('event', data);
    }

    /**
     * @param {any} remoteUser
     */
    function createRTCPeerConnection(remoteUser) {
      const rtcPeerConnection = new RTCPeerConnection(iceServers);
      rtcPeerConnection.onicecandidate = e => onIceCandidate(e, remoteUser);
      rtcPeerConnection.ontrack = e => onAddStream(e, remoteUser);
      if (myRole === 'p') {
        // @ts-ignore
        rtcPeerConnection.addTrack(localVideo.srcObject.getVideoTracks()[0]);
      }

      return rtcPeerConnection;
    }

    /**
     * @param {string} type
     * @param {any} sdp
     * @param {string | number} to
     */
    function onOfferAnswer(type, sdp, to) {
      // @ts-ignore
      rtcPeerConnections[to].setLocalDescription(sdp);
      console.log('sending ' + type + ' to:', to);
      sendSignaling({
        to: to,
        from: myId,
        type: type,
        sdp: sdp,
      });
    }

    /**
     * @param {RTCPeerConnectionIceEvent} event
     * @param {any} to
     */
    function onIceCandidate(event, to) {
      if (event.candidate) {
        console.log('sending ice candidate to:', to);
        sendSignaling({
          type: 'candidate',
          from: myId,
          to: to,
          label: event.candidate.sdpMLineIndex,
          id: event.candidate.sdpMid,
          candidate: event.candidate.candidate
        })
      }
    }

    /**
     * @param {RTCTrackEvent} event
     * @param {string} from
     */
    function onAddStream(event, from) {
      console.log('got video from', from);
      const remoteVideo = document.createElement('video');
      remoteVideo.id = 'remote_' + from;
      remoteVideo.autoplay = true;
      remoteVideo.srcObject = new MediaStream([event.track]);
      document.getElementsByClassName('videos')[0].appendChild(remoteVideo);

      styleVideos();
    }
  }

  /**
   * @param {{ channel: any; }} event
   * @param {string | number} from
   */
  function receiveChannelCallback(event, from) {
    // @ts-ignore
    dataChannels[from] = event.channel;
    // @ts-ignore
    dataChannels[from].onmessage = (/** @type {{ data: any; }} */ e) => addChatMessage(e.data);
  }

  /**
   * @param {string} message
   */
  function addChatMessage(message, textAlign = 'left') {
    const el = document.createElement('p');
    el.style.textAlign = textAlign;

    const txtNode = document.createTextNode(message);
    
    el.appendChild(txtNode);
    // @ts-ignore
    chat.appendChild(el);
  }

  function styleVideos() {
    const videosGrid = document.querySelector('.videos');
    // @ts-ignore
    const videos = videosGrid.querySelectorAll('video');

    if (videos.length === 1) {
      videos[0].style.gridRow = '1 / span 2';
      videos[0].style.gridColumn = '1 / span 2';
    } else if (videos.length === 2) {
      videos[0].style.gridRow = '1 / span 2';
      videos[0].style.gridColumn = '1 / span 1';
      videos[1].style.gridRow = '1 / span 2';
      videos[1].style.gridColumn = '2 / span 1';
    } else if (videos.length >= 3) {
      if (videos.length > 4) {
        // @ts-ignore
        videosGrid.style.gridTemplateColumns = `repeat(${Math.ceil(videos.length / 2)}, 1fr)`;
      }
      // @ts-ignore
      videos.forEach((video, index) => {
        video.style.gridRow = 'auto';
        video.style.gridColumn = 'auto';
      });
    }
  }

  onMount(() => {
    const btn = document.getElementById('join');
    // @ts-ignore
    btn.addEventListener('click', () => {
      // @ts-ignore
      btn.style.display = 'none';
      startSignaling('v');
    });
  });
</script>

<style>
  .container {
    display: flex;
    justify-content: center;
    align-items: center;
  }

  .videos {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 10px;
  }

  h1 {
    text-align: center;
  }
</style>

<h1>Live Video Processing Demo</h1>
<button id="join">Join</button>
<div class="container">
  <div class="videos"></div>
</div>
