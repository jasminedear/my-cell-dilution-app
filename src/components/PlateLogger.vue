<template>
  <div class="plate-logger">
    <h2>📋 孔板实验数据记录</h2>
    
    <div class="settings-panel">
      <div class="input-group">
        <label for="plateType">板型</label>
        <select id="plateType" v-model.number="plateConfig.type">
          <option :value="96">96 孔板 (8x12)</option>
          <option :value="24">24 孔板 (4x6)</option>
          <option :value="12">12 孔板 (3x4)</option>
          <option :value="6">6 孔板 (2x3)</option>
        </select>
      </div>

      <div class="input-group">
        <label for="replicates">每组重复数 (n=)</label>
        <input type="number" id="replicates" v-model.number="plateConfig.replicates" min="1" max="12" />
      </div>

      <div class="input-group">
        <label for="experimentName">实验名称</label>
        <input type="text" id="experimentName" v-model="plateConfig.name" placeholder="例如: 药物A剂量曲线" />
      </div>
    </div>

    <div class="treatments-panel">
        <h4>处理组设置 (最多 {{ maxTreatments }} 组)</h4>
        <div class="treatment-list">
            <div v-for="(treatment, index) in treatments" :key="index" class="treatment-item">
                <span>{{ index + 1 }}.</span>
                <input type="text" v-model="treatment.name" placeholder="处理组名称 (例如: DMSO, 药物A)" />
                <input type="text" v-model="treatment.conc" placeholder="浓度 (例如: 10 uM)" />
                <button @click="removeTreatment(index)" class="remove-btn" v-if="treatments.length > 1">&times;</button>
            </div>
        </div>
        <button @click="addTreatment" :disabled="treatments.length >= maxTreatments" class="add-btn">+ 添加处理组</button>
    </div>

    <div class="plate-visualization">
      <h4>孔板布局预览 (行 x 列: {{ plateDimensions.rows }} x {{ plateDimensions.cols }})</h4>
      <div class="plate-grid" :style="gridStyle">
        <span class="header-cell"></span>
        <span v-for="col in plateDimensions.cols" :key="'col-' + col" class="header-cell">{{ col }}</span>

        <template v-for="row in plateDimensions.rows" :key="'row-' + row">
          <span class="header-cell">{{ getRowLetter(row) }}</span>
          <div 
            v-for="col in plateDimensions.cols" 
            :key="'well-' + getRowLetter(row) + col" 
            :class="['well-cell', 'treatment-' + getWellTreatmentIndex(row, col)]"
            :title="getWellTreatmentName(row, col)"
          >
            <input type="number" step="0.001" :placeholder="getWellShortName(row, col)" v-model.number="plateData[getWellShortName(row, col)]"/>
          </div>
        </template>
      </div>
    </div>

    <button @click="exportToCSV" class="export-btn">💾 导出数据到 CSV</button>

  </div>
</template>

<script setup>
import { reactive, computed } from 'vue';

// 配置状态
const plateConfig = reactive({
  type: 96, // 96, 24, 12, 6
  replicates: 3, // 每组重复数
  name: '',
});

// 处理组状态
const treatments = reactive([
  { name: 'Control (DMSO)', conc: '0' },
  { name: 'Test Drug 1', conc: '10 uM' },
  { name: 'Test Drug 2', conc: '50 uM' },
]);

// 存储输入的读数数据，键是 'A1', 'A2' 等
const plateData = reactive({});


// =========== 计算属性 (核心逻辑) ===========

// 定义最大处理组数（例如 96/3 = 32 组）
const maxTreatments = computed(() => Math.floor(plateConfig.type / plateConfig.replicates));

// 计算孔板的行列数
const plateDimensions = computed(() => {
  if (plateConfig.type === 96) return { rows: 8, cols: 12 }; // A-H, 1-12
  if (plateConfig.type === 24) return { rows: 4, cols: 6 }; // A-D, 1-6
  if (plateConfig.type === 12) return { rows: 3, cols: 4 }; // A-C, 1-4
  if (plateConfig.type === 6) return { rows: 2, cols: 3 }; // A-B, 1-3
  return { rows: 0, cols: 0 };
});

// CSS 样式控制网格布局
const gridStyle = computed(() => ({
  gridTemplateColumns: `1fr repeat(${plateDimensions.value.cols}, 4fr)`,
  gridTemplateRows: `1fr repeat(${plateDimensions.value.rows}, 4fr)`,
}));

// 获取行字母 (1 -> A, 2 -> B, ...)
function getRowLetter(row) {
  return String.fromCharCode(64 + row);
}

// 获取孔名 (例如：A1, C5)
function getWellShortName(row, col) {
  return `${getRowLetter(row)}${col}`;
}

// 根据孔的位置 (row, col) 计算它属于哪个处理组的索引 (0, 1, 2...)
function getWellTreatmentIndex(row, col) {
  // 孔的线性索引 (从 A1 开始)
  const rows = plateDimensions.value.rows;
  const cols = plateDimensions.value.cols;
  if (rows === 0 || cols === 0) return -1;
  
  const linearIndex = (row - 1) * cols + (col - 1);
  const treatmentIndex = Math.floor(linearIndex / plateConfig.replicates);
  
  return treatmentIndex % treatments.length; // 循环使用处理组
}

// 获取孔对应的处理组名称
function getWellTreatmentName(row, col) {
  const index = getWellTreatmentIndex(row, col);
  if (index >= 0 && index < treatments.length) {
    const t = treatments[index];
    return `处理组: ${t.name}\n浓度: ${t.conc}\n孔位: ${getRowLetter(row)}${col}`;
  }
  return `空/未分配`;
}

// =========== 方法 ===========

function addTreatment() {
  if (treatments.length < maxTreatments.value) {
    treatments.push({ name: '', conc: '' });
  }
}

function removeTreatment(index) {
  if (treatments.length > 1) {
    treatments.splice(index, 1);
  }
}

function exportToCSV() {
  // 1. 构建 CSV 头部 (孔名)
  let csv = 'Well,Treatment Name,Concentration,Readout\n';
  
  // 2. 遍历所有孔
  for (let r = 1; r <= plateDimensions.value.rows; r++) {
    for (let c = 1; c <= plateDimensions.value.cols; c++) {
      const wellName = getWellShortName(r, c);
      const treatmentIndex = getWellTreatmentIndex(r, c);
      const treatment = treatments[treatmentIndex];
      const readout = plateData[wellName] !== undefined ? plateData[wellName] : '';
      
      const treatmentName = treatment ? treatment.name : 'N/A';
      const concentration = treatment ? treatment.conc : 'N/A';

      csv += `${wellName},"${treatmentName}","${concentration}",${readout}\n`;
    }
  }

  // 3. 创建并下载文件
  const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
  const link = document.createElement("a");
  const url = URL.createObjectURL(blob);
  
  link.setAttribute("href", url);
  link.setAttribute("download", `${plateConfig.name || 'Experiment_Data'}_${plateConfig.type}well.csv`);
  link.style.visibility = 'hidden';
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}
</script>

<style scoped>
/* 容器样式 */
.plate-logger {
  min-width: 600px;
  max-width: 90vw;
}

/* 顶部设置 */
.settings-panel {
  display: flex;
  gap: 20px;
  margin-bottom: 20px;
  padding-bottom: 15px;
  border-bottom: 1px solid #eee;
}

.input-group label {
  display: block;
  font-weight: bold;
  margin-bottom: 5px;
}

.input-group input, .input-group select {
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
  width: 100%;
}

/* 处理组样式 */
.treatments-panel h4 {
    margin-bottom: 10px;
    color: #007bff;
}

.treatment-list {
    margin-bottom: 15px;
    max-height: 200px;
    overflow-y: auto;
}

.treatment-item {
    display: flex;
    gap: 10px;
    margin-bottom: 8px;
    align-items: center;
}

.treatment-item input {
    flex: 1;
    padding: 6px;
    border: 1px solid #ddd;
    border-radius: 3px;
}

.remove-btn {
    background: #ff4d4f;
    color: white;
    border: none;
    padding: 4px 8px;
    border-radius: 3px;
    cursor: pointer;
}

.add-btn {
    background: #42b983;
    color: white;
    border: none;
    padding: 8px 15px;
    border-radius: 4px;
    cursor: pointer;
}

/* 孔板网格可视化 */
.plate-visualization h4 {
    margin-top: 20px;
    margin-bottom: 10px;
}

.plate-grid {
  display: grid;
  border: 1px solid #ccc;
  border-radius: 5px;
  overflow: hidden;
  max-width: 100%;
}

.header-cell {
  background-color: #f0f0f0;
  font-weight: bold;
  padding: 5px;
  text-align: center;
  border-right: 1px solid #ccc;
  border-bottom: 1px solid #ccc;
  display: flex;
  align-items: center;
  justify-content: center;
}

.well-cell {
  border-right: 1px solid #eee;
  border-bottom: 1px solid #eee;
  padding: 5px;
  text-align: center;
}

.well-cell input {
    width: 90%;
    text-align: center;
    border: none;
    background: transparent;
    font-size: 0.9em;
}

/* 处理组颜色分配 (最多 10 组颜色) */
.treatment-0 { background-color: #e6ffe6; } /* 浅绿 */
.treatment-1 { background-color: #ffe6e6; } /* 浅红 */
.treatment-2 { background-color: #e6e6ff; } /* 浅蓝 */
.treatment-3 { background-color: #ffffcc; } /* 浅黄 */
.treatment-4 { background-color: #ffccff; } /* 浅紫 */
.treatment-5 { background-color: #ccffff; } /* 浅青 */
.treatment-6 { background-color: #f0f0f0; } /* 浅灰 */
.treatment-7 { background-color: #fff0e0; } /* 浅橙 */
.treatment-8 { background-color: #e0fff0; } /* 浅薄荷绿 */
.treatment-9 { background-color: #fff0ff; } /* 浅粉 */


.export-btn {
    width: 100%;
    background-color: #007bff;
    color: white;
    border: none;
    padding: 10px 15px;
    border-radius: 4px;
    cursor: pointer;
    margin-top: 20px;
}
</style>