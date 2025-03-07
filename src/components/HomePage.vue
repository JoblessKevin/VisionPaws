<template>
  <div class="homepage">
    <!-- 頁面標題區域 -->
    <div class="header">
      <h1>VisionPaws.ai</h1>
      <p>Upload an image to analyze animals</p>
    </div>

    <!-- 主要內容區域 -->
    <div class="content-container">
      <!-- 上傳區域 -->
      <FileUpload
        :multiple="false"
        accept="image/*"
        :maxFileSize="5000000"
        @select="onSelect"
        :auto="true"
        chooseLabel="選擇圖片"
        :showCancelButton="false"
        :showUploadButton="false"
        class="upload-section"
      >
        <template #empty>
          <div class="upload-placeholder">
            <i class="pi pi-image" style="font-size: 2rem"></i>
            <span>拖放圖片至此或點擊上傳</span>
            <span class="upload-hint">支援的格式: JPG, PNG (最大 5MB)</span>
          </div>
        </template>
      </FileUpload>

      <!-- 預覽和結果區域 -->
      <div v-if="selectedImage || analysisResult" class="result-container">
        <!-- 圖片預覽 -->
        <div v-if="selectedImage" class="preview-section">
          <h2>預覽圖片</h2>
          <img :src="selectedImage" alt="Preview" class="preview-image" />
        </div>

        <!-- 分析結果 -->
        <div v-if="analysisResult" class="analysis-section">
          <h2>分析結果</h2>
          <Card>
            <template #content>
              <div v-html="analysisResult"></div>
            </template>
          </Card>
        </div>
      </div>

      <!-- 載入中狀態 -->
      <ProgressSpinner v-if="loading" class="spinner" />
    </div>

    <!-- 錯誤訊息 -->
    <Toast position="bottom-right" />
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import { useToast } from "primevue/usetoast";

// 狀態管理
const selectedImage = ref(null);
const analysisResult = ref(null);
const loading = ref(false);
const toast = useToast();

// 當選擇檔案時的處理
const onSelect = async (event) => {
  const file = event.files[0];
  if (file) {
    const reader = new FileReader();
    reader.onload = (e) => {
      selectedImage.value = e.target.result;
    };
    reader.readAsDataURL(file);

    try {
      await processImage(file);
    } catch (error) {
      console.error("Error processing image:", error);
      toast.add({
        severity: "error",
        summary: "錯誤",
        detail: "圖片處理失敗，請稍後再試",
        life: 3000,
      });
    }
  }
};

// 處理圖片上傳和分析的完整流程
const processImage = async (file) => {
  try {
    loading.value = true;

    // 步驟1: 上傳圖片並取得 file id
    const uploadResult = await uploadImage(file);
    console.log("Upload result:", uploadResult); // 除錯用

    if (!uploadResult || !uploadResult.id) {
      throw new Error("Failed to get upload file id");
    }

    // 步驟2: 使用取得的 file id 執行影像分析
    await analyzeImage(uploadResult.id);
  } catch (error) {
    console.error("Process error:", error);
    throw error;
  } finally {
    loading.value = false;
  }
};

// 上傳圖片到 Dify
const uploadImage = async (file) => {
  const formData = new FormData();
  formData.append("file", file);

  try {
    console.log("Uploading to:", import.meta.env.VITE_DIFY_UPLOAD_ENDPOINT);
    const response = await fetch(import.meta.env.VITE_DIFY_UPLOAD_ENDPOINT, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${import.meta.env.VITE_DIFY_API_KEY}`,
      },
      body: formData,
    });

    if (!response.ok) {
      const errorData = await response.json().catch(() => ({}));
      console.error("Upload response:", response.status, errorData);
      throw new Error(`Upload failed: ${response.status}`);
    }

    const data = await response.json();
    console.log("Upload response data:", data); // 除錯用
    return data; // 返回完整的回應數據，包含 id
  } catch (error) {
    console.error("Error uploading image:", error);
    toast.add({
      severity: "error",
      summary: "錯誤",
      detail: "圖片上傳失敗：" + error.message,
      life: 3000,
    });
    throw error;
  }
};

// 使用 Dify Workflow 分析圖片
const analyzeImage = async (fileId) => {
  console.log("Analyzing with:", import.meta.env.VITE_DIFY_WORKFLOW_ENDPOINT);
  try {
    // 準備 payload，使用上傳後獲得的 file id
    const payload = {
      inputs: {
        pic: {
          transfer_method: "local_file",
          upload_file_id: fileId, // 使用從第一個 API 獲得的 id
          type: "image",
        },
      },
      response_mode: "blocking",
      user: "Ambre",
    };

    console.log("Analysis payload:", payload); // 除錯用

    const response = await fetch(import.meta.env.VITE_DIFY_WORKFLOW_ENDPOINT, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${import.meta.env.VITE_DIFY_API_KEY}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(payload),
    });

    if (!response.ok) {
      const errorData = await response.json().catch(() => ({}));
      console.error("Analysis response:", response.status, errorData);
      throw new Error(`Analysis failed: ${response.status}`);
    }

    const data = await response.json();
    console.log("Analysis response data:", data); // 除錯用

    // 根據實際 API 回傳格式處理結果
    analysisResult.value = data.answer || data.response || JSON.stringify(data);

    toast.add({
      severity: "success",
      summary: "分析完成",
      detail: "圖片分析已完成",
      life: 3000,
    });
  } catch (error) {
    console.error("Error analyzing image:", error);
    toast.add({
      severity: "error",
      summary: "錯誤",
      detail: "圖片分析失敗：" + error.message,
      life: 3000,
    });
    throw error;
  }
};

onMounted(() => {
  console.log("Environment variables:", {
    apiKey: import.meta.env.VITE_DIFY_API_KEY,
    uploadEndpoint: import.meta.env.VITE_DIFY_UPLOAD_ENDPOINT,
    workflowEndpoint: import.meta.env.VITE_DIFY_WORKFLOW_ENDPOINT,
  });
});
</script>

<style scoped>
/* 保持原有的 CSS 樣式不變 */
.homepage {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem;
}

.header {
  text-align: center;
  margin-bottom: 2rem;
}

.header h1 {
  color: var(--primary-color);
  margin-bottom: 0.5rem;
}

.content-container {
  display: flex;
  flex-direction: column;
  gap: 2rem;
}

.upload-section {
  width: 100%;
  min-height: 200px;
}

.upload-placeholder {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
  padding: 2rem;
}

.upload-hint {
  font-size: 0.875rem;
  color: var(--text-color-secondary);
}

.result-container {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 2rem;
}

.preview-section,
.analysis-section {
  padding: 1rem;
}

.preview-image {
  max-width: 100%;
  height: auto;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.spinner {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

@media (max-width: 768px) {
  .homepage {
    padding: 1rem;
  }

  .result-container {
    grid-template-columns: 1fr;
  }
}
</style>
