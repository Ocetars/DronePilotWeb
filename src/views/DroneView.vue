<script setup>
import { ref, watch, onBeforeUnmount, onMounted } from 'vue'
import { ElDrawer, ElContainer, ElMain, ElFooter, ElMessage, ElMessageBox } from 'element-plus'
import ThreeScene from '@/components/ThreeScene.vue'
import GroundControls from '@/components/GroundControls.vue'
import CodeEditor from '@/components/CodeEditor.vue'
// 导入 AppHeader 组件
import AppHeader from '@/components/AppHeader.vue'
import { useSceneStore } from '@/stores/sceneStore'
import { useAuthStore } from '@/stores/authStore'
import { Delete, RefreshLeft } from '@element-plus/icons-vue'
import { useUser } from '@clerk/vue'

const groundWidth = ref(4)
const groundDepth = ref(4)
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

const authStore = useAuthStore()

const { isSignedIn, user, isLoaded } = useUser()

// 添加一个新的 ref 来控制按钮状态
const isRefreshingScenes = ref(false)

// 监听用户状态变化
watch([isLoaded, isSignedIn, user], () => {
  if (isLoaded.value) {
    if (isSignedIn.value && user.value) {
      authStore.setUser(user.value)
      // 如果用户已登录，加载场景
      loadScenesAfterLogin()
    } else {
      authStore.clearUser()
      console.log('用户未登录')
    }
  }
})

// 将场景加载逻辑抽取为单独的函数
const loadScenesAfterLogin = async () => {
  try {
    console.log('第一次尝试加载场景')
    await sceneStore.fetchScenes()
  } catch (error) {
    console.log('第一次加载失败，3秒后重试')
    setTimeout(async () => {
      try {
        console.log('第二次尝试加载场景')
        await sceneStore.fetchScenes()
      } catch (retryError) {
        console.log('第二次加载失败')
        ElMessage.error({
          message: '同步场景列表失败，请检查国际网络环境',
          offset: 100
        })
      }
    }, 3000)
  }
}

function onUploadImage(file) {
  if (file) {
    if (file instanceof File) {
      // 将文件转换为 base64
      const reader = new FileReader();
      reader.onload = (e) => {
        currentTexture.value = e.target.result;
        threeScene.value.handleImageUpload(e.target.result);
      };
      reader.readAsDataURL(file);
    } else if (typeof file === 'string' && file.startsWith('data:image')) {
      // 如果已经是 base64 格式，直接使用
      currentTexture.value = file;
      threeScene.value.handleImageUpload(file);
    }
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

// 修改 onMounted 钩子
onMounted(() => {
  // 不需要手动调用 setUser，watch 会处理用户状态变化
  if (isLoaded.value && isSignedIn.value && user.value) {
    authStore.setUser(user.value)
    setTimeout(() => {
      loadScenesAfterLogin()
    }, 2000)
  }
})

// 修改：保存当前场景到 pinia store
async function saveCurrentScene() {
  if (!authStore.isLoggedIn) {
    ElMessage.warning({
      message: '登录账号以保存场景',
      offset: 100
    })
    return
  }

  // 在弹出对话框之前，确保 currentTexture 已赋值
  if (!currentTexture.value && threeScene.value && threeScene.value.getDefaultTextureData) {
    currentTexture.value = threeScene.value.getDefaultTextureData() || '';
  }

  // 生成默认场景名称
  const defaultSceneName = `我的场景${sceneStore.scenes.length + 1}`

  // 弹出对话框，让用户输入场景名称
  try {
    const { value: sceneName } = await ElMessageBox.prompt('', '保存场景', {
      confirmButtonText: '保存',
      cancelButtonText: '取消',
      inputPattern: /.+/,
      inputErrorMessage: '场景名称不能为空',
      inputValue: defaultSceneName,
      customClass: 'scene-save-dialog',
      inputPlaceholder: '  请输入场景名称',
      roundButton: true,
      // 自定义对话框样式
      customStyle: {
        padding: '20px',
        borderRadius: '8px'
      }
    })

    if (sceneName) {
      await sceneStore.addScene({
        name: sceneName,
        groundWidth: groundWidth.value,
        groundDepth: groundDepth.value,
        texture: currentTexture.value
      })
    }
  } catch (error) {
    // 用户取消了输入
  }
}

// 修改：加载保存的场景
async function loadScene(scene) {
  if (!authStore.isLoggedIn) {
    ElMessage.warning('请先登录')
    return
  }
  groundWidth.value = scene.groundWidth
  groundDepth.value = scene.groundDepth
  threeScene.value.loadSceneTexture(scene.texture)
  ElMessage.success({
    message: '场景加载成功',
    offset: 100
  })
  savedScenesDrawerVisible.value = false
}

// 添加删除场景的方法
async function handleDeleteScene(sceneId) {
  try {
    await ElMessageBox.confirm(
      '确定要删除这个场景吗？',
      '警告',
      {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning',
      }
    )
    await sceneStore.removeScene(sceneId)
    ElMessage.success({
      message: '场景已删除',
      offset: 100
    })
  } catch (error) {
    if (error !== 'cancel') {
      ElMessage.error({
        message: '删除场景失败,您点击的太快了',
        offset: 100
      })
    }
  }
}

// 修改刷新场景列表的方法
async function refreshScenes() {
  if (isRefreshingScenes.value) return
  
  try {
    isRefreshingScenes.value = true
    await sceneStore.fetchScenes()
  } catch (error) {
    ElMessage.error({
      message: '获取场景列表失败，请检查国际网络环境',
      offset: 100
    })
  } finally {
    // 3秒后重新启用按钮
    setTimeout(() => {
      isRefreshingScenes.value = false
    }, 500)
  }
}

// 修改 formatDate 函数，精确到分钟
function formatDate(dateString) {
  const date = new Date(dateString);
  return date.toLocaleString('zh-CN', {
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
  });
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
    <el-drawer 
      v-model="drawerVisible" 
      title="代码编辑器" 
      size="80%" 
      direction="ttb"
      class="code-editor-drawer"
    >
      <CodeEditor ref="codeEditor" @execute-code="onExecuteCodeFromEditor" />
    </el-drawer>

    <!-- 场景管理器抽屉 -->
    <el-drawer
      v-model="savedScenesDrawerVisible"
      title="已保存场景"
      :with-header="true"
      direction="rtl"
      size="400px"
      class="scene-manager-drawer"
    >
      <template #header>
        <div class="drawer-header">
          <span>已保存场景</span>
          <el-tooltip v-if="authStore.isLoggedIn" content="更新场景" placement="bottom" effect="dark">
            <el-button 
              class="refresh-btn" 
              :icon="RefreshLeft" 
              @click="refreshScenes"
              :disabled="isRefreshingScenes"
              :loading="isRefreshingScenes"
            />
          </el-tooltip>
        </div>
      </template>
      
      <!-- 未登录时显示提示信息 -->
      <div v-if="!authStore.isLoggedIn" class="login-prompt">
        <p>登录账号以启用此功能</p>
      </div>
      
      <!-- 已登录时显示场景列表 -->
      <div v-else class="scene-list">
        <!-- 场景列表为空时显示提示 -->
        <div v-if="!sceneStore.scenes.length" class="empty-scene-prompt">
          <p>场景空空如也</p>
        </div>
        
        <!-- 有场景时显示列表 -->
        <div 
          v-else
          v-for="scene in sceneStore.scenes" 
          :key="scene.id" 
          class="scene-item"
        >
          <div class="scene-content" @click="loadScene(scene)">
            <img :src="scene.texture" alt="场景预览" class="scene-preview" />
            <div class="scene-info">
              <div class="scene-name">{{ scene.name }}</div>
              <div class="scene-dimensions">尺寸：{{ scene.groundWidth }}m × {{ scene.groundDepth }}m</div>
              <div class="scene-created-at">{{ formatDate(scene.createdAt) }}</div>
            </div>
          </div>
          <el-button
            type="danger"
            size="small"
            class="delete-btn"
            @click.stop="handleDeleteScene(scene._id)"
          >
            <el-icon><Delete /></el-icon>
          </el-button>
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
  justify-content: space-between;
  gap: 12px;
  padding: 12px;
  border: 1px solid #e4e7ed;
  border-radius: 8px;
  cursor: pointer;
  position: relative; /* 添加相对定位 */
  overflow: hidden; /* 防止删除按钮溢出 */
}

.scene-content {
  display: flex;
  align-items: center;
  gap: 16px;
  flex: 1;
  padding-right: 40px; /* 为删除按钮预留空间 */
}

/* 修改删除按钮样式 */
.delete-btn {
  opacity: 0; /* 默认隐藏 */
  transition: all 0.3s ease;
  position: absolute;
  right: 12px;
  top: 50%;
  transform: translateY(-50%);
  border: none;
  background-color: rgba(245, 108, 108, 0.9);
  color: white;
  width: 32px;
  height: 32px;
  padding: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 4px;
}

.delete-btn:hover {
  background-color: rgb(245, 108, 108);
}

/* 当鼠标悬停在场景项上时显示删除按钮 */
.scene-item:hover .delete-btn {
  opacity: 1;
}

/* 修改场景内容样式，为删除按钮留出空间 */
.scene-content {
  display: flex;
  align-items: center;
  gap: 16px;
  flex: 1;
  padding-right: 40px; /* 为删除按钮预留空间 */
}

/* 优化场景项的悬停效果 */
.scene-item:hover {
  background-color: #f5f7fa;
  transform: translateY(-2px);
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
}

.scene-item:hover .scene-content {
  opacity: 0.8; /* 当鼠标悬停时，轻微降低内容透明度 */
}

.refresh-btn {
  padding: 12px;
  font-size: 16px;
  margin-right: 20px;
}

/* 确保抽屉标题样式正确 */
:deep(.el-drawer__header) {
  padding: 16px 0;
  margin-bottom: 0;
}

/* 修改：区分代码编辑器抽屉和场景管理器抽屉的样式 */
:deep(.code-editor-drawer.el-drawer) {
  background: #1e1e1e;
}

:deep(.code-editor-drawer .el-drawer__header) {
  margin-bottom: 0;
  padding: 16px 70px;
  background: #333333;
  color: white;
  font-weight: 600;
  text-align: center;
  height: 28px;
  position: relative;
}

:deep(.code-editor-drawer .el-drawer__close-btn) {
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

:deep(.code-editor-drawer .el-drawer__close-btn:hover) {
  background-color: rgba(255, 255, 255, 0.1);
}

/* 新增：场景管理器抽屉的样式 */
:deep(.scene-manager-drawer.el-drawer) {
  background: #ffffff;
}

:deep(.scene-manager-drawer .el-drawer__header) {
  margin-bottom: 0;
  padding: 16px 20px;
  background: #f5f7fa;
  color: #303133;
  font-weight: 600;
  border-bottom: 1px solid #e4e7ed;
}

.scene-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
  padding: 16px;
}

.scene-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 12px;
  border: 1px solid #e4e7ed;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.scene-content {
  display: flex;
  align-items: center;
  gap: 16px;
  flex: 1;
}

.scene-item:hover {
  background-color: #f5f7fa;
  transform: translateY(-2px);
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
}

.scene-preview {
  width: 80px;
  height: 80px;
  object-fit: cover;
  border-radius: 6px;
  border: 1px solid #e4e7ed;
}

.drawer-header {
  display: flex;
  align-items: center;
  justify-content: flex-start;  /* 让内容整体靠左 */
  gap: 20px;                   /* 添加按钮与标题之间的间距 */
  width: 100%;
  padding: 0 20px;
  margin-left: 5px;
}

.refresh-btn {
  margin-right: 40px;
}

/* 修改场景管理器抽屉的关闭按钮样式 */
:deep(.scene-manager-drawer .el-drawer__close-btn) {
  color: #303133;
  font-size: 20px;
  width: 32px;
  height: 32px;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
}

:deep(.scene-manager-drawer .el-drawer__close-btn:hover) {
  background-color: rgba(0, 0, 0, 0.1);
  color: #970a0a;
}

/* 调整场景信息样式 */
.scene-info {
  display: flex;
  flex-direction: column;
  gap: 8px; /* 增大行距 */
}

.scene-name {
  font-weight: bold; /* 加粗场景名称 */
}

.scene-dimensions{
    color: #909399;
}

.scene-created-at {
  color: #909399; /* 可以给时间设置一个不同的颜色 */
}

.login-prompt {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 200px;
  color: #909399;
  font-size: 16px;
}

.empty-scene-prompt {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 200px;
  color: #909399;
  font-size: 16px;
}
</style> 
