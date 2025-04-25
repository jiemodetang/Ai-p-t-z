<template>
    <el-tabs v-model="activeName" class="demo-tabs" @tab-click="handleClick">
      <el-tab-pane label="造价pdf处理" name="first">
        <div class="button-group">
          <!-- 添加自定义类名 narrow-input  -->
          <el-input v-model="currentHighlightKeyword" placeholder="输入要高亮的文本" @input="highlightSearchText"
            class="narrow-input"></el-input>
          <!-- 替换配置选择按钮为下拉框，添加自定义类名 -->
          <el-select v-model="selectedConfig" placeholder="请选择配置" class="narrow-select">
            <el-option label="doubao" value="doubao"></el-option>
            <el-option label="deepSeekR1" value="deepSeekR1"></el-option>
          </el-select>
          <!-- 新增关键词下拉框 -->
          <el-select v-model="selectedKeyword" placeholder="选择关键词高亮" class="narrow-select"
            @change="highlightSelectedKeyword">
            <el-option v-for="(keyword, index) in pdfTransformTextList" :key="index" :label="keyword"
              :value="keyword"></el-option>
          </el-select>
          <!-- 新增标记全部文档内容按钮 -->
          <el-button type="primary" @click="markAllContents">标记全部匹配内容</el-button>
          <!-- 新增上传按钮 -->
          <el-button type="primary" @click="handleFileUpload">上传 PDF</el-button>
  
          <!-- 添加 loading 属性 -->
          <el-button type="primary" @click="submitPrompt" :loading="submitLoading">
            开始生成
          </el-button>
          <!-- 新增输入框 -->
        </div>
  
        <div class="container">
          <!-- 左侧PDF上传和预览区域 -->
          <div class="left-panel">
            <!-- 条件渲染拖拽上传区域 -->
            <div v-if="!pdfStr.length">
              <el-upload class="upload-demo" drag action="" :auto-upload="false" :on-change="handleFileChange"
                :show-file-list="false" accept=".pdf">
                <el-icon class="el-icon--upload">
                  <UploadFilled />
                </el-icon>
                <div class="el-upload__text">
                  拖拽PDF文件到此处或<em>点击上传</em>
                </div>
                <template #tip>
                  <div class="el-upload__tip">
                    请上传PDF文件，系统将自动提取文本内容
                  </div>
                </template>
              </el-upload>
            </div>
  
            <div v-if="pdfText.length > 0" class="pdf-preview">
              <div v-for="(page, index) in pdfText" :key="index" class="pdf-page">
                <div v-for="(item, i) in page.items" :key="i" class="text-item" :style="item.style">
                  <!-- 高亮显示匹配文本 -->
                  <span v-html="highlightText(item.str, i, page.items)"></span>
                </div>
              </div>
            </div>
          </div>
  
          <!-- 右侧表格区域 -->
          <div class="right-panel">
            <PdfEditTable :pdfDealTableData="pdfDealTableData" :submitLoading="submitLoading"></PdfEditTable>
          </div>
        </div>
  
        <!-- 移除新增配置选择弹窗 -->
      </el-tab-pane>
      <el-tab-pane label="next" name="s">
        test2
      </el-tab-pane>
    </el-tabs>
    <!-- 全屏加载遮罩 -->
    <div v-if="loading" class="fullscreen-loading">
      <div class="loading-content">
        <el-progress type="circle" :percentage="progress" />
        <p>正在解析PDF文件...</p>
      </div>
    </div>
  </template>
  
  <script setup>
  import { ref, computed } from 'vue';
  import { UploadFilled } from '@element-plus/icons-vue';
  import * as pdfjsLib from 'pdfjs-dist';
  import PdfEditTable from './pdfEditTable';
  import aiAxios from './pdfEditTable/config';
  import { ElMessage, ElLoading } from 'element-plus';
  import { data2 } from './pdfEditTable/mock';
  import { ifpugFunctionPointEvaluationPrompt } from './pdfEditTable/ai';
  // 配置pdfjs worker路径
  pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdn.jsdelivr.net/npm/pdfjs-dist@5.1.91/+esm';
  
  // PDF文本数据
  const pdfText = ref([]);
  const loading = ref(false);
  const progress = ref(0);
  const activeName = ref('first');
  //pdf 文本内容
  const pdfStr = ref('');
  const pdfDealTableData = ref([]);
  
  // 新增弹窗相关数据，移除 dialogVisible
  const selectedConfig = ref('doubao');
  
  // test 
  // 新增 submitLoading 响应式变量
  const submitLoading = ref(false);
  
  // 搜索文本，统一使用 currentHighlightKeyword
  const currentHighlightKeyword = ref('');
  
  
  const submitPrompt = async () => {
    try {
      // 开始加载，显示加载状态
      submitLoading.value = true;
      const text = await aiAxios(selectedConfig.value, pdfStr.value + '，' + ifpugFunctionPointEvaluationPrompt);
  
      // 调用辅助函数提取 JSON 数据
      const jsonData = extractJsonFromText(text);
      if (jsonData) {
        console.log('解析后的 JSON 数据:', jsonData);
        pdfDealTableData.value = jsonData;
      } else {
        console.error('未找到有效的 JSON 数据');
        ElMessage.error('未找到有效的 JSON 数据');
      }
    } catch (error) {
      console.error('请求 AI 接口出错:', error);
      ElMessage.error('请求 AI 接口时出错');
    } finally {
      // 加载结束，隐藏加载状态
      submitLoading.value = false;
    }
  };
  
  // 提取 JSON 数据的辅助函数
  const extractJsonFromText = (text) => {
    return JSON.parse(text);
  };
  
  // 处理文件上传
  const handleFileChange = async (file) => {
    if (file.raw.type !== 'application/pdf') {
      ElMessage.error('请上传PDF文件');
      return;
    }
  
    try {
      loading.value = true;
      progress.value = 0;
  
      const arrayBuffer = await file.raw.arrayBuffer();
      await extractPdfText(arrayBuffer);
    } catch (error) {
      console.error('PDF处理错误:', error);
      ElMessage.error('处理PDF文件时出错');
    } finally {
      loading.value = false;
    }
  };
  
  // 新增手动触发文件上传方法
  const handleFileUpload = () => {
    const input = document.createElement('input');
    input.type = 'file';
    input.accept = '.pdf';
    input.addEventListener('change', (e) => {
      const file = e.target.files[0];
      if (file) {
        handleFileChange({ raw: file });
      }
    });
    input.click();
  };
  
  const handleClick = (tab, event) => {
    console.log(tab, event);
  };
  
  // 提取PDF文本内容
  const extractPdfText = async (arrayBuffer) => {
    pdfText.value = [];
  
    // 加载PDF文档
    const pdf = await pdfjsLib.getDocument({
      data: arrayBuffer,
      // 启用PDF文本内容提取
      cMapUrl: 'https://cdn.jsdelivr.net/npm/pdfjs-dist@3.4.120/cmaps/',
      cMapPacked: true,
    }).promise;
  
    let str = '';
  
    // 逐页提取文本
    for (let i = 1; i <= pdf.numPages; i++) {
      const page = await pdf.getPage(i);
      const textContent = await page.getTextContent({
        normalizeWhitespace: true,
        disableCombineTextItems: false
      });
  
      // 更新进度
      progress.value = Math.round((i / pdf.numPages) * 100);
  
      // 处理文本项
      const items = textContent.items.map(item => {
        // 计算字体大小
        const fontSize = Math.round(item.transform[0]);
  
        const scaledViewport = page.getViewport({ scale: 1.0 });
        const top = scaledViewport.height - item.transform[5];
        str += item.str;
        return {
          str: item.str,
          style: {
            fontSize: `${fontSize}px`,
            left: `${item.transform[4]}px`,
            top: `${top}px`,
            fontFamily: item.fontName,
            width: item.width,
          }
        };
      });
      console.log(pdfText);
  
      pdfText.value.push({
        pageNum: i,
        items: items
      });
    }
    pdfStr.value = str;
  };
  
  
  
  
  // 计算属性，抽离 data2 中的 pdfTransformText 字段
  const pdfTransformTextList = computed(() => {
    return data2.map(item => item.pdfTransformText).filter(text => text);
  });
  
  // 存储当前选中的关键词
  const selectedKeyword = ref('');
  // 存储当前要高亮的单个关键词
  
  // 高亮选中关键词的函数
  const highlightSelectedKeyword = () => {
    if (selectedKeyword.value) {
      currentHighlightKeyword.value = selectedKeyword.value;
      highlightSearchText();
      ElMessage.success(`已标记 "${selectedKeyword.value}" 内容`);
    }
  };
  
  // 高亮显示匹配文本的函数，统一使用 currentHighlightKeyword
  const highlightText = (text, currentIndex, allItems) => {
    if (!currentHighlightKeyword.value) return text;
  
    // 拼接整页文本
    const pageText = allItems.map(item => item.str).join('');
    const keyword = currentHighlightKeyword.value;
  
    // 查找高亮关键词在整页文本中的位置
    let keywordIndex = pageText.indexOf(keyword);
    if (keywordIndex === -1) return text;
  
    let startIndex = 0;
    const highlightedItems = [];
  
    // 找到所有匹配项
    while (keywordIndex !== -1) {
      const keywordEndIndex = keywordIndex + keyword.length;
      let currentStart = 0;
      for (let i = 0; i < allItems.length; i++) {
        const item = allItems[i];
        const currentEnd = currentStart + item.str.length;
  
        if (currentStart < keywordEndIndex && currentEnd > keywordIndex) {
          const innerStart = Math.max(0, keywordIndex - currentStart);
          const innerEnd = Math.min(item.str.length, keywordEndIndex - currentStart);
          const highlightedItem =
            item.str.slice(0, innerStart) +
            '<span class="highlight">' +
            item.str.slice(innerStart, innerEnd) +
            '</span>' +
            item.str.slice(innerEnd);
          highlightedItems.push({ index: i, text: highlightedItem });
        }
        currentStart = currentEnd;
      }
      keywordIndex = pageText.indexOf(keyword, keywordIndex + 1);
    }
  
    // 如果当前项是部分匹配，查找上下 5 个节点
    if (text.includes(keyword)) {
      const start = Math.max(0, currentIndex - 2);
      const end = Math.min(allItems.length, currentIndex + 3);
      for (let i = start; i < end; i++) {
        if (allItems[i].str.includes(keyword)) {
          const index = highlightedItems.findIndex(item => item.index === i);
          if (index === -1) {
            const item = allItems[i];
            const innerStart = item.str.indexOf(keyword);
            const innerEnd = innerStart + keyword.length;
            const highlightedItem =
              item.str.slice(0, innerStart) +
              '<span class="highlight">' +
              item.str.slice(innerStart, innerEnd) +
              '</span>' +
              item.str.slice(innerEnd);
            highlightedItems.push({ index: i, text: highlightedItem });
          }
        }
      }
    }
  
    const highlightedItem = highlightedItems.find(item => item.index === currentIndex);
    return highlightedItem ? highlightedItem.text : text;
  };
  
  // 触发高亮搜索
  const highlightSearchText = () => {
    // 强制重新渲染
    pdfText.value = [...pdfText.value];
  };
  
  // 标记文档所有匹配内容的函数
  const markAllContents = () => {
    // 拼接所有 pdfTransformText 为一个正则表达式模式
    const allKeywords = pdfTransformTextList.value.join('|');
    if (allKeywords) {
      currentHighlightKeyword.value = allKeywords;
      highlightSearchText();
      ElMessage.success('已标记文档中所有匹配内容');
    }
  };
  </script>
  
  <style scoped>
  .container {
    display: flex;
    height: 100vh;
    width: 100%;
  }
  
  .right-panel {
    flex: 1;
    padding: 20px;
    overflow-y: auto;
    height: 100%;
    box-sizing: border-box;
  }
  
  .left-panel {
    border-right: 1px solid #ebeef5;
    padding: 20px;
    overflow-y: auto;
    height: 100%;
    box-sizing: border-box;
    width: 800px;
  }
  
  .upload-demo {
    margin-bottom: 20px;
  }
  
  .pdf-preview {
    margin-top: 20px;
    position: relative;
    background: #f9f9f9;
  }
  
  .pdf-page {
    margin-bottom: 30px;
    padding: 15px;
    border: 1px solid #ebeef5;
    border-radius: 4px;
    background-color: #fff;
    position: relative;
    min-height: 842px;
    width: 80%;
    margin: 0 auto;
    margin-bottom: 20px;
  }
  
  .text-item {
    position: absolute;
    white-space: pre;
    line-height: 1.2;
    margin: 2px 0;
    transition: all 0.2s;
  }
  
  .table-header {
    margin-bottom: 15px;
    display: flex;
    gap: 10px;
  }
  
  .demo-tabs>.el-tabs__content {
    padding: 32px;
    color: #6b778c;
    font-size: 32px;
    font-weight: 600;
  }
  
  .demo-tabs {
    padding: 20px;
  }
  
  .button-group {
    margin-bottom: 20px;
    display: flex;
    gap: 10px;
  }
  
  /* 自定义下拉框宽度 */
  .narrow-select {
    width: 150px;
    /* 可根据需要调整宽度 */
  }
  
  /* 全局加载样式 */
  .global-loading-container {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    z-index: 9999;
  }
  
  /* 遮罩层样式 */
  .loading-mask {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: rgba(0, 0, 0, 0.3);
  }
  
  /* 加载内容样式 */
  .global-loading-container>*:not(.loading-mask) {
    position: relative;
    background-color: rgba(255, 255, 255, 0.9);
    padding: 20px;
    border-radius: 8px;
    box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
    z-index: 1;
  }
  
  /* 全屏加载遮罩样式 */
  .fullscreen-loading {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.7);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 9999;
  }
  
  /* 加载内容样式 */
  .loading-content {
    background-color: transparent;
    padding: 20px;
    border-radius: 8px;
    /* box-shadow: 0 0 10px rgba(0, 0, 0, 0.1); */
    text-align: center;
    color: #fff;
  }
  
  /* 将 el-progress__text 文字设置为白色 */
  :deep(.el-progress__text) {
    color: #fff;
  }
  
  
  /* 使用 :deep() 穿透作用域，并提高样式优先级 */
  :deep(.pdf-preview .highlight) {
    background-color: yellow !important;
  }
  
  /* 自定义输入框宽度 */
  .narrow-input {
    width: 200px;
    /* 可根据需要调整宽度 */
  }
  </style>
  