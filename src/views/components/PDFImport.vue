<template>
    <div class="main-container-top">
      <!-- 左边容器 -->
      <div :class="state.pdfPages ? 'left-container' : 'left-container-drag'">
        <!-- 原有的上传按钮和 PDF 容器 -->
        <div v-if="!state.pdfPages">
          <!-- 添加 el-upload 拖拽上传组件 -->
          <el-upload 
            class="upload-demo"  
            action="#" 
            :auto-upload="false" 
            :on-change="handleFileChange" 
            :limit="1"
            :on-exceed="handleExceed" 
            :on-remove="handleRemove" 
            drag
          >
            <i class="el-icon-upload"></i>
            <div class="el-upload__text">将文件拖到此处，或<em>点击上传</em></div>
          </el-upload>
        </div>
        <div v-else id="pdf-container" @mousemove="handleMouseMove" @mouseleave="handleMouseLeave">
          <canvas v-for="page in state.pdfPages" :key="page" :id="`pdfCanvas${page}`"
            style="border-bottom:1px solid #d4d2d2" />
        </div>
      </div>
      <!-- 右边容器 -->
      <div class="right-container">
        <!-- 右边上半部分 -->
        <div class="right-top" style="display: flex; align-items: center; justify-content: space-between; flex-direction: column;">
          <!-- 新增显示上传文件名字和当前配置 AI 模型名字的元素 -->
          <div class="info-container">
            <span class="info-item" title="{{ state.pdfSrc }}">文件名字: {{ state.pdfSrc }}</span>
            <span class="info-item" title="{{ selectedModel.value }}">当前配置 AI 模型: {{ selectedModel }}</span>
          </div>
          <!-- 使用 flex 布局让按钮自适应 -->
          <div class="button-container">
            <!-- 添加 Element Plus 上传按钮 -->
            <el-upload 
              action="#"  
              :show-file-list="false"
              :auto-upload="false" 
              :on-change="handleFileChange" 
              :limit="1" 
              :on-exceed="handleExceed" 
              :on-remove="handleRemove"
              type="success" 
              class="custom-upload"
            >
              <el-button type="primary" class="custom-upload-button">
                点击上传
              </el-button>
            </el-upload>
            <!-- 可以添加更多按钮 -->
            <!-- 添加配置 AI 模型的按钮 -->
            <el-button type="primary" @click="dialogVisible = true">配置 AI 模型</el-button>
            <!-- 修改按钮，添加 :loading 属性 -->
            <el-button type="primary" @click="submitPrompt" :loading="loading">开始生成</el-button>
          </div>
        </div>
        <!-- 右边下半部分 -->
        <div class="right-bottom">
          <!-- 这里可以添加右边下半部分的自定义内容 -->
          <PdfEditTable :pdfDealTableData="state.tableData" :submitLoading="loading"></PdfEditTable>
        </div>
      </div>
      <!-- 添加对话框 -->
      <el-dialog v-model="dialogVisible" title="配置 AI 模型">
        <el-form>
          <!-- 选择模型 -->
          <el-form-item label="选择模型">
            <el-select v-model="selectedModel" placeholder="请选择模型">
              <el-option v-for="(config, key) in apiConfigs" :key="config.key" :label="config.name"
                :value="config.value"></el-option>
            </el-select>
          </el-form-item>
          <!-- 二选一的单选框 -->
          <el-form-item label="参数设置">
            <el-radio-group v-model="paramType">
              <el-radio label="default">AI 默认参数</el-radio>
              <el-radio label="custom">自定义参数</el-radio>
            </el-radio-group>
          </el-form-item>
          <!-- 自定义参数文本框 -->
          <el-form-item v-if="paramType === 'custom'" label="自定义参数">
            <el-input type="textarea" v-model="customParams" placeholder="请输入自定义参数" :rows="35"></el-input>
          </el-form-item>
        </el-form>
        <template #footer>
          <span class="dialog-footer">
            <el-button @click="dialogVisible = false">取消</el-button>
            <el-button type="primary" @click="saveConfig">确定</el-button>
          </span>
        </template>
      </el-dialog>
    </div>
  </template>
  
  <script setup>
  import { onMounted, reactive, nextTick, ref } from "vue";
  import * as PDF from "pdfjs-dist/legacy/build/pdf.mjs";
  import aiAxios, { apiConfigs, ifpugFunctionPointEvaluationPrompt } from './pdfEditTable/config'; // 确保路径正确
  import PdfEditTable from './pdfEditTable';
  import { data2 } from './pdfEditTable/mock'; // 确保路径正确
  const data3 = [
    {
      id: 1,
      "subsystem": "智慧养殖大数据管理平台",
      "level1": "基础信息管理",
      "countItem": "养殖场信息管理",
      "description": "对养殖场内鸡场进行动态维护，包括名称、地址、联系人、联系电话等信息的增删改查。",
      "ufp": 5,
      "pdfTransformText": "黄花"
    },
    {
      id: 2,
      "subsystem": "智慧养殖大数据管理平台",
      "level1": "基础信息管理",
      "countItem": "供应商信息管理",
      "description": "对鸡场所对应的供应商进行动态维护，包括商家信息、对应供应类型、联系人、联系电话、供应商所在地址等信息的增删改查。",
      "ufp": 5,
      "pdfTransformText": "对鸡场内养殖场信息、供应商信息、员工信息、鸡舍信息进行动态维护。"
    },
    {
      id: 3,
      "subsystem": "智慧养殖大数据管理平台",
      "level1": "基础信息管理",
      "countItem": "员工信息管理",
      "description": "对鸡场内的工作人员进行动态维护，包括员工编号、姓名、性别、联系方式、负责鸡舍等信息的增删改查。",
      "ufp": 5,
      "pdfTransformText": "对鸡场内养殖场信息、供应商信息、员工信息、鸡舍信息进行动态维护。"
    },
    {
      id: 4,
      "subsystem": "智慧养殖大数据管理平台",
      "level1": "基础信息管理",
      "countItem": "鸡舍信息管理",
      "description": "对鸡场内鸡舍进行动态维护，包括鸡舍号、类别、最大存栏量、所在位置等信息的增删改查。",
      "ufp": 5,
      "pdfTransformText": "对鸡场内养殖场信息、供应商信息、员工信息、鸡舍信息进行动态维护。"
    }]
  PDF.GlobalWorkerOptions.workerSrc = 'node_modules/pdfjs-dist/legacy/build/pdf.worker.mjs';
  
  const state = reactive({
    pdfPath: '',
    pdfPages: '',
    pdfWidth: '',
    pdfSrc: '',
    pdfScale: 1.0,
    pdfValue: '',
    tableData: [], // 用于存储表格数据的数组
    selectedRow: null, // 用于存储选中的行数据
    selectedCell: null, // 用于存储选中的单元格数据 
  });
  
  let pdfDoc = null;
  const redCharPositions = ref([]);
  const highlightIdPositions = ref([]);
  
  // 新增 loading 状态
  const loading = ref(false);
  
  // Handle file upload change event
  const handleFileChange = (file) => {
    let actualFile;
    if (file && file.raw instanceof File) {
      actualFile = file.raw;
    } else if (file instanceof File) {
      actualFile = file;
    } else {
      return;
    }
    handleRemove(); 
    console.log('File has changed:', actualFile);
    state.pdfSrc = actualFile.name;
    const fileReader = new FileReader();
    fileReader.onload = (e) => {
      state.pdfPath = e.target.result;
      loadFile(state.pdfPath);
    };
    fileReader.readAsDataURL(actualFile);
  };
  
  // 处理超出限制事件
  const handleExceed = (files, fileList) => {
    console.log('每次只能上传一个文件，新文件将覆盖现有文件。');
    // 使用 splice 方法清空 fileList
    if (fileList.length > 0) {
      fileList.splice(0, fileList.length);
    }
    // 检查是否有新文件
    if (files.length > 0) {
      handleFileChange(files[0]);
    }
  };
  
  // 处理文件移除事件
  const handleRemove = () => {
    // 清空 PDF 相关状态
    state.pdfPath = '';
    state.pdfPages = '';
    state.pdfWidth = '';
    state.pdfSrc = '';
    state.pdfValue = '';
    pdfDoc = null;
  };
  
  onMounted(() => {
    // 初始不加载文件，等待用户上传
  });
  
  function loadFile(url) {
    PDF.getDocument({
      url,
      cMapUrl: 'node_modules/pdfjs-dist/cmaps/',
      cMapPacked: true
    }).promise.then((p) => {
      pdfDoc = p;
      const { numPages } = p;
      state.pdfPages = numPages;
      nextTick(() => {
        renderPage(1);
      });
    }).catch((error) => {
      console.error('加载 PDF 文件出错:', error);
    });
  }
  
  async function renderPage(num) {
    const page = await pdfDoc.getPage(num);
    const canvas = document.getElementById(`pdfCanvas${num}`);
    if (!canvas) {
      console.error(`Canvas element with ID pdfCanvas${num} not found.`);
      return;
    }
    const ctx = canvas.getContext('2d');
    const dpr = window.devicePixelRatio || 1;
    const bsr = ctx.webkitBackingStorePixelRatio
      || ctx.mozBackingStorePixelRatio
      || ctx.msBackingStorePixelRatio
      || ctx.oBackingStorePixelRatio
      || ctx.backingStorePixelRatio
      || 1;
    const ratio = dpr / bsr;
    const viewport = page.getViewport({ scale: state.pdfScale });
  
    canvas.width = viewport.width * ratio;
    canvas.height = viewport.height * ratio;
    canvas.style.width = '100%';
    canvas.style.height = 'auto';
    state.pdfWidth = `${viewport.width}px`;
    ctx.setTransform(ratio, 0, 0, ratio, 0, 0);
  
    ctx.clearRect(0, 0, canvas.width, canvas.height);
  
    const textContent = await page.getTextContent();
    const textItems = textContent.items;
  
    redCharPositions.value = [];
    highlightIdPositions.value = [];
    let fullText = '';
  
    for (const item of textItems) {
      const { str } = item;
      fullText += str;
    }
  
    // 遍历 data3 中的 pdfTransformText 进行匹配
    for (const item of data3) {
      const searchText = item.pdfTransformText;
      let startIndex = 0;
      while (true) {
        const index = fullText.toLowerCase().indexOf(searchText.toLowerCase(), startIndex);
        if (index === -1) break;
  
        // 找到匹配项，记录位置和对应的 id
        redCharPositions.value.push({
          start: index,
          end: index + searchText.length
        });
        highlightIdPositions.value.push({
          start: index,
          end: index + searchText.length,
          id: item.id
        });
  
        startIndex = index + searchText.length;
      }
    }
  
  
    let currentTextIndex = 0;
    for (const item of textItems) {
      const { str, transform } = item;
      const fontSize = transform[3];
      const x = transform[4];
      const y = transform[5];
      const newY = viewport.height - y;
  
      for (let i = 0; i < str.length; i++) {
        const char = str[i];
        state.pdfValue = state.pdfValue + char;
  
        // 检查当前字符是否在高亮范围内
        let isHighlight = false;
        for (const highlight of redCharPositions.value) {
          if (currentTextIndex >= highlight.start && currentTextIndex < highlight.end) {
            isHighlight = true;
            break;
          }
        }
  
        ctx.fillStyle = isHighlight ? 'red' : 'black';
        ctx.font = `${fontSize}px sans-serif`;
        ctx.fillText(char, x + ctx.measureText(str.slice(0, i)).width, newY);
  
        currentTextIndex++;
      }
    }
  
    if (state.pdfPages > num) {
      renderPage(num + 1);
    }
  }
  
  // 处理鼠标移动事件
  async function handleMouseMove(event) {
    const canvas = event.target.closest('canvas');
    if (!canvas) return;
    
    const rect = canvas.getBoundingClientRect();
    const x = event.clientX - rect.left;
    const y = event.clientY - rect.top;
  
    const pageNum = parseInt(canvas.id.replace('pdfCanvas', ''));
    const page = await pdfDoc.getPage(pageNum);
    const textContent = await page.getTextContent();
    const textItems = textContent.items;
    const viewport = page.getViewport({ scale: state.pdfScale });
    const ctx = canvas.getContext('2d');
  
    let currentTextIndex = 0;
    let foundHighlight = null;
  
    // 遍历所有文本项
    for (const item of textItems) {
      const { transform, str } = item;
      const fontSize = transform[3];
      const itemX = transform[4];
      const itemY = viewport.height - transform[5];
  
      // 检查鼠标是否在当前文本项范围内
      if (
        x >= itemX &&
        x <= itemX + ctx.measureText(str).width &&
        y >= itemY - fontSize &&
        y <= itemY
      ) {
        // 计算当前字符的索引
        const charIndex = Math.floor((x - itemX) * str.length / ctx.measureText(str).width);
        currentTextIndex += charIndex;
        
        // 检查当前字符是否在高亮区域内
        for (const highlight of highlightIdPositions.value) {
          if (currentTextIndex >= highlight.start && currentTextIndex < highlight.end) {
            foundHighlight = highlight;
            break;
          }
        }
        break;
      } else {
        currentTextIndex += str.length;
      }
    }
    
    if (foundHighlight) {
      console.log('当前高亮文字对应的 id:', foundHighlight.id);
      canvas.style.cursor = 'pointer';
    } else {
      canvas.style.cursor = 'default';
    }
  }
  
  // 处理鼠标离开事件
  function handleMouseLeave() {
    // idList.value = [];
  }
  
  // 新增对话框相关响应式数据和方法
  const dialogVisible = ref(false);
  const selectedModel = ref('');
  const paramType = ref('default');
  const customParams = ref('');
  
  const saveConfig = () => {
    console.log('保存配置:', {
      selectedModel: selectedModel.value,
      paramType: paramType.value,
      customParams: customParams.value
    });
    dialogVisible.value = false;
  };
  
  const submitPrompt = async () => {
    // 开始加载，设置 loading 为 true
    loading.value = true;
    try {
      // 使用 Promise 封装 setTimeout
      await new Promise((resolve) => {
        setTimeout(() => {
          state.tableData = data2;
          resolve();
        }, 3000);
      });
    } catch (error) {
      console.error('生成过程出错:', error);
    } finally {
      // 加载结束，设置 loading 为 false
      loading.value = false;
    }
  };
  </script>
  
  <style scoped>
  .main-container-top {
    display: flex;
    height: 100vh;
  }
  
  .left-container {
    height: 100vh;
    overflow-y: auto;
    padding: 10px;
    width: 500px;
    margin: 20px;
    background: #eaeaea;
    flex-shrink: 0;
    /* 确保 left-container 不会被压缩 */
  }
  
  .left-container-drag {
    height: 100vh;
    overflow-y: auto;
    padding: 10px;
    width: 500px;
    margin: 20px;
    flex-shrink: 0;
    /* 确保 left-container-drag 不会被压缩 */
  }
  
  .right-container {
    flex:1;
    display: flex;
    flex-direction: column;
    overflow-x: auto;
    /* 支持横向滚动 */
    min-width: 0;
    /* 确保内容溢出时可以滚动 */
  }
  
  .right-top {
    height: 120px;
    padding: 10px;
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
  }
  
  .right-bottom {
    flex: 1;
    border: 1px solid #ccc;
    padding: 10px;
  }
  
  /* 新增按钮容器样式 */
  .button-container {
    display: flex;
    flex-wrap: wrap;
    /* 允许按钮换行 */
    gap: 10px;
    /* 设置按钮之间的间距 */
    align-items: center;
    /* 垂直居中对齐 */
  }
  
  .right-bottom {
    flex: 1;
    border: 1px solid #ccc;
    padding: 10px;
  }
  
  
  #pdf-container {
    width: 100%;
    background: #fff;
  }
  
  .upload-demo {
    width: 100%;
    height: 600px;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    border: 1px dashed #d9d9d9;
    border-radius: 4px;
    margin-top: 10px;
  }
  
  .upload-demo .el-upload-dragger {
    border: none;
  }
  
  /* 自定义 el-upload 样式 */
  .custom-upload {
    display: flex;
    align-items: flex-start;
    flex-direction: column;
  }
  
  /* 自定义 el-button 样式 */
  .custom-upload-button {
    line-height: normal;
    /* 重置行高 */
  }
  
  .info-container {
    display: flex;
    flex-direction: column; /* 垂直布局 */
    align-items: flex-start; /* 靠左对齐 */
    width: 100%;
    margin-bottom: 10px;
    padding: 8px;
    background-color: #f5f7fa;
    border-radius: 4px;
  }
  
  .info-item {
    font-size: 14px;
    color: #606266;
    white-space: nowrap; /* 禁止换行 */
    overflow: hidden; /* 隐藏溢出内容 */
    text-overflow: ellipsis; /* 溢出内容用省略号表示 */
    width: 100%; /* 占满父容器宽度 */
  }
  </style>
  