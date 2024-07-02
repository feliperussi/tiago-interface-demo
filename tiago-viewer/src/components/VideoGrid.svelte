<script>
  // @ts-nocheck
    import { onMount } from 'svelte';
    import io from 'socket.io-client';
  
    let myId;
    let rtcPeerConnections = {};
    let dataChannels = {};
    const iceServers = {
      'iceServers': [
        { 'urls': 'stun:stun.l.google.com:19302' }
      ]
    };
    
    let showVideo = false;
  
    function startSignaling(myRole) {
      const socket = io('http://192.168.0.103:3000', {
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
            rtcPeerConnections[evt.from] = createRTCPeerConnection(evt.from);
            if (evt.role === 'p') {
              rtcPeerConnections[evt.from].addTransceiver('video', { direction: 'recvonly' });
            }
  
            dataChannels[evt.from] = rtcPeerConnections[evt.from].createDataChannel('chat');
            dataChannels[evt.from].onmessage = (e) => addChatMessage(e.data);
  
            rtcPeerConnections[evt.from].createOffer().then((sdp) => onOfferAnswer('offer', sdp, evt.from));
            break;
          case 'offer':
            console.log(evt.from + ' has sent an offer:', evt.sdp);
            rtcPeerConnections[evt.from] = createRTCPeerConnection(evt.from);
            rtcPeerConnections[evt.from].setRemoteDescription(new RTCSessionDescription(evt.sdp));
  
            rtcPeerConnections[evt.from].ondatachannel = (e) => receiveChannelCallback(e, evt.from);
            
            rtcPeerConnections[evt.from].createAnswer().then((sdp) => onOfferAnswer('answer', sdp, evt.from));
            break;
          case 'answer':
            console.log(evt.from + ' has sent an answer:', evt.sdp);
            rtcPeerConnections[evt.from].setRemoteDescription(new RTCSessionDescription(evt.sdp));
            break;
          case 'candidate':
            console.log(evt.from, 'sent a candidate:', evt.candidate);
            rtcPeerConnections[evt.from].addIceCandidate(new RTCIceCandidate({
              sdpMLineIndex: evt.label,
              candidate: evt.candidate
            }));
            break;
          case 'bye':
            console.log(evt.from + ' has left');
            const videoElement = document.getElementById('remote_' + evt.from);
            if (videoElement) {
              videoElement.pause();
              videoElement.removeAttribute('srcObject');
              videoElement.load();
              videoElement.remove();
              styleVideos();
            }
            if (rtcPeerConnections[evt.from]) {
              rtcPeerConnections[evt.from].close();
              delete rtcPeerConnections[evt.from];
            }
            if (dataChannels[evt.from]) {
              dataChannels[from].close();
              delete dataChannels[evt.from];
            }
            break;
        }
      });
  
      function sendSignaling(data) {
        socket.emit('event', data);
      }
  
      function createRTCPeerConnection(remoteUser) {
        const rtcPeerConnection = new RTCPeerConnection(iceServers);
        rtcPeerConnection.onicecandidate = e => onIceCandidate(e, remoteUser);
        rtcPeerConnection.ontrack = e => onAddStream(e, remoteUser);
        if (myRole === 'p') {
          rtcPeerConnection.addTrack(localVideo.srcObject.getVideoTracks()[0]);
        }
  
        return rtcPeerConnection;
      }
  
      function onOfferAnswer(type, sdp, to) {
        rtcPeerConnections[to].setLocalDescription(sdp);
        console.log('sending ' + type + ' to:', to);
        sendSignaling({
          to: to,
          from: myId,
          type: type,
          sdp: sdp,
        });
      }
  
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
          });
        }
      }
  
      function onAddStream(event, from) {
        console.log('got video from', from);
        const remoteVideo = document.createElement('video');
        remoteVideo.id = 'remote_' + from;
        remoteVideo.autoplay = true;
        remoteVideo.classList.add('fixed-size-video', 'bg-black');
        remoteVideo.srcObject = new MediaStream([event.track]);
        document.getElementsByClassName('videos')[0].appendChild(remoteVideo);
  
        styleVideos();
      }
    }
  
    function receiveChannelCallback(event, from) {
      dataChannels[from] = event.channel;
      dataChannels[from].onmessage = (e) => addChatMessage(e.data);
    }
  
    function addChatMessage(message, textAlign = 'left') {
      const el = document.createElement('p');
      el.style.textAlign = textAlign;
  
      const txtNode = document.createTextNode(message);
      
      el.appendChild(txtNode);
      chat.appendChild(el);
    }
  
    function styleVideos() {
      const videosGrid = document.querySelector('.videos');
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
          videosGrid.style.gridTemplateColumns = `repeat(${Math.ceil(videos.length / 2)}, 1fr)`;
        }
        videos.forEach((video, index) => {
          video.classList.add('fixed-size-video');
          video.style.gridRow = 'auto';
          video.style.gridColumn = 'auto';
        });
      }
    }
  
    onMount(() => {
      const btn = document.getElementById('join');
      btn.addEventListener('click', () => {
        btn.style.display = 'none';
        showVideo = true;
        startSignaling('v');
      });
    });
  </script>
  
  <style>
    .container {
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      height: 40rem;
    }
  
    .video-placeholder {
      width: 40rem;
      height: 30rem;
      background-color: gray;
      display: flex;
      justify-content: center;
      align-items: center;
      margin-bottom: 1rem;
    }
  
    .videos {
      display: grid;
      grid-template-columns: repeat(2, 1fr);
    }
  
    /* .fixed-size-video {
      width: 100%;
      height: 100%;
    } */
  
    button {
      margin-top: 0.6rem;
    }
  
    /* h1 {
      text-align: center;
    } */
  </style>
  
  <div class="container">
    <div class="video-placeholder">
      {#if showVideo}
        <div class="videos grid grid-cols-2 gap-2"></div>
      {/if}
    </div>
    <button id="join" class="px-4 py-2 bg-blue-500 text-white rounded">Join</button>
  </div>
  