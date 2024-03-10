<template>
  <div id="app">
    <div class="header">
      <h1>字幕识别</h1>
      <!-- 文件上传指定类型为video -->
      <button ref="btFV" @click="setFileVideo()">选择视屏文件</button><span>{{ videosText }}</span>
      <input ref="fileVideo" style="display: none;" type="file" accept="video/mp4*" @change="setVideo($event)" />

      <!-- 上传制定类型文件为image -->
      <button ref="btFI" @click="setFileImage()">选择图片文件</button><span>{{ imageText }}</span>
      <input ref="fileImage" type="file" style="display: none;" accept="image/*" @change="setImage($event)">
    </div>
    <button @click="tesser(0)">识别图片</button>
    <img ref="imageElement" alt="Vue logo" :src="images" />
    <video ref="videoElement" :src="videos" controls @play="tesser(1)" @ended="tesser(2)" @pause="tesser(2)"></video>
    <canvas ref="vc" style="display: none;"></canvas>
  </div>
</template>

<script setup lang="ts">
/* eslint-disable */

import { createWorker, PSM, OEM } from "tesseract.js";
import { ref } from "vue";


//   获取视屏元素
const videoElement = ref()

//设置视屏链接
let videos = ref("")

// 获取画布元素用于图片视屏截图
const vc = ref()


// 显示此时上传文件的状态
let videosText = ref("暂未上传视屏文件")

function setVideo(el: any) {
  // 实现视屏文件上传预览
  const reader = new FileReader()
  reader.readAsDataURL(el.target.files[0])
  reader.onload = async () => {
    videos.value = await reader.result + ""
    videosText.value = "已选择视屏文件"
  }
}

// 点击按钮触发视屏上传的点击事件
let fileVideo = ref()
function setFileVideo() {
  fileVideo.value.click()
}

// 获取图片元素
const imageElement = ref()

// 设置图片链接
let images = ref("")

// 获取上传视屏元素的按钮
const fileImage = ref()

// 设置图片元素是否选择
let imageText = ref("暂未选择图片文件")

function setImage(el: any) {
  // 实现图片文件上传预览
  const reader = new FileReader()
  reader.readAsDataURL(el.target.files[0])
  reader.onload = async () => {
    images.value = await reader.result + ""
    imageText.value = "已选择视屏文件"
  }
}


// 点击按钮触发上传图片的点击事件
function setFileImage() {
  fileImage.value.click()
}

// 创建一个文本转语音实例
const sps = new SpeechSynthesisUtterance()
// 设置语言
sps.lang = "CN-US" //汉语
sps.rate = 1  //语速

// 选择值开启或关闭视屏识别
let sL: boolean = false

// 选择用于判断视屏两次识别的结果是否相同
let btext: boolean = false

//识别
async function ded(ifNumber: number) {

  //设置值用于视屏识别判断值是否重复
  let texts: string = ""

  // 初始化
  var worker = await createWorker({
    workerPath: "/tesseract/tesseract.js/dist/worker.min.js",
    corePath: "/tesseract/tesseract.js-core/tesseract-core.wasm.js",
    langPath: "/tesseract/tesseract-lang",  // TODO：prd环境下会报错
  });

  await worker.loadLanguage("chi_sim");
  await worker.initialize("chi_sim", OEM.LSTM_ONLY);
  await worker.setParameters({
    tessedit_pageseg_mode: PSM.SINGLE_BLOCK
  })

  if (ifNumber == 0) {
    // 若传入值为0则表示当前调用为图片文字识别
    try {
      // 捕获错误
      var { data } = await worker.recognize(imageElement.value)

    } catch {
      // 识别失败报错
      sps.text = "识别失败，请检查文件是否包含文字或检查你的网络"
      window.speechSynthesis.speak(sps)
      // 发现错误停止函数
      return
    }

    // 将识别结果加入播放队列
    sps.text = data.text

    // 语音播放
    window.speechSynthesis.speak(sps)

  } else if (ifNumber == 1) {
    sL = true
    // 若传入值为1则表示视屏开始播放，开始识别字幕
    while (sL) {
      //画布获取视屏的宽高
      vc.value.height = videoElement.value.videoHeight
      vc.value.width = videoElement.value.videoWidth
      const ctx = vc.value.getContext("2d")
      ctx.drawImage(videoElement.value, 0, 0, vc.value.width, vc.value.height)

      //识别,且只识别屏幕下半部分
      var { data } = await worker.recognize(vc.value, {
        rectangle: {
          top: vc.value.height / 2,
          left: vc.value.width / 2,
          width: vc.value.width / 2,
          height: vc.value.height / 2
        }
      })
      // 如果这次的值与上次相同，不再执行加入播放队列任务
      data.text == texts ? btext = true : btext = false

      if (!btext) {
        // 将识别结果加入播放队列
        sps.text = data.text
        // 两次识别的结果不同，将此次识别的结果覆盖上次的结果
        texts = data.text
        // 语音播放
        window.speechSynthesis.speak(sps)
        // 等待语音播放队列播放完毕再进行判断
        await new Promise((re,en) => {
          let a = setInterval(() => {
            if(!window.speechSynthesis.pending){
              // 直到播放队列播放完毕才停止函数
              clearInterval(a)
            }
          },500)
        })
      }
    }

  } else if (ifNumber == 2) {
    //若传入值为2则表示视屏停止播放
    sL = false
    // 关闭播放队列
    window.speechSynthesis.cancel()
  }
}

const tesser = ded

</script>

<style></style>