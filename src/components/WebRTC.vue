<script setup lang="ts">
import { onMounted, ref } from "vue";

const isDisableStart = ref(false);
const isDisableCall = ref(true);
const isDisableHangUp = ref(true);

let localVideo: HTMLVideoElement | null = null;
let remoteVideo: HTMLVideoElement | null = null;
let localStream: MediaStream | null = null;
let remoteStream: MediaStream | null = null;

let localPeerConnection: RTCPeerConnection | null = null;
let remotePeerConnection: RTCPeerConnection | null = null;

onMounted(() => {
  localVideo = document.getElementById("localVideo") as HTMLVideoElement;
  remoteVideo = document.getElementById("removeVideo") as HTMLVideoElement;
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
  localPeerConnection
    .createOffer({ offerToReceiveVideo: true })
    .then(createOffer);
}

async function startAction() {

  navigator.mediaDevices
    .getUserMedia(constraints)
    .then((mediaStream) => {
      localVideo && localVideo.srcObject = mediaStream;
      localStream = mediaStream;
      isDisableCall.value = false;
      isDisableHangUp.value = false;
      isDisableStart.value = true;
      addStream(mediaStream);

      console.log("stream is ready");
    })
    .catch(() => "navigator.mediaDevices not support");
}

function createOffer(description: RTCSessionDescriptionInit) {
  console.log("localPeerConnection setLocalDescription");
  localPeerConnection?.setLocalDescription(description);

  console.log("remotePeerConnection setRemoteDescription");
  remotePeerConnection?.setRemoteDescription(description);

  console.log("remotePeerConnection createAnswer");
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
      .then(() => handleConnectionSuccess(peerConnection))
      .catch((error) => handleConnectionFailure(peerConnection, error));
  }
}

function handleConnectionSuccess(peerConnection: RTCPeerConnection) {}

function handleConnectionFailure(
  peerConnection: RTCPeerConnection,
  error: Error
) {}

function handleConnectionChange() {}

function addStream(stream: MediaStream) {
  stream
    .getTracks()
    .forEach(
      (track) =>
        localPeerConnection && localPeerConnection.addTrack(track, stream)
    );
}
</script>

<script lang="ts">
export default {};
</script>

<template>
  <video id="localVideo" autoplay playsinline class="video" />
  <video id="remoteVideo" autoplay playsinline class="video" />

  <div>
    <button :disabled="isDisableStart" @click="startAction()" id="startButton">
      Start
    </button>
    <button :disabled="isDisableCall" id="callButton" @click="CallAction()">
      Call
    </button>
    <button id="hangupButton" :disabled="isDisableHangUp">Hang Up</button>
  </div>
</template>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.video {
  max-width: 100%;
  width: 320px;
}
</style>
