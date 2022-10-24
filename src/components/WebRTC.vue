<script setup lang="ts">
import { onMounted, ref } from "vue";
import { initializeApp } from "firebase/app";
import "firebase/firestore";
import {
  collection,
  setDoc,
  addDoc,
  doc,
  getFirestore,
  onSnapshot,
  getDoc,
  updateDoc,
} from "firebase/firestore";

const callInput = ref("");

let localVideo: HTMLVideoElement | null = null;
let remoteVideo: HTMLVideoElement | null = null;
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

const app = initializeApp(firebaseConfig);
const db = getFirestore(app);

const serversConfig = {
  iceServers: [
    {
      urls: ["stun:stun1.l.google.com:19302", "stun:stun2.l.google.com:19302"],
    },
  ],
  iceCandidatePoolSize: 10,
};

onMounted(() => {
  localVideo = document.getElementById("localVideo") as HTMLVideoElement;
  remoteVideo = document.getElementById("remoteVideo") as HTMLVideoElement;
});

async function startAction() {
  localPeerConnection = new RTCPeerConnection(serversConfig);

  localStream = await navigator.mediaDevices.getUserMedia({
    video: true,
    audio: true,
  });

  localStream
    .getTracks()
    .forEach((track) => localPeerConnection?.addTrack(track, localStream!));

  localPeerConnection!.ontrack = (event: RTCTrackEvent) => {
    console.log("event", event);

    event.streams[0]
      .getTracks()
      .forEach((track) => remoteStream?.addTrack(track));
  };

  remoteVideo!.srcObject = remoteStream;
  localVideo!.srcObject = localStream;
}

async function call() {
  // Reference Firestore collections for signaling
  const callDoc = doc(collection(db, "calls"));
  const offerCandidates = collection(db, "offerCandidates");
  const answerCandidates = collection(db, "answerCandidates");

  callInput.value = callDoc.id;

  // Get candidates for caller, save to db
  localPeerConnection!.onicecandidate = (event: RTCPeerConnectionIceEvent) => {
    event.candidate && addDoc(offerCandidates, event.candidate.toJSON());
  };

  // Create Offer
  const offerDescription = await localPeerConnection?.createOffer();
  await localPeerConnection?.setLocalDescription(offerDescription);

  const offer = {
    sdp: offerDescription!.sdp,
    type: offerDescription!.type,
  };

  await setDoc(callDoc, { offer });

  // Listen for remote answer
  const snapshots = await onSnapshot(callDoc, (snapshot) => {
    const data = snapshot.data();

    if (!localPeerConnection?.currentRemoteDescription && data?.answer) {
      const answerDescription = new RTCSessionDescription(data.answer);
      localPeerConnection?.setRemoteDescription(answerDescription);
    }
  });

  // Listen for remote ICE candidates
  onSnapshot(answerCandidates, (snapshot) => {
    snapshot.docChanges().forEach((change) => {
      if (change.type === "added") {
        const candidate = new RTCIceCandidate(change.doc.data());
        localPeerConnection?.addIceCandidate(candidate);
      }
    });
  });
}

async function answer() {
  const callId = callInput.value;
  const callDoc = doc(collection(db, "calls"), callId);
  const offerCandidates = collection(db, "offerCandidates");
  const answerCandidates = collection(db, "answerCandidates");

  localPeerConnection!.onicecandidate = (event: RTCPeerConnectionIceEvent) => {
    event.candidate && addDoc(offerCandidates, event.candidate.toJSON());
  };

  // Fetch data, then set the offer & answer
  const callData = (await getDoc(callDoc)).data();

  const offerDescription = callData?.offer;
  localPeerConnection?.setRemoteDescription(offerDescription);

  const answerDescription = await localPeerConnection?.createAnswer();
  localPeerConnection?.setLocalDescription(answerDescription);

  const answer = {
    type: answerDescription?.type,
    sdp: answerDescription?.sdp,
  };

  await updateDoc(callDoc, answer);

  onSnapshot(offerCandidates, (snapshot) => {
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
    <video id="remoteVideo" autoplay playsinline class="video" />
  </div>
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
