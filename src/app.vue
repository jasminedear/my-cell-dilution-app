<script setup>
import { ref } from 'vue';
import Modal from './components/Modal.vue';
import DilutionCalculator from './components/DilutionCalculator.vue';
import PlateLogger from './components/PlateLogger.vue'; 

// 控制弹窗显示/隐藏的状态
const isCalculatorModalVisible = ref(false);
const isLoggerModalVisible = ref(false); 

function openCalculator() {
  isCalculatorModalVisible.value = true;
}
function closeCalculator() {
  isCalculatorModalVisible.value = false;
}

function openLogger() { 
    isLoggerModalVisible.value = true;
}
function closeLogger() { 
    isLoggerModalVisible.value = false;
}
</script>

<template>
  <div class="dashboard">
    <header class="app-header">
      <h1>🔬 细胞培养实验助手</h1>
      <p>欢迎使用 BV2 细胞管理和计算工具</p>
      <small class="developer-info">由 Holmes晓茜开发（湖南长沙）</small>
    </header>

    <main class="module-grid">
      <div class="module-card primary-module" @click="openCalculator">
        <h2>细胞稀释计算</h2>
        <p>输入细胞计数和目标参数，计算原液和培养基体积。</p>
        <button class="action-button">启动稀释计算器</button>
      </div>

      <div class="module-card secondary-module" @click="openLogger">
        <h2>孔板实验数据记录</h2>
        <p>设置孔板布局、处理组，并记录最终的检测读数。</p>
        <button class="action-button secondary-action-button">启动数据记录</button>
      </div>
      
      <div class="module-card">
        <h2>试剂配制助手</h2>
        <p>（待开发）计算摩尔浓度或百分比溶液配制。</p>
      </div>
    </main>
  </div>

  <Modal 
    :visible="isCalculatorModalVisible" 
    @close="closeCalculator"
  >
    <DilutionCalculator />
  </Modal>

  <Modal 
    :visible="isLoggerModalVisible" 
    @close="closeLogger"
  >
    <PlateLogger />
  </Modal>
</template>

<style>
/* ... 样式保持不变 ... */
body {
  margin: 0;
  background-color: #f4f7f6;
  font-family: Arial, sans-serif;
}

.dashboard {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

.app-header {
  text-align: center;
  margin-bottom: 40px;
  padding: 20px;
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.app-header h1 {
    color: #007bff;
    margin-bottom: 5px;
}

/* 【新增】开发者信息样式 */
.developer-info {
    display: block;
    margin-top: 10px;
    color: #888;
    font-size: 0.9em;
}

.module-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 25px;
}

.module-card {
  background-color: white;
  padding: 25px;
  border-radius: 8px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
  transition: transform 0.2s, box-shadow 0.2s;
  cursor: pointer;
}

.module-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 6px 15px rgba(0, 0, 0, 0.15);
}

.module-card h2 {
    color: #333;
    border-bottom: 2px solid #eee;
    padding-bottom: 10px;
}

.module-card p {
    color: #666;
    margin-bottom: 15px;
}

.primary-module {
    border: 2px solid #42b983;
}

.secondary-module {
    border: 2px solid #007bff;
}

.action-button {
    background-color: #42b983;
    color: white;
    border: none;
    padding: 10px 15px;
    border-radius: 4px;
    cursor: pointer;
    font-weight: bold;
}

.secondary-action-button {
    background-color: #007bff;
}
</style>