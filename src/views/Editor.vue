<template>
  <div class="container" v-resize="onContainerResize">
    <Row style="margin: 4px 0">
    <Upload 
      :before-upload="handleUpload"
      action="/"
      :format="['jpg','jpeg','png']"
      :max-size="4096"
      style="display: inline-block; margin-right: 4px">
      <Button icon="ios-cloud-upload-outline">Select image file to detect</Button>
    </Upload>
    <Select v-model="model" style="width:100px; margin-right: 4px">
        <Option v-for="item in models" :value="item.value">{{ item.key }}</Option>
    </Select>
    <ButtonGroup>
      <Button type="primary" icon="md-folder-open" @click="detect">Detect</Button>
     </ButtonGroup>
  </Row>
  <div class="upload-img">
      <img id="srcImg" :src="imgUrl">
      <canvas id="overlay" />
  </div>
  </div>
</template>

<style scoped>
.container {
  overflow: hidden;
  height: 100%;
}

#scene {
  width: 100%;
  height: 100%;
  background-color: #fff;
}

.menu {
  position: absolute;
  top: 0;
  left: 0;
  overflow: visible;
  text-align: left;
  margin-left: 40px;
  margin-top: 10px;
}

.ivu-dropdown {
  margin-left: 10px;
}

.upload-img{
  display: inline-block;
  width: 640px;
  height: 480px;
  text-align: center;
  line-height: 480px;
  border: 1px solid transparent;
  border-radius: 4px;
  overflow: hidden;
  background: #fff;
  position: relative;
  box-shadow: 0 1px 1px rgba(0,0,0,.2);
  margin-right: 4px;
}

.upload-img img{
  width: 100%;
  height: 100%;
  object-fit: contain;
}

.upload-img canvas {
  position: absolute;
  top: 0;
  left: 0;
}
</style>

<script>
  import resize from 'vue-resize-directive'
  import FileSaver from 'file-saver'
  import LocalForage from "localforage"
  import * as faceapi from 'face-api.js'
import { async } from 'q';

  console.log(faceapi)

  export default {
    components: {},
    data: function () {
      return {
        size: {
          width: 0,
          height: 0
        },
        scene: {
          canvas: null
        },
        imgFile: null,
        imgUrl: '/img/default.jpeg',
        modelParams: {
          minConfidence: 0.5,
          inputSize: 512,
          scoreThreshold: 0.5,
          minFaceSize: 20
        },
        model: 'ssd_mobilenetv1',
        models: [
          {
            key: 'SSD',
            value: 'ssd_mobilenetv1'
          },
          {
            key: 'TINY',
            value: 'tiny_face_detector'
          },
          {
            key: 'MTCNN',
            value: 'mtcnn'
          }
        ]
      }
    },
    directives: {
      resize,
    },
    computed: {
    },
    methods: {
      onContainerResize() {
        this.size.width = this.$el.clientWidth
        this.size.height = this.$el.clientHeight
      },
      handleUpload (file) {
        this.imgFile = file
        this.imgUrl = window.URL.createObjectURL(file)
        this.clearCanvas()
        return false
      },
      clearCanvas() {
        let ctx = this.scene.canvas.getContext("2d")
        ctx.clearRect(0, 0, 640, 480)
      },
      async detectFace (model){
         let options = null
        if (model === 'ssd_mobilenetv1') {
          if (!faceapi.nets.ssdMobilenetv1.params) {
            await faceapi.loadSsdMobilenetv1Model('/model')
          }
          options = new faceapi.SsdMobilenetv1Options({ 
            minConfidence: this.modelParams.minConfidence 
          })
        } else if (model === 'tiny_face_detector') {
          if (!faceapi.nets.tinyFaceDetector.params) {
            await faceapi.loadTinyFaceDetectorModel('/model')
          }
          options = new faceapi.TinyFaceDetectorOptions({ 
            inputSize : this.modelParams.inputSize, 
            scoreThreshold: this.modelParams.scoreThreshold
          })

        } else if (model === 'mtcnn') {
          if (!faceapi.nets.mtcnn.params) {
            await faceapi.loadMtcnnModel('/model')
          }
          options = new faceapi.MtcnnOptions({ 
            minFaceSize: this.modelParams.minFaceSize
          })
        }
        
        const inputImg = this.$el.querySelector('#srcImg')
        const results = await faceapi.detectAllFaces(inputImg, options)
        this.drawDetections(results)
      },
      drawDetections(results){
        this.clearCanvas()
        const resizedDetections = faceapi.resizeResults(results, { width: 640, height: 480 })
        let ctx = this.scene.canvas.getContext("2d")
        resizedDetections.forEach((fd, i) => {
          ctx.strokeStyle="#ff0000"
          ctx.lineWidth = 4
          ctx.strokeRect(fd.box.left, fd.box.top, fd.box.width, fd.box.height)
        })
      },
      detect () {
        this.detectFace(this.model)
      }
    },
    mounted: function () {
      // 随窗口动态改变大小
      this.size.width = this.$el.clientWidth
      this.size.height = this.$el.clientHeight

      this.scene.canvas = this.$el.querySelector('#overlay')
      this.scene.canvas.width = 640
      this.scene.canvas.height = 480
    }
  }
</script>
