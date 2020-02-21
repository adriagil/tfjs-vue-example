<template>
  <div id="app">
    <h3 v-if="!isVideoStreamReady && !initFailMessage">Initializing webcam stream ...</h3>
    <h3 v-if="!isModelReady && !initFailMessage">loading model ...</h3>
    <h3 v-if="initFailMessage">Failed to init stream and/or model - {{ initFailMessage }}</h3>

    <div class="resultFrame">
      <video ref="video" autoplay></video>
      <canvas ref="canvas" :width="resultWidth" :height="resultHeight"></canvas>
    </div>
  </div>
</template>

<script>
import * as tf from '@tensorflow/tfjs'
import { loadGraphModel } from '@tensorflow/tfjs-converter'

const MODEL_URL = 'http://localhost:8081/web_model/model.json'

export default {
  name: 'app',

  data () {
    return {
      // store the promises of initialization
      streamPromise: null,
      modelPromise: null,

      // control the UI visibilities
      isVideoStreamReady: false,
      isModelReady: false,
      initFailMessage: '',

      // tfjs model related
      model: null,

      videoRatio: 1,
      resultWidth: 0,
      resultHeight: 0
    }
  },

  methods: {
    initWebcamStream () {
      // if the browser supports mediaDevices.getUserMedia API
      if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
        return navigator.mediaDevices.getUserMedia({
          audio: false, // don't capture audio
          video: { facingMode: 'environment' } // use the rear camera if there is
        })
          .then(stream => {
            // set <video> source as the webcam input
            let video = this.$refs.video
            try {
              video.srcObject = stream
            } catch (error) {
              // support older browsers
              video.src = URL.createObjectURL(stream)
            }

            /*
              model.detect uses tf.fromPixels to create tensors.
              tf.fromPixels api will get the <video> size from the width and height attributes,
                which means <video> width and height attributes needs to be set before called model.detect

              To make the <video> responsive, I get the initial video ratio when it's loaded (onloadedmetadata)
              Then addEventListener on resize, which will adjust the size but remain the ratio
              At last, resolve the Promise.
            */
            return new Promise((resolve, reject) => {
              // when video is loaded
              video.onloadedmetadata = () => {
                // calculate the video ratio
                this.videoRatio = video.offsetHeight / video.offsetWidth
                // add event listener on resize to reset the <video> and <canvas> sizes
                window.addEventListener('resize', this.setResultSize)
                // set the initial size
                this.setResultSize()

                this.isVideoStreamReady = true
                console.log('webcam stream initialized')
                resolve()
              }
            })
          })
          .catch(error => {
            console.log('failed to initialize webcam stream', error)
            throw (error)
          })
      } else {
        return Promise.reject(new Error('Your browser does not support mediaDevices.getUserMedia API'))
      }
    },

    setResultSize () {
      // get the current browser window size
      let clientWidth = document.documentElement.clientWidth

      // set max width as 600
      this.resultWidth = Math.min(600, clientWidth)
      // set the height according to the video ratio
      this.resultHeight = this.resultWidth * this.videoRatio

      // set <video> width and height
      /*
        Doesn't use vue binding :width and :height,
          because the initial value of resultWidth and resultHeight
          will affect the ratio got from the initWebcamStream()
      */
      let video = this.$refs.video
      video.width = this.resultWidth
      video.height = this.resultHeight
    },

    loadCustomModel () {
      this.isModelReady = false

      // load the model with loadGraphModel
      return loadGraphModel(MODEL_URL)
        .then((model) => {
          this.model = model
          this.isModelReady = true
          console.log('model loaded: ', model)
        })
        .catch((error) => {
          console.log('failed to load the model', error)
          throw (error)
        })
    },

    async detectObjects () {
      if (!this.isModelReady) return

      const tfImg = tf.browser.fromPixels(this.$refs.video)
      const smallImg = tf.image.resizeBilinear(tfImg, [300, 300]) // 600, 450
      const resized = tf.cast(smallImg, 'float32')
      const tf4d = tf.tensor4d(Array.from(resized.dataSync()), [1, 300, 300, 3]) // 600, 450
      let predictions = await this.model.executeAsync({ image_tensor: tf4d }, ['detection_boxes', 'num_detections', 'detection_classes', 'detection_scores'])

      this.renderPredictionBoxes(predictions[0].dataSync(), predictions[1].dataSync(), predictions[2].dataSync(), predictions[3].dataSync())

      tfImg.dispose()
      smallImg.dispose()
      resized.dispose()
      tf4d.dispose()

      requestAnimationFrame(() => {
        this.detectObjects()
      })
    },

    loadModelAndDetection () {
      this.modelPromise = this.loadCustomModel()

      // wait for both stream and model promise finished then start detecting objects
      Promise.all([this.streamPromise, this.modelPromise])
        .then(() => {
          this.detectObjects()
        }).catch((error) => {
          console.log('Failed to init stream and/or model: ')
          this.initFailMessage = error
        })
    },

    renderPredictionBoxes (predictionBoxes, totalPredictions, predictionClasses, predictionScores) {
      // get the context of canvas
      const ctx = this.$refs.canvas.getContext('2d')
      // clear the canvas
      ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height)
      // draw results
      for (let i = 0; i < totalPredictions[0]; i++) {
        const minY = predictionBoxes[i * 4] * 450
        const minX = predictionBoxes[i * 4 + 1] * 600
        const maxY = predictionBoxes[i * 4 + 2] * 450
        const maxX = predictionBoxes[i * 4 + 3] * 600
        const score = predictionScores[i * 3] * 100

        if (score > 75) {
          ctx.beginPath()
          ctx.rect(minX, minY, maxX - minX, maxY - minY)
          ctx.lineWidth = 3
          ctx.strokeStyle = 'red'
          ctx.fillStyle = 'red'
          ctx.stroke()
          ctx.shadowColor = 'white'
          ctx.shadowBlur = 10
          ctx.font = '14px Arial bold'
          ctx.fillText(
            `${score.toFixed(1)} - Jagermeister bottle`,
            minX,
            minY > 10 ? minY - 5 : 10
          )
        }
      }
    }

  },

  mounted () {
    this.streamPromise = this.initWebcamStream()
    this.loadModelAndDetection()
  }
}
</script>

<style lang="scss">
body {
  margin: 0;
}

.resultFrame {
  display: grid;

  video {
    grid-area: 1 / 1 / 2 / 2;
  }
  canvas {
    grid-area: 1 / 1 / 2 / 2;
  }
}
</style>
