<template>
  <div id="app">
    <video id="local-stream" width="300" height="200"></video>
    <canvas id="local-stream-output" ref="local-output" width="300" height="200"/>
    <video id="remote-stream"  width="300" height="200"></video>
    <canvas id="remote-stream-output" ref="remote-output" width="300" height="200"/>
    <button @click="call">call</button>
    <button @click="segmentBody">remove</button>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
import Peer from 'skyway-js'
import * as bodyPix from '@tensorflow-models/body-pix'
import * as tf from '@tensorflow/tfjs';
import { BodyPixInput } from '@tensorflow-models/body-pix/dist/types';

interface CanvasElement extends HTMLCanvasElement {
  captureStream(frameRate?: number): MediaStream;
}

@Component({
  components: {
  },
})
export default class App extends Vue {
  peer: Peer | null = null
  localStream: MediaStream | null = null

  async mounted() {
    await this.init()
    console.log('Using TensorFlow backend: ', tf.getBackend());
  }
  
  async init() {
    const peerId = document.location.search == "" ? 'user1' : 'user2'
    this.peer = new Peer(peerId, { key: '85902050-5528-4f31-8dc9-019fad7e974d', debug: 3, })
    this.peer.on('call', mediaConnection => {
      mediaConnection.answer(this.localStream!!);
      mediaConnection.on('stream', async stream => {
        const remoteVideoElement = document.getElementById('remote-stream') as HTMLVideoElement
        remoteVideoElement.srcObject = stream
        await remoteVideoElement.play()
      });
    })
    await this.setupVideo()
  }

  async setupVideo() {
    const localStream = await navigator.mediaDevices.getUserMedia({
      audio: false, video: true
    })
    this.localStream = localStream
    const localVideoElement = document.getElementById('local-stream') as HTMLVideoElement
    localVideoElement.srcObject = localStream
    await localVideoElement.play()
  }

  /** onclick methos */
  public call() {
    const targetPeerId = document.location.search != "" ? 'user1' : 'user2'
    const localCanvas = document.getElementById('local-stream-output') as CanvasElement
    const localStream = localCanvas.captureStream()
    const mediaConnection = this.peer!!.call(targetPeerId, localStream);
    // const mediaConnection = this.peer!!.call(targetPeerId, this.localStream!!);
    mediaConnection.on('stream', async stream => {
      const remoteVideoElement = document.getElementById('remote-stream') as HTMLVideoElement
      remoteVideoElement.srcObject = stream
      await remoteVideoElement.play()
    });
  }

  public async segmentBody() {
    const net = await bodyPix.load()
 
    async function renderFrame(){
      const input = document.getElementById('local-stream') as BodyPixInput
      const inputElement = document.getElementById('local-stream') as HTMLVideoElement
      const output = document.getElementById('local-stream-output') as HTMLCanvasElement
      const segmentation = await net.segmentPerson(input);
      const backgroundBlurAmount = 8;
      const edgeBlurAmount = 8;
      const flipHorizontal = false;
      bodyPix.drawBokehEffect(
        output,
        inputElement,
        segmentation,
        backgroundBlurAmount,
        edgeBlurAmount,
        flipHorizontal
      );
      requestAnimationFrame(renderFrame)
    }
    renderFrame()
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
