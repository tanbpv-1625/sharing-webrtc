<script setup lang="ts">
import { onMounted, ref, watch, reactive } from "vue";
import firebase from "firebase/app";
import "firebase/firestore";

const callInput = ref("");

let remoteVideo: HTMLVideoElement | null = null;
let localVideo: HTMLVideoElement | null = null;
let localStream: MediaStream | null = null;
let remoteStream: MediaStream | null = null;

let localPeerConnection: RTCPeerConnection | null = null;

const firebaseConfig = {
  apiKey: "AIzaSyCKoRejb4GtZmbfkicJ-jBLdSV73vyRNWU",
  authDomain: "sharing-webrtc-4abe3.firebaseapp.com",
  projectId: "sharing-webrtc-4abe3",
  storageBucket: "sharing-webrtc-4abe3.appspot.com",
  messagingSenderId: "1067733797158",
  appId: "1:1067733797158:web:075d1cea3d99c05f2459e7",
  measurementId: "G-44TW5F0W9P",
};

if (!firebase.apps.length) {
  firebase.initializeApp(firebaseConfig);
}

const firestore = firebase.firestore();

const serversConfig = {
  iceServers: [
    {
      urls: ["stun:stun1.l.google.com:19302", "stun:stun2.l.google.com:19302"],
    },
  ],
  iceCandidatePoolSize: 10,
};

async function startAction() {
  localVideo = document.getElementById("localVideo") as HTMLVideoElement;
  remoteVideo = document.getElementById("remoteVideo") as HTMLVideoElement;
  remoteStream = new MediaStream();

  localPeerConnection = new RTCPeerConnection(serversConfig);

  localStream = await navigator.mediaDevices.getUserMedia({
    video: true,
    audio: true,
  });

  localStream
    .getTracks()
    .forEach((track) => localPeerConnection?.addTrack(track, localStream!));

  localPeerConnection!.ontrack = (event: RTCTrackEvent) => {
    event.streams[0]
      .getTracks()
      .forEach((track) => remoteStream?.addTrack(track));

    remoteVideo!.srcObject = remoteStream;
  };

  localVideo!.srcObject = localStream;
}

async function call() {
  // Reference Firestore collections for signaling
  const callDoc = firestore.collection("calls").doc();
  const offerCandidates = callDoc.collection("offerCandidates");
  const answerCandidates = callDoc.collection("answerCandidates");

  callInput.value = callDoc.id;

  // Get candidates for caller, save to db
  localPeerConnection!.onicecandidate = (event: RTCPeerConnectionIceEvent) => {
    event.candidate && offerCandidates.add(event.candidate.toJSON());
  };

  // Create Offer
  const offerDescription = await localPeerConnection?.createOffer();
  await localPeerConnection?.setLocalDescription(offerDescription);

  const offer = {
    sdp: offerDescription!.sdp,
    type: offerDescription!.type,
  };

  await callDoc.set({ offer });

  // Listen for remote answer
  callDoc.onSnapshot((snapshot) => {
    const data = snapshot.data();

    if (!localPeerConnection?.currentRemoteDescription && data?.answer) {
      localPeerConnection?.setRemoteDescription(
        new RTCSessionDescription(data.answer)
      );
    }
  });

  // Listen for remote ICE candidates
  answerCandidates.onSnapshot((snapshot) => {
    snapshot.docChanges().forEach((change) => {
      if (change.type === "added") {
        localPeerConnection?.addIceCandidate(
          new RTCIceCandidate(change.doc.data())
        );
      }
    });
  });
}

async function answer() {
  const callId = callInput.value;

  const callDoc = firestore.collection("calls").doc(callId);
  const offerCandidates = firestore.collection("offerCandidates");
  const answerCandidates = firestore.collection("answerCandidates");

  localPeerConnection!.onicecandidate = (event: RTCPeerConnectionIceEvent) => {
    event.candidate && answerCandidates.add(event.candidate.toJSON());
  };

  // Fetch data, then set the offer & answer
  const callData = (await callDoc.get()).data();

  const offerDescription = callData?.offer;
  localPeerConnection?.setRemoteDescription(offerDescription);

  const answerDescription = await localPeerConnection?.createAnswer();
  localPeerConnection?.setLocalDescription(answerDescription);

  const answer = {
    type: answerDescription?.type,
    sdp: answerDescription?.sdp,
  };

  await callDoc.update({ answer });

  offerCandidates.onSnapshot((snapshot) => {
    snapshot.docChanges().forEach((change) => {
      if (change.type === "added") {
        let data = change.doc.data();
        localPeerConnection?.addIceCandidate(new RTCIceCandidate(data));
      }
    });
  });
}
</script>

<script lang="ts">
export default {};
</script>

<template>
  <div class="wrapper-video">
    <div>
      <video id="localVideo" autoplay playsinline class="video" />
      <div>
        <button @click="startAction()" id="startButton">Start</button>
        <button id="callButton" @click="call()">Call</button>
      </div>
    </div>
  </div>
  <video id="remoteVideo" autoplay playsinline class="video" />
  <input type="text" name="callInput" v-model="callInput" />
  <button id="callButton" @click="answer()">Answer</button>
</template>

<style scoped>
.wrapper-video {
  display: flex;
  flex-direction: column;
}
.video {
  max-width: 100%;
  width: 320px;
  height: 240px;
}
</style>
