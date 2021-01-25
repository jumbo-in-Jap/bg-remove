<template>
  <div id="app" >
    <el-row>
      <el-col :span="6">
        <span>自分の映像（Video）</span>
        <video id="local-stream" width="300" height="200"></video>
      </el-col>
      <el-col :span="6">
        <span>加工後の映像（Canvas）</span>
        <canvas id="local-stream-output" ref="local-output" width="300" height="200"/>
        <canvas id="temp-canvas" width="300" height="200"/>
      </el-col>
      <el-col :span="6">
        <span>背景画像</span>
        <img id="bg" src="/bg.jpg" width="300" height="200" />
      </el-col>
    </el-row>
    <el-row style="margin-top:20px;">
      <el-col :span="6">
        <span>相手の映像（Video）</span>
        <video id="remote-stream"  width="300" height="200"></video>
      </el-col>
      <el-col :span="6">
        <canvas id="remote-stream-output" ref="remote-output" width="300" height="200"/>
      </el-col>
    </el-row>
    <el-button @click="call">call</el-button>
    <el-button @click="segmentBody">remove</el-button>
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
  OPTION: any = {
      audioCodec: 'G722',
      audioBandwidth: 100,
      videoCodec: 'H264',
      videoBandwidth: 250,
      videoReceiveEnabled: true,
      audioReceiveEnabled: true
  }

  async mounted() {
    await this.init()
    console.log('Using TensorFlow backend: ', tf.getBackend());
  }
  
  async init() {
    const peerId = document.location.search == "" ? 'user1' : 'user2'
    this.peer = new Peer(peerId, { key: '85902050-5528-4f31-8dc9-019fad7e974d', debug: 3, })
    this.peer.on('call', mediaConnection => {
      mediaConnection.answer(this.localStream!!, this.OPTION);
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
      audio: true, video: true
    })
    this.localStream = localStream
    const localVideoElement = document.getElementById('local-stream') as HTMLVideoElement
    localVideoElement.srcObject = localStream
    await localVideoElement.play()
  }

  /** onclick methos */
  public call() {
    const targetPeerId = document.location.search != "" ? 'user1' : 'user2'
    // const localCanvas = document.getElementById('local-stream-output') as CanvasElement
    // const localStream = localCanvas.captureStream()
    const mediaConnection = this.peer!!.call(targetPeerId, this.localStream!!, this.OPTION);
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

      function setBokeh(outputCanvas: HTMLCanvasElement, inputVideoElement: HTMLVideoElement, segmentation: bodyPix.SemanticPersonSegmentation){
        const backgroundBlurAmount = 8;
        const edgeBlurAmount = 8;
        const flipHorizontal = false;
        bodyPix.drawBokehEffect(
          outputCanvas,
          inputVideoElement,
          segmentation,
          backgroundBlurAmount,
          edgeBlurAmount,
          flipHorizontal
        );
      }
      // setBokeh(output, inputElement, segmentation)

      function setBG(outputCanvas: HTMLCanvasElement, inputVideoElement: HTMLVideoElement, segmentation: bodyPix.SemanticPersonSegmentation){
        const tempCanvas = document.getElementById('temp-canvas') as HTMLCanvasElement
        const tempCtx = tempCanvas.getContext("2d")!!
        const destCtx = outputCanvas.getContext('2d')!!

        const mask = bodyPix.toMask(segmentation)
        // maskのピクセルを操作する
        const img = new Image()
        img.src = "/bg.jpg"
        img.width = 300
        img.height = 200
        const bgcanvas = document.createElement('canvas');
        bgcanvas.width = inputElement.width;
        bgcanvas.height = inputElement.height;
        const ctxBg = bgcanvas.getContext('2d')!!
        ctxBg.drawImage(img, 0, 0, img.width, img.height, 0, 0, inputElement.width, inputElement.height);
        const bgImg = ctxBg.getImageData(0, 0, inputElement.width, inputElement.height)

        for (let y = 0; y < mask.height; ++y) {
          for (let x = 0; x < mask.width; ++x) {
            const base = (y * mask.width + x) * 4;
            if(mask.data[base + 3] !== 0){
              // なんかピクセルに書き込む
              mask.data[base + 0] = bgImg.data[base + 0];  // Red
              mask.data[base + 1] = bgImg.data[base + 1];  // Green
              mask.data[base + 2] = bgImg.data[base + 2];  // Blue
              mask.data[base + 3] = bgImg.data[base + 3];  // Alpha
            }
          }
        }
        tempCtx.putImageData(mask, 0, 0)

        // 元映像を流す
        destCtx.drawImage(inputElement, 0, 0, inputElement.width, inputElement.height)
        destCtx.save();
        //destCtx.globalCompositeOperation = "destination-out";
        // マスク画像を流す
        destCtx.drawImage(tempCanvas, 0, 0, inputElement.width, inputElement.height)
        destCtx.restore()
      }
      setBG(output, inputElement, segmentation)

      // const foregroundColor = {r: 0, g: 0, b: 0, a: 0};
      // const backgroundColor = {r: 0, g: 0, b: 0, a: 255};
      // const backgroundDarkeningMask = bodyPix.toMask(
      //   segmentation, foregroundColor, backgroundColor);
      // const opacity = 1.0;
      // const maskBlurAmount = 3;
      // bodyPix.drawMask(
      //  output, inputElement, backgroundDarkeningMask, opacity, maskBlurAmount, flipHorizontal);

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
  color: #EEEEEE;
  background-color: #777777;
  padding: 10px;
}
</style>
