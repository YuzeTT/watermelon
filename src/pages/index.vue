<script setup lang="ts">
import { computed, onUnmounted, reactive, ref } from 'vue'

// const activeNames = ref(['1'])
// const checked = ref(false)
// --- STATE MANAGEMENT ---
const massKg = ref<number>(4.5)
const isListening = ref<boolean>(false)
const error = ref<string | null>(null)
const canvasRef = ref<HTMLCanvasElement | null>(null)
const sensitivity = ref<number>(8) // NEW: Sensitivity state, default to 5
// function onChange(status: boolean) {
//   checked.value = status
// }
const result = reactive({
  baseFrequency: null as number | null,
  ripenessIndex: null as number | null,
  ripenessScore: null as number | null, // NEW: State for the score
  sweetnessBrix: null as number | null,
  verdict: '' as string,
  ripenessClass: '' as string,
})

// --- Web Audio API Context ---
let audioContext: AudioContext | null = null
let analyser: AnalyserNode | null = null
let animationFrameId: number | null = null
let mediaStream: MediaStream | null = null

// --- COMPUTED PROPERTIES ---
const isMassValid = computed(() => massKg.value > 0)
// NEW: Dynamic thresholds based on sensitivity
const absoluteThreshold = computed(() => 160 - (sensitivity.value * 10)) // From 150 (low sens) down to 60 (high sens)
const relativeThreshold = computed(() => 45 - (sensitivity.value * 3)) // From 42 (low sens) down to 15 (high sens)

// --- CORE LOGIC ---
async function startDetection() {
  if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
    error.value = 'Your browser doesn\'t support audio recording. Please use a modern browser like Chrome or Firefox.'
    return
  }

  // UPDATED: Reset previous results including the score
  Object.assign(result, { baseFrequency: null, ripenessIndex: null, ripenessScore: null, sweetnessBrix: null, verdict: '', ripenessClass: '' })
  error.value = null
  isListening.value = true

  try {
    mediaStream = await navigator.mediaDevices.getUserMedia({ audio: true, video: false })
    audioContext = new AudioContext()
    const source = audioContext.createMediaStreamSource(mediaStream)

    analyser = audioContext.createAnalyser()
    analyser.fftSize = 16384
    analyser.minDecibels = -70
    analyser.maxDecibels = -10
    analyser.smoothingTimeConstant = 0.5

    source.connect(analyser)

    detectThump()
  }
  catch (err) {
    error.value = `Could not access the microphone. Please grant permission and try again. ${err}`
    isListening.value = false
  }
}

function detectThump() {
  if (!analyser || !audioContext)
    return

  const bufferLength = analyser.frequencyBinCount
  const dataArray = new Uint8Array(bufferLength)

  const processAudio = () => {
    analyser!.getByteFrequencyData(dataArray)

    const sampleRate = audioContext!.sampleRate
    const relevantFreqMin = 100
    const relevantFreqMax = 300
    const minIndex = Math.round(relevantFreqMin / (sampleRate / analyser!.fftSize))
    const maxIndex = Math.round(relevantFreqMax / (sampleRate / analyser!.fftSize))

    let peakAmplitude = 0
    let peakIndex = 0
    let totalAmplitude = 0

    for (let i = minIndex; i < maxIndex; i++) {
      totalAmplitude += dataArray[i]
      if (dataArray[i] > peakAmplitude) {
        peakAmplitude = dataArray[i]
        peakIndex = i
      }
    }

    const averageAmplitude = totalAmplitude / (maxIndex - minIndex)

    // UPDATED: Use dynamic thresholds
    if (peakAmplitude > absoluteThreshold.value && peakAmplitude > averageAmplitude + relativeThreshold.value) {
      const baseFrequency = peakIndex * (sampleRate / analyser!.fftSize)
      calculateRipeness(baseFrequency)
      stopListening()
    }
    else {
      animationFrameId = requestAnimationFrame(processAudio)
    }

    drawVisualization(dataArray, minIndex, maxIndex)
  }

  animationFrameId = requestAnimationFrame(processAudio)
}

// UPDATED: Calculation function now includes the 0-10 score
function calculateRipeness(frequency: number) {
  if (massKg.value <= 0)
    return

  const m = massKg.value
  const f = frequency
  const ripenessIndex = f ** 2 * m ** (2 / 3)

  result.baseFrequency = f
  result.ripenessIndex = ripenessIndex

  // Verdict logic based on "Jingxin" variety study
  if (ripenessIndex < 74000) {
    result.verdict = '✅ 恰到好处'
    result.ripenessClass = 'ripe'
  }
  else if (ripenessIndex < 85000) {
    result.verdict = '🤔 听起来不太妙'
    result.ripenessClass = 'borderline'
  }
  else {
    result.verdict = '❌ 生瓜蛋子'
    result.ripenessClass = 'unripe'
  }

  // Estimate sweetness based on the graph in Wang Shumo's paper
  const sweetness = 12 - 4 * ((ripenessIndex - 60000) / 30000)
  result.sweetnessBrix = Math.max(5, Math.min(13, sweetness))

  // NEW: Calculate 0-10 Score
  // We map the ripeness index to a 0-10 score. Lower index = higher score.
  // Let's define a "perfect" score of 10 at index 60,000 and a score of 0 at index 95,000.
  const ripeThreshold = 60000
  const unripeThreshold = 95000
  const score = 100 * (unripeThreshold - ripenessIndex) / (unripeThreshold - ripeThreshold)

  // Clamp the score between 0 and 10
  result.ripenessScore = Math.max(0, Math.min(100, score))
}

function stopListening() {
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId)
    animationFrameId = null
  }
  mediaStream?.getTracks().forEach(track => track.stop())
  audioContext?.close().catch(() => {})
  isListening.value = false
}

function drawVisualization(dataArray: Uint8Array, minIndex: number, maxIndex: number) {
  const canvas = canvasRef.value
  if (!canvas)
    return

  const ctx = canvas.getContext('2d')
  if (!ctx)
    return

  ctx.fillStyle = 'rgb(245, 245, 245)'
  ctx.fillRect(0, 0, canvas.width, canvas.height)

  ctx.lineWidth = 2
  ctx.strokeStyle = 'rgb(50, 50, 50)'
  ctx.beginPath()

  const sliceWidth = (canvas.width * 1.0) / (maxIndex - minIndex)
  let x = 0

  for (let i = minIndex; i < maxIndex; i++) {
    const v = dataArray[i] / 256.0
    const y = v * canvas.height

    if (i === 0) {
      ctx.moveTo(x, canvas.height - y)
    }
    else {
      ctx.lineTo(x, canvas.height - y)
    }

    x += sliceWidth
  }

  ctx.lineTo(canvas.width, canvas.height)
  ctx.stroke()
}

onUnmounted(() => {
  stopListening()
})
</script>

<template>
  <div bg-zinc-100>
    <div p-4 space-y-4>
      <div>
        <div flex gap-2 items-center>
          <div text-lg font-bold rounded-lg bg-white flex h-8 w-8 items-center justify-center>
            🍉
          </div>
          <div text-lg font-bold>
            生瓜蛋子鉴定器
          </div>

          <div flex-1 />

          <!-- <el-check-tag :checked="checked" @change="onChange">
            专业模式
          </el-check-tag> -->
        </div>
      </div>

      <el-card shadow="never" w-auto>
        <div>
          <div mb-4>
            <div font-bold mb-1>
              预估重量
            </div>
            <div text-xs op50>
              输入可以获得更精确的计算结果，麒麟瓜大多数在4.5kg上下
            </div>
          </div>

          <el-input v-model.number="massKg" type="number" :disabled="isListening" size="large" w-full>
            <template #append>
              Kg
            </template>
          </el-input>
        </div>
        <div my-6 />

        <div>
          <div mb-4>
            <div font-bold mb-1>
              设置灵敏度
            </div>
            <div text-xs op50>
              普通环境 6-7, 嘈杂环境 3-4, 安静环境 12-15
            </div>
          </div>

          <el-slider v-model="sensitivity" :min="1" :max="20" :disabled="isListening" show-input />
        </div>
        <div my-6 />
        <div>
          <div mb-4>
            <div font-bold mb-1>
              点击开始 靠近西瓜 轻轻拍打
            </div>
            <div text-xs op50>
              手机麦克风放在拍打区域旁，不接触西瓜
            </div>
          </div>

          <el-button type="success" size="large" w-full :disabled="isListening || !isMassValid" @click="startDetection">
            开始鉴瓜
          </el-button>
        </div>
      </el-card>

      <el-card shadow="never" w-auto>
        <div v-if="result.sweetnessBrix" mb-3 text-center>
          声学鉴定结果
        </div>
        <div v-if="result.sweetnessBrix">
          <div text-5xl font-bold text-center :class="result.ripenessClass" class="score-value">
            {{ result.ripenessScore?.toFixed(1) }}
          </div>
          <div text-center op50>
            此瓜评分 / 100
          </div>
          <div my-5 />
        </div>
        <canvas ref="canvasRef" width="400" height="100" />
        <div v-if="!result.sweetnessBrix && isListening">
          <div text-xs mt-2 op75>
            正在捕获音频，请持续拍打西瓜。捕获到频谱数据后自动结束开始分析，无法触发请调高灵敏度哦~
          </div>
        </div>
        <div v-if="result.sweetnessBrix">
          <div my-5 />
          <div gap-4 grid grid-cols-2>
            <div>
              <div text-sm mb-1 op75>
                综合评定
              </div>
              <div text-xl font-bold>
                {{ result.verdict }}
              </div>
            </div>

            <div>
              <div text-sm mb-1 op75>
                预估甜度
              </div>
              <div text-xl font-bold>
                {{ result.sweetnessBrix ? `~${result.sweetnessBrix.toFixed(1)}% Brix` : 'N/A' }}
              </div>
            </div>
            <div>
              <div text-sm mb-1 op75>
                基础频率
              </div>
              <div text-xl font-bold>
                {{ result.baseFrequency ? `${result.baseFrequency.toFixed(0)} Hz` : 'N/A' }}
              </div>
            </div>
            <div>
              <div text-sm mb-1 op75>
                成熟度指数
              </div>
              <div text-xl font-bold>
                {{ result.ripenessIndex ? result.ripenessIndex.toFixed(0) : 'N/A' }}
              </div>
            </div>
          </div>
        </div>
        <div my-5 />

        <div v-if="result.sweetnessBrix" flex gap-2 items-center>
          <div>
            <img src="/water.png" alt="" h-15 w-15>
          </div>
          <div>
            <div font-bold>
              生瓜蛋子鉴定器🍉
            </div>
            <div>watermelon.hsott.cn</div>
          </div>
        </div>
      </el-card>

      <el-card shadow="never" w-auto>
        <div mb-4>
          <div font-bold mb-1>
            特别鸣谢 & 参考文献
          </div>
          <div text-xs op75>
            衷心感谢以下文献为这项工具研发注入的智慧力量，其学术成果为技术实现提供核心支撑，谨致谢忱。
          </div>
        </div>
        <div text-xs>
          <div>
            [1]王书茂,焦群英,籍俊杰.西瓜成熟度无损检验的冲击振动方法[J].农业工程学报,1999,15(3):241-245
          </div>
          <div mt-0.5>
            [2]汪迪松,浦宏杰,李臻峰,等. 基于声振法的西瓜内部糖度检测研究[J]. 上海农业学报,2017,33(2):125-130. DOI:10.15955/j.issn1000-3924.2017.02.23.
          </div>
          <div mt-0.5>
            [3]陆勇,李臻峰,李静,等. 声学技术在西瓜品质无损检测中的应用[J]. 福建农业科技,2014(10):77-81. DOI:10.3969/j.issn.0253-2301.2014.10.031.
          </div>
        </div>
      </el-card>
    </div>
  </div>
</template>

<style scoped>
canvas {
  width: 100%;
  background: #f5f5f5;
  border-radius: 4px;
  border: 1px solid #eee;
}

/* NEW: Styles for the score */
.score-display {
  text-align: center;
  margin-bottom: 1.5rem;
}
.score-value {
  font-size: 4rem;
  font-weight: bold;
  line-height: 1;
}
.score-value.ripe {
  color: #4caf50;
}
.score-value.borderline {
  color: #ffc107;
}
.score-value.unripe {
  color: #f44336;
}
</style>
