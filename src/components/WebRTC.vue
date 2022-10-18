<script setup lang="ts">
import { onMounted, ref } from "vue";

const isDisableStart = ref(false);
const isDisableCall = ref(true);

let localVideo: HTMLVideoElement | null = null;
let remoteVideo: HTMLVideoElement | null = null;
let localStream: MediaStream | null = null;
let remoteStream: MediaStream | null = null;

let localPeerConnection: RTCPeerConnection | null = null;
let remotePeerConnection: RTCPeerConnection | null = null;

onMounted(() => {
  localVideo = document.getElementById("localVideo") as HTMLVideoElement;
  remoteVideo = document.getElementById("remoteVideo") as HTMLVideoElement;
});

const constraints = {
  video: true,
};

function CallAction() {
  localPeerConnection = new RTCPeerConnection();
  localPeerConnection.addEventListener("icecandidate", handleConnection);
  localPeerConnection.addEventListener(
    "connectionstatechange",
    handleConnectionChange
  );

  remotePeerConnection = new RTCPeerConnection();
  remotePeerConnection.addEventListener("icecandidate", () => handleConnection
  );
  remotePeerConnection.addEventListener(
    "connectionstatechange",
    handleConnectionChange
  );
  localStream!.getTracks().forEach((track) => localPeerConnection!.addTrack(track, localStream!));
  remotePeerConnection.addEventListener('track', gotRemoteMediaStream);

  localPeerConnection
    .createOffer({ offerToReceiveVideo: true })
    .then(createOffer);
}

async function startAction() {
  if (!localVideo) return;
  
  navigator.mediaDevices
    .getUserMedia(constraints)
    .then((mediaStream) => {
      localVideo!.srcObject = mediaStream;
      localStream = mediaStream;
      isDisableCall.value = false;
      isDisableStart.value = true;
      console.log(mediaStream);
      
    })
}

function gotRemoteMediaStream(e: RTCTrackEvent) {
    remoteVideo!.srcObject = e.streams[0];
}

function createOffer(description: RTCSessionDescriptionInit) {
  localPeerConnection?.setLocalDescription(description);
  remotePeerConnection?.setRemoteDescription(description);

  remotePeerConnection?.createAnswer().then(createAnswer);
}

function createAnswer(description: RTCSessionDescriptionInit) {
  remotePeerConnection?.setLocalDescription(description);
  localPeerConnection?.setRemoteDescription(description);
}

function getOther(connection: RTCPeerConnection) {
  return connection === localPeerConnection
    ? remotePeerConnection
    : localPeerConnection;
}

function handleConnection(event: RTCPeerConnectionIceEvent) {
  const peerConnection = event.target as RTCPeerConnection;
  const iceCandidate = event.candidate;

  if (iceCandidate) {
    const newIceCandidate = new RTCIceCandidate(iceCandidate);
    const otherPeer = getOther(peerConnection);

    otherPeer
      ?.addIceCandidate(newIceCandidate)
      // .then(() => handleConnectionSuccess(peerConnection))
      // .catch((error) => handleConnectionFailure(peerConnection, error));
  }
}

function handleConnectionSuccess(peerConnection: RTCPeerConnection) {}

function handleConnectionFailure(
  peerConnection: RTCPeerConnection,
  error: Error
) {}

function handleConnectionChange() {}
</script>

<script lang="ts">
export default {};
</script>

<template>
  <div class="wrapper-video">
    <div>
      <video id="localVideo" autoplay playsinline class="video" />
      <div>
        <button :disabled="isDisableStart" @click="startAction()" id="startButton">
          Start
        </button>
        <button :disabled="isDisableCall" id="callButton" @click="CallAction()">
          Call
        </button>
      </div>
  
    </div>
    <video id="remoteVideo" autoplay playsinline class="video" />

  </div>

</template>

<style scoped>

.wrapper-video {
  display: flex;
}
.video {
  max-width: 100%;
  width: 320px;
  height: 240px;
}
</style>
