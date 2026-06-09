<template>
  <div class="container">
    <section class="page-shell">
      <el-card class="page-card" shadow="never">
        <template #header>
          <header class="card-header">
            <div>
              <h1 class="page-title">多模态控制系统演示</h1>
              <p class="page-subtitle">
                集成语音、手势与眼动三种交互方式，兼顾桌面端与移动端体验。
              </p>
            </div>
          </header>
        </template>

        <section class="control-panel">
          <el-row :gutter="16" class="control-grid">
            <el-col :xs="24" :sm="12" :lg="8">
              <el-card shadow="hover" class="control-card">
                <div class="control-card-top">
                  <h3 class="control-title">语音控制</h3>
                  <span class="control-badge" :class="{ 'control-badge-active': voiceActive }">
                    {{ voiceActive ? "运行中" : "待启动" }}
                  </span>
                </div>
                <p class="control-tech">Web Speech API</p>
                <el-button class="control-action" :type="voiceActive ? 'danger' : 'primary'" @click="toggleVoice">
                  {{ voiceActive ? "停止录音" : "开始录音" }}
                </el-button>
                <p class="control-desc">
                  支持指令："打开弹窗"、"关闭弹窗"、"变红"、"变蓝"
                </p>
                <div class="status">
                  识别结果：{{ voiceText || "等待识别" }}
                </div>
              </el-card>
            </el-col>

            <el-col :xs="24" :sm="12" :lg="8">
              <el-card shadow="hover" class="control-card">
                <div class="control-card-top">
                  <h3 class="control-title">手势控制</h3>
                  <span class="control-badge" :class="{ 'control-badge-active': handActive }">
                    {{ handActive ? "运行中" : "待启动" }}
                  </span>
                </div>
                <p class="control-tech">HandTrack.js</p>
                <el-button class="control-action" :type="handActive ? 'danger' : 'primary'" @click="toggleHand">
                  {{ handActive ? "停止追踪" : "开始追踪" }}
                </el-button>
                <p class="control-desc">
                  检测到手部变绿，靠近镜头时切换为黄色高亮提示
                </p>
                <div class="status">检测状态：{{ handStatus }}</div>
                <video ref="handVideo" muted playsinline style="display: none"></video>
              </el-card>
            </el-col>

            <el-col :xs="24" :sm="24" :lg="8">
              <el-card shadow="hover" class="control-card">
                <div class="control-card-top">
                  <h3 class="control-title">眼动控制</h3>
                  <span class="control-badge" :class="{ 'control-badge-active': gazeActive }">
                    {{ gazeActive ? "运行中" : "待启动" }}
                  </span>
                </div>
                <p class="control-tech">WebGazer.js</p>
                <el-button class="control-action" :type="gazeActive ? 'danger' : 'primary'" @click="toggleGaze">
                  {{ gazeActive ? "停止眼动" : "开始眼动" }}
                </el-button>
                <p class="control-desc">
                  注视屏幕不同区域，观察视线坐标与图标联动
                </p>
                <div class="gaze-helper">
                  <span class="gaze-helper-text">
                    {{
            hasSeenGazeGuide
              ? "可随时重新查看权限与校准说明"
              : "首次使用建议先阅读权限与校准说明"
          }}
                  </span>
                  <el-button link type="primary" @click="openGazeGuide">
                    {{ hasSeenGazeGuide ? "查看引导" : "首次引导" }}
                  </el-button>
                </div>
                <div class="status">
                  状态：{{ gazeStatus }}<br />
                  坐标：{{ gazeX }}, {{ gazeY }}
                </div>
                <div v-if="gazeCalibrating" class="gaze-progress">
                  校准中：请注视{{ currentCalibrationPoint.label }}， 第
                  {{ gazeCalibrationIndex + 1 }} /
                  {{ GAZE_CALIBRATION_POINTS.length }} 个点
                </div>
                <el-button v-if="gazeInitialized" class="gaze-recalibrate" text type="primary"
                  @click="startCalibrationGuide">
                  重新校准
                </el-button>
              </el-card>
            </el-col>
          </el-row>
        </section>

        <el-divider />

        <section class="demo-section">
          <div class="demo-area" :style="{ backgroundColor: demoColor }">
            <div class="demo-copy">
              <span class="demo-label">实时反馈区</span>
              <h2 class="demo-title">当前交互效果</h2>
              <p class="demo-text">
                语音会控制弹窗与背景色，手势会切换区域状态，眼动会驱动图标位置。
              </p>
            </div>
            <div class="gaze-target" :style="{ left: gazeX + 'px', top: gazeY + 'px' }">
              👀
            </div>
            <div v-if="gazeCalibrating" class="gaze-calibration-panel">
              <div class="gaze-calibration-copy">
                <span class="gaze-calibration-label">
                  校准步骤 {{ gazeCalibrationIndex + 1 }} /
                  {{ GAZE_CALIBRATION_POINTS.length }}
                </span>
                <p class="gaze-calibration-text">
                  请注视{{ currentCalibrationPoint.label }}约 1
                  秒，保持头部尽量稳定。
                </p>
              </div>
              <div class="gaze-calibration-point" :style="{
            left: currentCalibrationPoint.x,
            top: currentCalibrationPoint.y,
          }"></div>
            </div>
          </div>
        </section>
      </el-card>
    </section>

    <el-dialog v-model="dialogVisible" title="语音控制弹窗" width="min(92vw, 420px)">
      <span>你通过语音指令打开了这个弹窗！</span>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="dialogVisible = false">关闭</el-button>
        </span>
      </template>
    </el-dialog>

    <el-dialog v-model="gazeGuideVisible" title="眼动权限与校准提示" width="min(92vw, 480px)">
      <div class="gaze-guide-dialog">
        <p class="gaze-guide-intro">首次启动眼动控制前，请先确认以下事项：</p>
        <ul class="gaze-guide-list">
          <li>点击“允许”授予浏览器摄像头权限。</li>
          <li>保持面部在画面中，环境光线尽量均匀。</li>
          <li>启动后请依次注视屏幕上的五个校准点。</li>
          <li>建议优先使用 `localhost` 或 HTTPS 地址访问页面。</li>
        </ul>
      </div>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="gazeGuideVisible = false">我知道了，稍后再试</el-button>
          <el-button type="primary" @click="confirmGazeGuide">
            继续启动眼动
          </el-button>
        </span>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from "vue";
import { ElMessage } from "element-plus";
import * as handTrack from "handtrackjs";
import webgazerScriptUrl from "webgazer/dist/webgazer.js?url";

// --- State ---
const demoColor = ref("#f0f2f5");
const dialogVisible = ref(false);

// --- Voice Control ---
const voiceActive = ref(false);
const voiceText = ref("");
let recognition = null;

const initVoice = () => {
  const SpeechRecognition =
    window.SpeechRecognition || window.webkitSpeechRecognition;
  if (!SpeechRecognition) {
    ElMessage.error("您的浏览器不支持 Web Speech API");
    return;
  }
  recognition = new SpeechRecognition();
  recognition.lang = "zh-CN";
  recognition.continuous = true;
  recognition.interimResults = false;

  recognition.onresult = (event) => {
    const lastResult = event.results[event.results.length - 1];
    const text = lastResult[0].transcript.trim();
    voiceText.value = text;

    if (text.includes("打开弹窗")) {
      dialogVisible.value = true;
    } else if (text.includes("关闭弹窗")) {
      dialogVisible.value = false;
    } else if (text.includes("变红")) {
      demoColor.value = "#ffcccc";
    } else if (text.includes("变蓝")) {
      demoColor.value = "#ccccff";
    }
  };

  recognition.onend = () => {
    if (voiceActive.value) {
      recognition.start(); // Restart to keep listening
    }
  };
};

const toggleVoice = () => {
  if (!recognition) initVoice();
  if (voiceActive.value) {
    voiceActive.value = false;
    recognition.stop();
  } else {
    voiceActive.value = true;
    recognition.start();
  }
};

// --- Hand Control ---
const handActive = ref(false);
const handStatus = ref("未开始");
const handVideo = ref(null);
let model = null;
let handInterval = null;
let handStream = null;

const HAND_DETECTION_INTERVAL = 400;
const HAND_MODEL_PARAMS = {
  flipHorizontal: true,
  maxNumBoxes: 1,
  scoreThreshold: 0.6,
  iouThreshold: 0.5,
};

const getHandErrorMessage = (error) => {
  const errorName = error?.name || "";

  if (
    errorName === "NotAllowedError" ||
    errorName === "PermissionDeniedError"
  ) {
    return "摄像头权限被拒绝，请在浏览器中允许访问摄像头";
  }

  if (errorName === "NotFoundError" || errorName === "DevicesNotFoundError") {
    return "未检测到可用摄像头，请检查设备";
  }

  if (errorName === "NotReadableError" || errorName === "TrackStartError") {
    return "摄像头正在被其他应用占用，请先关闭占用程序";
  }

  if (
    errorName === "OverconstrainedError" ||
    errorName === "ConstraintNotSatisfiedError"
  ) {
    return "当前设备不支持所需的摄像头参数";
  }

  return error?.message || "无法启动摄像头";
};

const stopHandDetection = () => {
  if (handInterval) {
    clearInterval(handInterval);
    handInterval = null;
  }
};

const stopHandCamera = () => {
  stopHandDetection();

  if (handStream) {
    handStream.getTracks().forEach((track) => track.stop());
    handStream = null;
  }

  if (handVideo.value) {
    handVideo.value.pause();
    handVideo.value.srcObject = null;
  }
};

const prepareHandVideo = () => {
  if (!handVideo.value) {
    throw new Error("未找到手势视频容器");
  }

  handVideo.value.width = 640;
  handVideo.value.height = 480;
  handVideo.value.muted = true;
  handVideo.value.autoplay = true;
  handVideo.value.playsInline = true;
  handVideo.value.setAttribute("muted", "true");
  handVideo.value.setAttribute("autoplay", "true");
  handVideo.value.setAttribute("playsinline", "true");
};

const startHandCamera = async () => {
  if (!window.isSecureContext) {
    throw new Error("手势控制需要在 HTTPS 或 localhost 环境下访问摄像头");
  }

  if (!navigator.mediaDevices?.getUserMedia) {
    throw new Error("当前浏览器不支持摄像头访问");
  }

  prepareHandVideo();

  handStream = await navigator.mediaDevices.getUserMedia({
    audio: false,
    video: {
      facingMode: "user",
      width: { ideal: 640 },
      height: { ideal: 480 },
    },
  });

  handVideo.value.srcObject = handStream;

  await new Promise((resolve) => {
    handVideo.value.onloadedmetadata = () => {
      resolve();
    };
  });

  await handVideo.value.play();
};

const updateHandStatus = async () => {
  const predictions = await model.detect(handVideo.value);

  if (!predictions.length) {
    handStatus.value = "未检测到手部，请将手移入镜头范围";
    return;
  }

  const primaryHand = predictions[0];
  const [x = 0, y = 0, width = 0, height = 0] = primaryHand.bbox || [];
  const handAreaRatio =
    (width * height) /
    ((handVideo.value?.videoWidth || 1) * (handVideo.value?.videoHeight || 1));
  const confidence = Math.round((primaryHand.score || 0) * 100);

  if (handAreaRatio > 0.18) {
    demoColor.value = "#fff4cc";
    handStatus.value = `检测到近距离手部，置信度 ${confidence}%`;
    return;
  }

  demoColor.value = "#ccffcc";
  handStatus.value = `检测到手部，位置 (${Math.round(x)}, ${Math.round(
    y
  )})，置信度 ${confidence}%`;
};

const toggleHand = async () => {
  if (handActive.value) {
    handActive.value = false;
    stopHandCamera();
    handStatus.value = "已停止";
    return;
  }

  try {
    handActive.value = true;
    handStatus.value = "加载手势模型中...";

    if (!model) {
      model = await handTrack.load(HAND_MODEL_PARAMS);
    }

    handStatus.value = "启动摄像头中...";
    await startHandCamera();
    handStatus.value = "正在追踪手部...";

    stopHandDetection();
    handInterval = setInterval(async () => {
      try {
        await updateHandStatus();
      } catch (error) {
        handStatus.value = "手势识别中断，请重新启动";
        console.error(error);
      }
    }, HAND_DETECTION_INTERVAL);
  } catch (error) {
    handActive.value = false;
    stopHandCamera();
    handStatus.value = getHandErrorMessage(error);
    ElMessage.error(handStatus.value);
    console.error(error);
  }
};

// --- Eye Tracking Control ---
const gazeActive = ref(false);
const gazeX = ref(window.innerWidth / 2);
const gazeY = ref(window.innerHeight / 2);
const gazeStatus = ref("未开始");
const gazeGuideVisible = ref(false);
const hasSeenGazeGuide = ref(false);
const gazeCalibrating = ref(false);
const gazeCalibrationIndex = ref(0);

const GAZE_GUIDE_STORAGE_KEY = "zero-threshold-gaze-guide-seen";
const GAZE_CALIBRATION_POINTS = [
  { label: "左上角", x: "12%", y: "14%" },
  { label: "右上角", x: "88%", y: "14%" },
  { label: "屏幕中央", x: "50%", y: "50%" },
  { label: "左下角", x: "12%", y: "84%" },
  { label: "右下角", x: "88%", y: "84%" },
];

let webgazerApi = null;
let gazeInitialized = false;
let gazeCalibrationTimer = null;
let webgazerLoadPromise = null;

const currentCalibrationPoint = ref(GAZE_CALIBRATION_POINTS[0]);

const loadWebgazerScript = () => {
  if (window.webgazer) {
    return Promise.resolve(window.webgazer);
  }

  if (webgazerLoadPromise) {
    return webgazerLoadPromise;
  }

  webgazerLoadPromise = new Promise((resolve, reject) => {
    const existingScript = document.querySelector(
      'script[data-webgazer-script="true"]'
    );

    if (existingScript) {
      existingScript.addEventListener(
        "load",
        () => {
          if (window.webgazer) {
            resolve(window.webgazer);
            return;
          }
          webgazerLoadPromise = null;
          reject(new Error("WebGazer 脚本加载完成，但未找到可用实例"));
        },
        { once: true }
      );
      existingScript.addEventListener(
        "error",
        () => {
          webgazerLoadPromise = null;
          reject(new Error("WebGazer 脚本加载失败"));
        },
        { once: true }
      );
      return;
    }

    const script = document.createElement("script");
    script.src = webgazerScriptUrl;
    script.async = true;
    script.dataset.webgazerScript = "true";
    script.onload = () => {
      if (window.webgazer) {
        resolve(window.webgazer);
        return;
      }
      webgazerLoadPromise = null;
      reject(new Error("WebGazer 脚本加载完成，但未找到可用实例"));
    };
    script.onerror = () => {
      webgazerLoadPromise = null;
      reject(new Error("WebGazer 脚本加载失败"));
    };
    document.head.appendChild(script);
  });

  return webgazerLoadPromise;
};

const getWebgazer = async () => {
  if (webgazerApi) return webgazerApi;
  webgazerApi = await loadWebgazerScript();
  return webgazerApi;
};

const loadGazeGuideState = () => {
  hasSeenGazeGuide.value =
    window.localStorage.getItem(GAZE_GUIDE_STORAGE_KEY) === "true";

  if (!hasSeenGazeGuide.value) {
    gazeStatus.value = "首次使用请先阅读权限与校准提示";
  }
};

const markGazeGuideSeen = () => {
  hasSeenGazeGuide.value = true;
  window.localStorage.setItem(GAZE_GUIDE_STORAGE_KEY, "true");
};

const getGazeErrorMessage = (error) => {
  const errorName = error?.name || "";

  if (
    errorName === "NotAllowedError" ||
    errorName === "PermissionDeniedError"
  ) {
    return "摄像头权限被拒绝，请允许眼动模块访问摄像头";
  }

  if (errorName === "NotFoundError" || errorName === "DevicesNotFoundError") {
    return "未检测到可用于眼动追踪的摄像头";
  }

  if (errorName === "NotReadableError" || errorName === "TrackStartError") {
    return "摄像头正在被其他应用占用，暂时无法启动眼动追踪";
  }

  return error?.message || "眼动模块启动失败";
};

const updateGazePoint = (data) => {
  if (!data) {
    return;
  }

  gazeX.value = Math.round(data.x);
  gazeY.value = Math.round(data.y);
};

const applyWebgazerDisplay = (webgazer, visible) => {
  webgazer.showVideoPreview(visible);
  webgazer.showPredictionPoints(false);
};

const clearGazeCalibrationTimer = () => {
  if (gazeCalibrationTimer) {
    clearTimeout(gazeCalibrationTimer);
    gazeCalibrationTimer = null;
  }
};

const finishCalibrationGuide = () => {
  clearGazeCalibrationTimer();
  gazeCalibrating.value = false;

  if (gazeActive.value) {
    gazeStatus.value = "校准完成，请缓慢移动视线观察跟踪效果";
  }
};

const runCalibrationGuide = () => {
  currentCalibrationPoint.value =
    GAZE_CALIBRATION_POINTS[gazeCalibrationIndex.value];
  gazeStatus.value = `校准中：请注视${currentCalibrationPoint.value.label}`;

  gazeCalibrationTimer = setTimeout(() => {
    if (gazeCalibrationIndex.value >= GAZE_CALIBRATION_POINTS.length - 1) {
      finishCalibrationGuide();
      return;
    }

    gazeCalibrationIndex.value += 1;
    runCalibrationGuide();
  }, 1200);
};

const startCalibrationGuide = () => {
  if (!gazeInitialized) {
    gazeStatus.value = "请先启动眼动控制，再进行校准";
    return;
  }

  clearGazeCalibrationTimer();
  gazeCalibrating.value = true;
  gazeCalibrationIndex.value = 0;
  runCalibrationGuide();
};

const openGazeGuide = () => {
  gazeGuideVisible.value = true;
};

const startGazeTracking = async () => {
  if (!window.isSecureContext) {
    throw new Error("眼动控制需要在 HTTPS 或 localhost 环境下运行");
  }

  const webgazer = await getWebgazer();

  if (!webgazer.detectCompatibility()) {
    throw new Error("当前浏览器不支持 WebGazer 眼动追踪");
  }

  webgazer
    .setGazeListener((data) => {
      updateGazePoint(data);
    })
    .saveDataAcrossSessions(false);

  if (!gazeInitialized) {
    gazeStatus.value = "请求摄像头权限中...";
    await webgazer.begin();
    gazeInitialized = true;
  } else {
    gazeStatus.value = "恢复眼动追踪中...";
    await webgazer.resume();
  }

  applyWebgazerDisplay(webgazer, false);
  gazeStatus.value = "正在追踪视线";
};

const stopGazeTracking = async () => {
  clearGazeCalibrationTimer();
  gazeCalibrating.value = false;

  if (!webgazerApi || !gazeInitialized) {
    gazeActive.value = false;
    gazeStatus.value = "已停止";
    return;
  }

  webgazerApi.pause();
  applyWebgazerDisplay(webgazerApi, false);
  webgazerApi.clearGazeListener();
  gazeActive.value = false;
  gazeStatus.value = "已停止";
};

const activateGazeTracking = async () => {
  gazeActive.value = true;
  await startGazeTracking();
  startCalibrationGuide();
};

const confirmGazeGuide = async () => {
  gazeGuideVisible.value = false;
  markGazeGuideSeen();

  try {
    await activateGazeTracking();
  } catch (error) {
    gazeActive.value = false;
    gazeStatus.value = getGazeErrorMessage(error);
    ElMessage.error(gazeStatus.value);
    console.error(error);
  }
};

const toggleGaze = async () => {
  if (gazeActive.value) {
    await stopGazeTracking();
    return;
  }

  if (!hasSeenGazeGuide.value) {
    openGazeGuide();
    return;
  }

  try {
    await activateGazeTracking();
  } catch (error) {
    gazeActive.value = false;
    gazeStatus.value = getGazeErrorMessage(error);
    ElMessage.error(gazeStatus.value);
    console.error(error);
  }
};

onMounted(() => {
  initVoice();
  loadGazeGuideState();
});

onUnmounted(() => {
  if (recognition) recognition.stop();
  stopHandCamera();
  clearGazeCalibrationTimer();
  if (webgazerApi && gazeInitialized) {
    webgazerApi.clearGazeListener();
    webgazerApi.end();
  }
});
</script>

<style scoped>
.container {
  min-height: 100vh;
  padding: 24px 16px 40px;
  background: radial-gradient(circle at top,
      rgba(64, 158, 255, 0.16),
      transparent 28%),
    linear-gradient(180deg, #f7faff 0%, #eef3fb 100%);
  box-sizing: border-box;
}

.page-shell {
  width: 100%;
  max-width: 1200px;
  margin: 0 auto;
}

.page-card {
  border: 0;
  border-radius: 24px;
  background: rgba(255, 255, 255, 0.88);
  box-shadow: 0 18px 40px rgba(15, 23, 42, 0.08);
  backdrop-filter: blur(12px);
}

.card-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.page-title {
  margin: 0;
  font-size: 30px;
  font-weight: 700;
  line-height: 1.2;
  color: #18222f;
}

.page-subtitle {
  margin: 10px 0 0;
  font-size: 15px;
  line-height: 1.7;
  color: #5b6472;
}

.control-panel {
  margin-top: 8px;
}

.control-card {
  height: 100%;
  border: 1px solid rgba(64, 158, 255, 0.12);
  border-radius: 20px;
  transition: transform 0.2s ease, box-shadow 0.2s ease, border-color 0.2s ease;
}

.control-card:hover {
  transform: translateY(-4px);
  border-color: rgba(64, 158, 255, 0.28);
  box-shadow: 0 14px 30px rgba(64, 158, 255, 0.12);
}

.control-card-top {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 12px;
}

.control-title {
  margin: 0;
  font-size: 20px;
  font-weight: 600;
  line-height: 1.4;
  color: #18222f;
}

.control-badge {
  flex-shrink: 0;
  padding: 4px 10px;
  border-radius: 999px;
  font-size: 12px;
  font-weight: 600;
  line-height: 1.4;
  color: #667085;
  background: #f2f4f7;
}

.control-badge-active {
  color: #0c7a43;
  background: #dcfae6;
}

.control-tech {
  margin: 10px 0 0;
  font-size: 13px;
  font-weight: 600;
  letter-spacing: 0.04em;
  color: #409eff;
  text-transform: uppercase;
}

.control-action {
  width: 100%;
  margin-top: 18px;
}

.control-desc {
  min-height: 44px;
  margin: 16px 0 0;
  font-size: 14px;
  line-height: 1.7;
  color: #5b6472;
}

.status {
  margin-top: 16px;
  padding: 12px 14px;
  border-radius: 14px;
  font-size: 14px;
  font-weight: 600;
  line-height: 1.6;
  color: #344054;
  background: #f6f9fc;
  word-break: break-word;
}

.gaze-helper {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  margin-top: 14px;
}

.gaze-helper-text {
  font-size: 13px;
  line-height: 1.6;
  color: #667085;
}

.gaze-progress {
  margin-top: 12px;
  font-size: 13px;
  line-height: 1.6;
  color: #409eff;
}

.gaze-recalibrate {
  margin-top: 4px;
  padding-left: 0;
}

.demo-section {
  margin-top: 8px;
}

.demo-area {
  position: relative;
  display: flex;
  align-items: flex-start;
  min-height: 380px;
  padding: 28px;
  border-radius: 24px;
  overflow: hidden;
  transition: background-color 0.3s ease;
  box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.35);
}

.demo-copy {
  position: relative;
  z-index: 1;
  max-width: 420px;
  padding: 20px;
  border-radius: 20px;
  background: rgba(255, 255, 255, 0.72);
  backdrop-filter: blur(10px);
}

.demo-label {
  display: inline-flex;
  padding: 6px 12px;
  border-radius: 999px;
  font-size: 12px;
  font-weight: 700;
  line-height: 1;
  color: #409eff;
  background: rgba(64, 158, 255, 0.14);
}

.demo-title {
  margin: 14px 0 0;
  font-size: 28px;
  font-weight: 700;
  line-height: 1.25;
  color: #18222f;
}

.demo-text {
  margin: 12px 0 0;
  font-size: 15px;
  line-height: 1.8;
  color: #4b5563;
}

.gaze-target {
  position: absolute;
  top: 50%;
  left: 50%;
  z-index: 2;
  font-size: 40px;
  transform: translate(-50%, -50%);
  pointer-events: none;
  transition: all 0.1s ease;
}

.gaze-calibration-panel {
  position: absolute;
  inset: 0;
  z-index: 3;
  background: rgba(15, 23, 42, 0.12);
}

.gaze-calibration-copy {
  position: absolute;
  right: 24px;
  bottom: 24px;
  max-width: 320px;
  padding: 14px 16px;
  border-radius: 16px;
  background: rgba(255, 255, 255, 0.9);
  box-shadow: 0 10px 24px rgba(15, 23, 42, 0.12);
}

.gaze-calibration-label {
  display: inline-flex;
  font-size: 12px;
  font-weight: 700;
  line-height: 1;
  color: #409eff;
}

.gaze-calibration-text {
  margin: 10px 0 0;
  font-size: 14px;
  line-height: 1.7;
  color: #344054;
}

.gaze-calibration-point {
  position: absolute;
  width: 22px;
  height: 22px;
  border: 4px solid rgba(255, 255, 255, 0.95);
  border-radius: 50%;
  background: #409eff;
  box-shadow: 0 0 0 10px rgba(64, 158, 255, 0.2);
  transform: translate(-50%, -50%);
  animation: calibration-pulse 1.2s infinite;
}

.gaze-guide-dialog {
  padding-top: 4px;
}

.gaze-guide-intro {
  margin: 0;
  font-size: 14px;
  line-height: 1.7;
  color: #344054;
}

.gaze-guide-list {
  margin: 12px 0 0;
  padding-left: 18px;
  color: #4b5563;
}

.gaze-guide-list li {
  margin-bottom: 8px;
  line-height: 1.7;
}

@keyframes calibration-pulse {
  0% {
    transform: translate(-50%, -50%) scale(0.92);
  }

  50% {
    transform: translate(-50%, -50%) scale(1);
  }

  100% {
    transform: translate(-50%, -50%) scale(0.92);
  }
}

@media (max-width: 1024px) {
  .page-title {
    font-size: 26px;
  }

  .demo-area {
    min-height: 340px;
  }
}

@media (max-width: 768px) {
  .container {
    padding: 16px 12px 28px;
  }

  .page-card {
    border-radius: 20px;
  }

  .page-title {
    font-size: 24px;
  }

  .page-subtitle {
    font-size: 14px;
  }

  .control-title {
    font-size: 18px;
  }

  .control-desc {
    min-height: auto;
  }

  .demo-area {
    min-height: 300px;
    padding: 20px;
  }

  .gaze-calibration-copy {
    right: 20px;
    bottom: 20px;
    max-width: 280px;
  }

  .demo-copy {
    max-width: 100%;
    padding: 16px;
  }

  .demo-title {
    font-size: 22px;
  }
}

@media (max-width: 480px) {
  .card-header {
    display: block;
  }

  .page-title {
    font-size: 22px;
  }

  .control-card-top {
    flex-direction: column;
    align-items: flex-start;
  }

  .control-badge {
    margin-top: 2px;
  }

  .gaze-helper {
    align-items: flex-start;
    flex-direction: column;
  }

  .demo-area {
    min-height: 260px;
    padding: 16px;
  }

  .gaze-calibration-copy {
    right: 16px;
    bottom: 16px;
    left: 16px;
    max-width: none;
  }

  .demo-title {
    font-size: 20px;
  }

  .demo-text,
  .status {
    font-size: 13px;
  }

  .gaze-target {
    font-size: 32px;
  }
}
</style>
