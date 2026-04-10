<template>
  <div class="container">
    <el-card class="box-card">
      <template #header>
        <div class="card-header">
          <span>多模态控制系统演示</span>
        </div>
      </template>
      <div class="control-panel">
        <el-row :gutter="20">
          <el-col :span="8">
            <el-card shadow="hover">
              <h3>语音控制 (Web Speech API)</h3>
              <el-button type="primary" @click="toggleVoice" :type="voiceActive ? 'danger' : 'primary'">
                {{ voiceActive ? '停止录音' : '开始录音' }}
              </el-button>
              <p>指令: "打开弹窗", "关闭弹窗", "变红", "变蓝"</p>
              <div class="status">识别结果: {{ voiceText }}</div>
            </el-card>
          </el-col>
          <el-col :span="8">
            <el-card shadow="hover">
              <h3>手势控制 (HandTrack.js)</h3>
              <el-button type="primary" @click="toggleHand" :type="handActive ? 'danger' : 'primary'">
                {{ handActive ? '停止追踪' : '开始追踪' }}
              </el-button>
              <p>动作: 张开手变绿，握拳变黄</p>
              <div class="status">检测状态: {{ handStatus }}</div>
              <video ref="handVideo" style="display:none"></video>
            </el-card>
          </el-col>
          <el-col :span="8">
            <el-card shadow="hover">
              <h3>眼动控制 (WebGazer.js)</h3>
              <el-button type="primary" @click="toggleGaze" :type="gazeActive ? 'danger' : 'primary'">
                {{ gazeActive ? '停止眼动' : '开始眼动' }}
              </el-button>
              <p>动作: 注视屏幕左侧或右侧以移动方块</p>
              <div class="status">坐标: {{ gazeX }}, {{ gazeY }}</div>
            </el-card>
          </el-col>
        </el-row>
      </div>
      
      <el-divider />
      
      <div class="demo-area" :style="{ backgroundColor: demoColor }">
        <h2>效果演示区</h2>
        <div class="gaze-target" :style="{ left: gazeX + 'px', top: gazeY + 'px' }">👀</div>
      </div>
    </el-card>

    <el-dialog v-model="dialogVisible" title="语音控制弹窗">
      <span>你通过语音指令打开了这个弹窗！</span>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="dialogVisible = false">关闭</el-button>
        </span>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import { ElMessage } from 'element-plus'
import * as handTrack from 'handtrackjs'

// --- State ---
const demoColor = ref('#f0f2f5')
const dialogVisible = ref(false)

// --- Voice Control ---
const voiceActive = ref(false)
const voiceText = ref('')
let recognition = null

const initVoice = () => {
  const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition
  if (!SpeechRecognition) {
    ElMessage.error('您的浏览器不支持 Web Speech API')
    return
  }
  recognition = new SpeechRecognition()
  recognition.lang = 'zh-CN'
  recognition.continuous = true
  recognition.interimResults = false

  recognition.onresult = (event) => {
    const lastResult = event.results[event.results.length - 1]
    const text = lastResult[0].transcript.trim()
    voiceText.value = text
    
    if (text.includes('打开弹窗')) {
      dialogVisible.value = true
    } else if (text.includes('关闭弹窗')) {
      dialogVisible.value = false
    } else if (text.includes('变红')) {
      demoColor.value = '#ffcccc'
    } else if (text.includes('变蓝')) {
      demoColor.value = '#ccccff'
    }
  }

  recognition.onend = () => {
    if (voiceActive.value) {
      recognition.start() // Restart to keep listening
    }
  }
}

const toggleVoice = () => {
  if (!recognition) initVoice()
  if (voiceActive.value) {
    voiceActive.value = false
    recognition.stop()
  } else {
    voiceActive.value = true
    recognition.start()
  }
}

// --- Hand Control ---
const handActive = ref(false)
const handStatus = ref('未开始')
const handVideo = ref(null)
let model = null
let handInterval = null

const toggleHand = async () => {
  if (handActive.value) {
    handActive.value = false
    handTrack.stopVideo(handVideo.value)
    clearInterval(handInterval)
    handStatus.value = '已停止'
  } else {
    handActive.value = true
    handStatus.value = '加载模型中...'
    if (!model) {
      model = await handTrack.load()
    }
    handStatus.value = '启动摄像头...'
    handTrack.startVideo(handVideo.value).then(status => {
      if (status) {
        handStatus.value = '正在追踪'
        handInterval = setInterval(async () => {
          const predictions = await model.detect(handVideo.value)
          if (predictions.length > 0) {
            const hand = predictions[0]
            if (hand.label === 'open') {
              demoColor.value = '#ccffcc'
              handStatus.value = '检测到：张开手'
            } else if (hand.label === 'closed') {
              demoColor.value = '#ffffcc'
              handStatus.value = '检测到：握拳'
            }
          }
        }, 500)
      } else {
        handStatus.value = '无法启动摄像头'
        handActive.value = false
      }
    })
  }
}

// --- Eye Tracking Control ---
const gazeActive = ref(false)
const gazeX = ref(window.innerWidth / 2)
const gazeY = ref(window.innerHeight / 2)

const toggleGaze = () => {
  if (gazeActive.value) {
    gazeActive.value = false
    window.webgazer.pause()
    // 隐藏 WebGazer 元素
    document.getElementById('webgazerVideoContainer')?.setAttribute('style', 'display: none !important')
  } else {
    gazeActive.value = true
    if (!window.webgazer.isReady) {
      window.webgazer.setGazeListener((data, elapsedTime) => {
        if (data == null) return
        gazeX.value = data.x
        gazeY.value = data.y
      }).begin()
      // 首次启动时默认显示视频以供校准，我们这里简单隐藏
      setTimeout(() => {
        document.getElementById('webgazerVideoContainer')?.setAttribute('style', 'display: none !important')
      }, 1000)
    } else {
      window.webgazer.resume()
      document.getElementById('webgazerVideoContainer')?.setAttribute('style', 'display: none !important')
    }
  }
}

onMounted(() => {
  initVoice()
})

onUnmounted(() => {
  if (recognition) recognition.stop()
  if (handActive.value) handTrack.stopVideo(handVideo.value)
  if (handInterval) clearInterval(handInterval)
  if (window.webgazer) window.webgazer.end()
})
</script>

<style scoped>
.container {
  padding: 20px;
  max-width: 1200px;
  margin: 0 auto;
}
.card-header {
  font-size: 20px;
  font-weight: bold;
}
.status {
  margin-top: 10px;
  font-weight: bold;
  color: #666;
}
.demo-area {
  height: 400px;
  border-radius: 8px;
  display: flex;
  justify-content: center;
  align-items: center;
  transition: background-color 0.3s ease;
  position: relative;
  overflow: hidden;
}
.gaze-target {
  position: absolute;
  font-size: 40px;
  transform: translate(-50%, -50%);
  pointer-events: none;
  transition: all 0.1s ease;
}
</style>
