<script setup lang="ts">
import { ref } from "vue";
import firebase from "firebase/app";
import "firebase/firestore";

const roomInput = ref("");

let remoteVideo: HTMLVideoElement | null = null;
let localVideo: HTMLVideoElement | null = null;
let localStream: MediaStream | null = null;
let remoteStream: MediaStream | null = null;

let peerConnection: RTCPeerConnection | null = null;

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
      urls: "stun:openrelay.metered.ca:80",
    },
    {
      urls: "turn:openrelay.metered.ca:80",
      username: "openrelayproject",
      credential: "openrelayproject",
    },
    {
      urls: "turn:openrelay.metered.ca:443",
      username: "openrelayproject",
      credential: "openrelayproject",
    },
    {
      urls: "turn:openrelay.metered.ca:443?transport=tcp",
      username: "openrelayproject",
      credential: "openrelayproject",
    },
  ],
  iceCandidatePoolSize: 10,
};

async function startAction() {
  localVideo = document.getElementById("localVideo") as HTMLVideoElement;
  remoteVideo = document.getElementById("remoteVideo") as HTMLVideoElement;
  remoteStream = new MediaStream();

  peerConnection = new RTCPeerConnection(serversConfig);

  localStream = await navigator.mediaDevices.getUserMedia({
    video: true,
    audio: true,
  });

  localStream
    .getTracks()
    .forEach((track) => peerConnection?.addTrack(track, localStream!));

  remoteVideo.srcObject = remoteStream;
  localVideo!.srcObject = localStream;
}

async function call() {
  // Code for collecting ICE candidates below
  const room = firestore.collection("rooms").doc();
  const callerCandidates = room.collection("callerCandidates");
  const answerCandidates = room.collection("answerCandidates");

  roomInput.value = room.id;

  // Get candidates for caller, save to db
  peerConnection!.onicecandidate = (event: RTCPeerConnectionIceEvent) => {
    if (!event.candidate) {
      console.log("not found event.candidate");

      return;
    }

    callerCandidates.add(event.candidate.toJSON());
  };

  // Create Offer
  const offer = await peerConnection?.createOffer();
  await peerConnection?.setLocalDescription(offer);

  const offerRooms = {
    offer: {
      sdp: offer!.sdp,
      type: offer!.type,
    },
    roomId: room.id,
  };

  await room.set(offerRooms);
  console.log("roomId", room.id);

  peerConnection!.ontrack = (event: RTCTrackEvent) => {
    console.log("Get remote track", event.streams[0]);
    // event.streams[0]
    //   .getTracks()
    //   .forEach((track) => remoteStream?.addTrack(track));

    remoteVideo!.srcObject = event.streams[0];
  };

  // Listen for remote answer
  room.onSnapshot(async (snapshot) => {
    const data = snapshot.data();
    if (peerConnection?.iceConnectionState !== "closed") {
      if (!peerConnection?.currentRemoteDescription && data && data.answer) {
        const rtcSessionDescription = new RTCSessionDescription(data.answer);
        await peerConnection?.setRemoteDescription(rtcSessionDescription);
      }
    }
  });

  // Listen for remote ICE candidates
  answerCandidates.onSnapshot((snapshot) => {
    snapshot.docChanges().forEach(async (change) => {
      if (change.type === "added") {
        const ice = new RTCIceCandidate(change.doc.data());
        console.log(`Got new remote ICE candidate: ${ice}`);
        await peerConnection?.addIceCandidate(ice);
      }
    });
  });
}

async function answer() {
  const roomId = roomInput.value;
  peerConnection = new RTCPeerConnection(serversConfig);

  const room = firestore.collection("rooms").doc(roomId);

  // Fetch data, then set the offer & answer
  const roomSnapshot = await room.get();

  if (roomSnapshot.exists) {
    localStream?.getTracks().forEach((track) => {
      peerConnection?.addTrack(track, localStream!);
    });

    const callerCandidates = room.collection("callerCandidates");
    const answerCandidates = room.collection("answerCandidates");

    // Collect ICE candidate
    peerConnection!.onicecandidate = (event: RTCPeerConnectionIceEvent) => {
      if (!event.candidate) {
        console.log("Not found candidate");

        return;
      }

      answerCandidates.add(event.candidate.toJSON());
    };

    peerConnection!.ontrack = (event: RTCTrackEvent) => {
      console.log("Get remote track", event.streams[0]);
      // event.streams[0]
      //   .getTracks()
      //   .forEach((track) => remoteStream?.addTrack(track));

      remoteVideo!.srcObject = event.streams[0];
    };
    const offerRoom = roomSnapshot.data()?.offer;
    console.log("Got offer", offerRoom);

    await peerConnection.setRemoteDescription(
      new RTCSessionDescription(offerRoom)
    );

    const answer = await peerConnection?.createAnswer();
    peerConnection?.setLocalDescription(answer);

    const answerRoom = {
      answer: {
        type: answer?.type,
        sdp: answer?.sdp,
      },
    };

    await room.update(answerRoom);

    callerCandidates.onSnapshot((snapshot) => {
      snapshot.docChanges().forEach(async (change) => {
        if (change.type === "added") {
          let ice = change.doc.data();
          console.log(`Got new remote ICE candidate: ${ice}`);
          await peerConnection?.addIceCandidate(new RTCIceCandidate(ice));
        }
      });
    });
  }
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
  <input type="text" name="roomInput" v-model="roomInput" />
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
