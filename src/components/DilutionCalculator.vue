<template>
  <div class="dilution-calculator">
    <h2>🔬 BV2 细胞稀释计算器</h2>

    <div class="input-section">
      <label for="countReading">计数仪读数 (Cells/mL)</label>
      <input
        id="countReading"
        type="text"
        v-model="input.countReadingStr"
        placeholder="例如: 8.5E5 或 850000"
        inputmode="decimal"
        @blur="normalizeOnBlur('countReadingStr')"
      />
      <small v-if="input.countReadingStr" class="hint">
        预览：<span v-html="formatSciString(input.countReadingStr)"></span> Cells/mL
      </small>

      <label for="trypanBlueFactor">台盼蓝稀释倍数 (例如: 2)</label>
      <input
        id="trypanBlueFactor"
        type="number"
        v-model.number="input.trypanBlueFactor"
        placeholder="通常为 2"
        min="1"
        step="1"
      />

      <hr />

      <label for="targetConcentration">🎯 目标浓度 (Cells/mL)</label>
      <input
        id="targetConcentration"
        type="text"
        v-model="input.targetConcentrationStr"
        placeholder="例如: 1E5 或 100000"
        inputmode="decimal"
        @blur="normalizeOnBlur('targetConcentrationStr')"
      />
      <small v-if="input.targetConcentrationStr" class="hint">
        预览：<span v-html="formatSciString(input.targetConcentrationStr)"></span> Cells/mL
      </small>

      <label for="targetTotalVolume">💧 目标总体积 (mL)</label>
      <input
        id="targetTotalVolume"
        type="number"
        v-model.number="input.targetTotalVolume"
        placeholder="例如: 10"
        min="0.1"
        step="0.1"
      />
    </div>

    <div class="results-section">
      <button @click="calculate">开始计算</button>

      <div v-if="result.message" :class="['message', { warning: !result.success }]">
        {{ result.message }}
      </div>

      <div v-if="result.success" class="output-data">
        <p>
          ✅ 原液浓度 (C1):
          <span v-html="formatNumberDisplay(result.originalConcentration)"></span>
          Cells/mL
        </p>

        <h3>➡️ 需取原液体积 (V1): {{ volumeFormatter(result.volumeOriginalSuspension) }}</h3>

        <p>➕ 需加培养基体积: {{ volumeFormatter(result.volumeMedium) }}</p>

        <p class="note">
          配制总量: {{ (result.volumeOriginalSuspension + result.volumeMedium).toFixed(3) }} mL
        </p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { reactive } from 'vue'

/* ============ 输入/结果模型 ============ */
const input = reactive({
  countReadingStr: '8.5E5',
  trypanBlueFactor: 2,
  targetConcentrationStr: '1E5',
  targetTotalVolume: 10
})

const result = reactive({
  success: false,
  message: '',
  originalConcentration: 0,
  volumeOriginalSuspension: 0,
  volumeMedium: 0
})

/* ============ 数字格式化（结果区） ============ */
/** 显示为：X.XX × 10^y（大数/很小数），否则整数千分位 */
function formatNumberDisplay(num) {
  if (typeof num !== 'number' || !isFinite(num) || num === 0) return '0'
  const abs = Math.abs(num)
  if (abs >= 10000 || (abs > 0 && abs <= 0.0001)) {
    const exp = num.toExponential(2) // 2 位小数有效数字，例如 8.50e+5
    const [baseStr, powStr] = exp.split('e')
    const base = parseFloat(baseStr).toFixed(2)
    const power = parseInt(powStr, 10)
    return `${base} &times; 10<sup>${power}</sup>`
  }
  return Math.round(num).toLocaleString()
}

/* ============ 体积单位格式化 ============ */
function volumeFormatter(volumeInMl) {
  if (volumeInMl < 1) {
    const ul = volumeInMl * 1000
    return `${ul.toFixed(1)} \u03bcL`
  }
  return `${volumeInMl.toFixed(3)} mL`
}

/* ============ 解析/预览/规范化 ============ */
/** 支持：8.5E5 / 8.5e5 / 8.5×10^5 / 8.5 * 10^5 / 8.5 x 10^5 / 850000 / 850,000 */
function parseSci(str) {
  if (typeof str !== 'string') return NaN
  const s = str.trim()
    .replace(/，/g, ',')     // 中文逗号
    .replace(/\s+/g, '')     // 去所有空白
    .replace(/[×*]/g, 'x')   // 统一乘号
    .toLowerCase()

  // 1) e 记法：如 8.5e5
  if (/^-?\d*\.?\d+e[+-]?\d+$/.test(s)) {
    return Number(s)
  }

  // 2) 乘方记法：a x 10^b 或 a x 10b（容错）
  const m = s.match(/^(-?\d*\.?\d+)x10\^?([+-]?\d+)$/)
         || s.match(/^(-?\d*\.?\d+)x10([+-]?\d+)$/)
  if (m) {
    const a = parseFloat(m[1])
    const b = parseInt(m[2], 10)
    return a * Math.pow(10, b)
  }

  // 3) 纯数字（含千分位）
  const plain = s.replace(/,/g, '')
  return Number(plain)
}

/** 用于“预览”的美化：把任意写法转成漂亮的 HTML（与结果区一致） */
function formatSciString(str) {
  const n = parseSci(str)
  if (!isFinite(n)) return ''
  return formatNumberDisplay(n)
}

/** 把数值转成输入框的“规范文本”
 *  规则：
 *   - |n| >= 1e4 或 |n| <= 1e-3 → 科学计数法（3 位有效数字），例如 8.50E5
 *   - 否则 → 普通数字（四舍五入为整数），不加千分位，便于再次解析
 */
function numberToCanonicalInput(n) {
  if (!isFinite(n)) return ''
  const abs = Math.abs(n)
  if (abs >= 1e4 || (abs > 0 && abs <= 1e-3)) {
    // 3 位有效数字，E 大写，指数去掉前导 +
    const expStr = n.toExponential(3).replace('e+', 'E').replace('e-', 'E-')
    return expStr
  }
  return String(Math.round(n))
}

/** 失焦时规范化指定字段（countReadingStr / targetConcentrationStr） */
function normalizeOnBlur(fieldName) {
  const raw = input[fieldName]
  if (raw == null || String(raw).trim() === '') return
  const n = parseSci(String(raw))
  if (isFinite(n) && n > 0) {
    input[fieldName] = numberToCanonicalInput(n)
  }
  // 如果解析失败或非正数，就保持原样（也可清空/提示，根据需要改）
}

/* ============ 核心计算 ============ */
function calculateCellDilution(data) {
  const countReading = parseSci(data.countReadingStr)
  const targetConcentration = parseSci(data.targetConcentrationStr)
  const trypanBlueFactor = data.trypanBlueFactor
  const targetTotalVolume = data.targetTotalVolume

  if (
    isNaN(countReading) ||
    isNaN(targetConcentration) ||
    countReading <= 0 ||
    trypanBlueFactor <= 0 ||
    targetConcentration <= 0 ||
    targetTotalVolume <= 0
  ) {
    return {
      success: false,
      message: '输入值无效或小于零，请检查输入格式（例如：8.5E5）。',
      originalConcentration: 0
    }
  }

  // 原液浓度 = 计数读数 × 染料稀释倍数
  const originalConcentration = countReading * trypanBlueFactor

  if (originalConcentration < targetConcentration) {
    return {
      success: false,
      message: `⚠ 警告：原液浓度 (${formatNumberDisplay(originalConcentration)}) Cells/mL 低于目标浓度，无法稀释！`,
      originalConcentration
    }
  }

  // C1 * V1 = C2 * V2
  const volumeOriginalSuspension = (targetConcentration * targetTotalVolume) / originalConcentration
  const volumeMedium = targetTotalVolume - volumeOriginalSuspension

  return {
    success: true,
    message: '计算成功，请按体积配制。',
    originalConcentration,
    volumeOriginalSuspension,
    volumeMedium
  }
}

function calculate() {
  const r = calculateCellDilution(input)
  result.success = r.success
  result.message = r.message
  result.originalConcentration = r.originalConcentration || 0
  result.volumeOriginalSuspension = r.success ? r.volumeOriginalSuspension : 0
  result.volumeMedium = r.success ? r.volumeMedium : 0
}
</script>

<style scoped>
.dilution-calculator {
  max-width: 480px;
  margin: 20px auto;
  padding: 25px;
  border: 1px solid #e0e0e0;
  border-radius: 12px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
  background-color: #ffffff;
}

h2 {
  color: #333;
  border-bottom: 2px solid #42b983;
  padding-bottom: 10px;
  margin-bottom: 20px;
  font-size: 1.5em;
}

.input-section label {
  display: block;
  font-weight: bold;
  margin-top: 15px;
  margin-bottom: 5px;
  color: #555;
}

.input-section input {
  width: 95%;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 6px;
  font-size: 1em;
}

hr {
  margin: 20px 0;
  border: none;
  border-top: 1px dashed #eee;
}

button {
  width: 100%;
  padding: 12px;
  background-color: #42b983;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  margin-top: 20px;
  font-size: 1.1em;
  transition: background-color 0.3s;
}

button:hover {
  background-color: #368a6a;
}

.results-section {
  margin-top: 25px;
  padding-top: 15px;
}

.message {
  padding: 12px;
  margin-top: 15px;
  border-radius: 6px;
  font-weight: bold;
  background-color: #e6f7ff;
  color: #1890ff;
  border: 1px solid #91d5ff;
}

.warning {
  background-color: #fff1f0;
  color: #ff4d4f;
  border: 1px solid #ffa39e;
}

.output-data {
  margin-top: 15px;
  padding: 15px;
  background-color: #f7f7f7;
  border-radius: 8px;
}

.output-data h3 {
  color: #007bff;
  margin: 10px 0;
  font-size: 1.4em;
}

.hint {
  display: block;
  margin-top: 4px;
  color: #666;
  font-size: 0.85em;
}

.note {
  font-size: 0.85em;
  color: #888;
  margin-top: 10px;
}
</style>
