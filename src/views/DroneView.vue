<script setup>
import { ref, watch, onBeforeUnmount } from 'vue'
import { ElDrawer, ElContainer, ElMain, ElFooter, ElMessage } from 'element-plus'
import ThreeScene from '@/components/ThreeScene.vue'
import GroundControls from '@/components/GroundControls.vue'
import CodeEditor from '@/components/CodeEditor.vue'
// 导入 AppHeader 组件
import AppHeader from '@/components/AppHeader.vue'
import { useSceneStore } from '@/stores/sceneStore'

const groundWidth = ref(2)
const groundDepth = ref(2)
const threeScene = ref(null)
const cvOutputContainer = ref(null)

// 添加抽屉控制变量
const drawerVisible = ref(false)

const isCustomPositionMode = ref(false)

// 获取 CodeEditor 组件的引用
const codeEditor = ref(null)

// 添加计时相关的状态
const isTimerRunning = ref(false)
const elapsedTime = ref(0)
let timerInterval = null

const sceneStore = useSceneStore()
const currentTexture = ref('')               // 保存当前地面纹理的 URL
const savedScenesDrawerVisible = ref(false)    // 控制保存场景抽屉的显示

function onUploadImage(file) {
  if (file) {
    if (file instanceof File) {
      currentTexture.value = URL.createObjectURL(file)
    } else if (typeof file === 'string') {
      currentTexture.value = file
    }
    threeScene.value.handleImageUpload(file)
  }
}

// 修改为两个不同的执行函数
function onExecuteCodeFromEditor(code) {
  // 从编辑器执行时，先关闭抽屉再执行
  drawerVisible.value = false
  
  // 等待抽屉完全关闭后执行代码
  setTimeout(() => {
    executeCode(code)
  }, 500)
}

function onExecuteCodeFromControls() {
  // 从控制栏执行时，直接执行当前代码
  if (codeEditor.value) {
    const currentCode = codeEditor.value.getCurrentCode()
    if (!currentCode?.trim()) {
      ElMessage.warning('请先编辑您的代码')
      drawerVisible.value = true
      return
    }
    executeCode(currentCode)
  } else {
    ElMessage.warning('请先打开代码编辑器输入代码')
    drawerVisible.value = true
  }
}

// 统一的代码执行函数
function executeCode(code) {
  threeScene.value.executeUserCode(code)
  startTimer()
}

// 当 ThreeScene 组件内部因图片更新而需要调整地面宽度时，会通过事件通知父组件
const updateGroundDimensions = (payload) => {
  groundWidth.value = payload.groundWidth
}

function handleCVOutput(canvas) {
  const containerDom = cvOutputContainer.value
  if (!containerDom) return

  while (containerDom.firstChild) {
    containerDom.removeChild(containerDom.firstChild)
  }
  containerDom.appendChild(canvas)
}

function handleCustomPosition() {
  if (threeScene.value) {
    threeScene.value.enterCustomPositionMode()
  }
}

// 计时器控制函数
function startTimer() {
  if (!isTimerRunning.value) {
    isTimerRunning.value = true
    elapsedTime.value = 0
    timerInterval = setInterval(() => {
      elapsedTime.value++
    }, 1000)
  }
}

function stopTimer() {
  if (isTimerRunning.value) {
    isTimerRunning.value = false
    clearInterval(timerInterval)
    timerInterval = null
  }
}

// 在组件卸载时清理计时器
onBeforeUnmount(() => {
  stopTimer()
})

// 修改停止代码的方法
function handleStopCode() {
  if (threeScene.value) {
    // 停止代码执行
    threeScene.value.executeUserCode('')  // 传入空代码来停止执行
    // 重置无人机位置
    threeScene.value.resetDronePosition()
  }
  // 停止计时器
  stopTimer()
  
  // 清空 CV 输出
  const containerDom = cvOutputContainer.value
  if (containerDom) {
    while (containerDom.firstChild) {
      containerDom.removeChild(containerDom.firstChild)
    }
  }
}

// 新增：保存当前场景到 pinia store
function saveCurrentScene() {
  sceneStore.addScene({
    groundWidth: groundWidth.value,
    groundDepth: groundDepth.value,
    texture: currentTexture.value
  })
  ElMessage.success('场景已保存')
}

// 新增：加载保存的场景
function loadScene(scene) {
  groundWidth.value = scene.groundWidth
  groundDepth.value = scene.groundDepth
  threeScene.value.loadSceneTexture(scene.texture)
  ElMessage.success('场景加载成功')
  savedScenesDrawerVisible.value = false
}

// 将计时状态传递给 GroundControls
</script>

<template>
  <el-container class="layout-container">
    <!-- 顶部导航栏 (使用 AppHeader 组件) -->
    <el-header>
      <AppHeader :drawer-visible="drawerVisible" @update:drawer-visible="drawerVisible = $event" />
    </el-header>

    <!-- 主体内容区域 -->
    <el-main class="main-area">
      <!-- 3D 场景及 CV 输出容器 -->
      <div class="scene-container-wrapper">
        <!-- 3D 场景组件 -->
        <ThreeScene :groundWidth="groundWidth" :groundDepth="groundDepth" ref="threeScene"
          @update-ground-dimensions="updateGroundDimensions" @cv-output="handleCVOutput"
          v-model:is-custom-position-mode="isCustomPositionMode" @save-scene="saveCurrentScene"
          @load-scene="savedScenesDrawerVisible = true" />
        <!-- CV 输出容器 (用于显示摄像头处理后的图像) -->
        <div ref="cvOutputContainer" class="floating-camera"></div>
      </div>
    </el-main>

    <!-- 底部控制栏 -->
    <el-footer height="auto" class="footer">
      <!-- 控制栏内容 -->
      <div class="footer-content">
        <!-- 地面控制组件 -->
        <GroundControls v-model:ground-width="groundWidth" v-model:ground-depth="groundDepth"
          v-model:is-custom-position-mode="isCustomPositionMode" :timer-running="isTimerRunning"
          :elapsed-time="elapsedTime" @custom-position="handleCustomPosition" @upload-image="onUploadImage"
          @execute-code="onExecuteCodeFromControls" @stop-code="handleStopCode" />
      </div>
    </el-footer>

    <!-- 代码编辑器抽屉 -->
    <el-drawer v-model="drawerVisible" title="代码编辑器" size="80%" direction="ttb">
      <CodeEditor ref="codeEditor" @execute-code="onExecuteCodeFromEditor" />
    </el-drawer>

    <!-- 新增：右侧抽屉展示已保存场景 -->
    <el-drawer
      v-model="savedScenesDrawerVisible"
      title="已保存场景"
      :with-header="true"
      direction="rtl"
      size="30%"
    >
      <div class="scene-list">
        <div 
          v-for="scene in sceneStore.scenes" 
          :key="scene.id" 
          class="scene-item"
          @click="loadScene(scene)"
        >
          <img :src="scene.texture" alt="场景预览" class="scene-preview" />
          <div>宽: {{ scene.groundWidth }} m, 深: {{ scene.groundDepth }} m</div>
        </div>
      </div>
    </el-drawer>
  </el-container>
</template>

<style scoped>
.layout-container {
  height: 100vh;
  display: flex;
  flex-direction: column;
}

.main-area {
  flex: 1;
  padding: 20px;
  overflow: hidden;
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
}

.footer {
  background-color: #fff;
  padding: 10px;
  /* border-top: 1px solid #dcdfe6; */
  box-shadow: 0 -4px 10px rgba(0, 0, 0, 0.1);
}

.footer-content {
  max-width: 800px;
  margin: 0 auto;
  display: flex;
  justify-content: center;
}

/* 抽屉样式 */
:deep(.el-drawer) {
  background: #1e1e1e;
}

:deep(.el-drawer__header) {
  margin-bottom: 0;
  padding: 16px 70px;
  background: #333333;
  color: white;
  font-weight: 600;
  text-align: center;
  height: 28px;
  position: relative; /* 添加相对定位 */
}

:deep(.el-drawer__close-btn) {
  position: absolute;
  right: 20px;
  top: 50%;
  transform: translateY(-50%);
  color: rgb(255, 255, 255);
  font-size: 20px;
  width: 32px;
  height: 32px;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
}

:deep(.el-drawer__close-btn:hover) {
  background-color: rgba(255, 255, 255, 0.1);
}

/* 添加浮动摄像头样式 */
.floating-camera {
  position: absolute;
  top: 20px;
  left: 20px;
  z-index: 10;
  width: 240px;
  height: 240px;
  aspect-ratio: 1;
  /* transition: all 0.3s ease; */
  backdrop-filter: blur(4px);
  background-color: rgba(150, 150, 150, 0.7);
  border: 2px solid #a3a3a3ae;
  border-radius: 3px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

/* 响应式样式统一管理 */
@media (max-width: 1200px) {
  .scene-container-wrapper {
    width: 80vw;
  }

  .floating-camera {
    width: 150px;
    height: 150px;
    aspect-ratio: 1;
  }
}

@media (max-width: 768px) {
  .scene-container-wrapper {
    width: 90vw;
    height: 50vh;
  }

  .floating-camera {
    width: 120px;
    height: 120px;
    aspect-ratio: 1;
  }

  .footer-content {
    max-width: 100%;
    padding: 0 10px;
  }
}

@media (max-width: 480px) {
  .scene-container-wrapper {
    width: 95vw;
    height: 45vh;
  }

  .floating-camera {
    width: 80px;
    height: 80px;
    aspect-ratio: 1;
  }

  .main-area {
    padding: 10px;
  }
}

.scene-container-wrapper {
  position: relative;
  width: 100%;
  height: 100%;
  min-height: 400px;
  border-radius: 12px;
  overflow: hidden;
  /* border: 1px solid rgba(0, 0, 0, 0.15); */
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1), 0 0 6px 0 rgba(0, 0, 0, 0.12);
  background-color: rgba(245, 245, 245, 0.03);
}

/* 新增：边缘模糊遮罩 */
.scene-container-wrapper::before {
  content: "";
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  /* 确保遮罩不影响鼠标事件 */
  z-index: 1;
  /* 确保在内容之上 */
  /* 径向渐变，从透明到完全模糊 */
  background: linear-gradient(to right, rgba(255, 255, 255, 0) 90%, rgba(255, 255, 255, 0.3) 95%, rgba(255, 255, 255, 1) 100%),
    linear-gradient(to bottom, rgba(255, 255, 255, 0) 90%, rgba(255, 255, 255, 0.3) 95%, rgba(255, 255, 255, 1) 100%),
    linear-gradient(to left, rgba(255, 255, 255, 0) 90%, rgba(255, 255, 255, 0.3) 95%, rgba(255, 255, 255, 1) 100%),
    linear-gradient(to top, rgba(255, 255, 255, 0) 90%, rgba(255, 255, 255, 0.3) 95%, rgba(255, 255, 255, 1) 100%);
}

/* 新增：已保存场景抽屉样式 */
.scene-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding: 10px;
}

.scene-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px;
  border: 1px solid #dcdfe6;
  border-radius: 4px;
  cursor: pointer;
}

.scene-item:hover {
  background-color: #f5f7fa;
}

.scene-preview {
  width: 60px;
  height: 60px;
  object-fit: cover;
  border-radius: 4px;
}
</style> 