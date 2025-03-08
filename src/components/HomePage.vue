<template>
  <div class="page-container">
    <!-- Left Side - Fixed Content -->
    <div class="left-panel">
      <!-- Title -->
      <div class="header">
        <h1>VisionPaws.ai</h1>
        <p>Upload an image to analyze animals</p>
      </div>

      <!-- Upload Section -->
      <FileUpload
        :multiple="false"
        accept="image/*"
        :maxFileSize="5000000"
        @select="onSelect"
        :auto="true"
        chooseLabel="choose file"
        :showCancelButton="false"
        :showUploadButton="false"
        class="upload-section"
      >
        <template #empty>
          <div class="upload-placeholder">
            <i class="pi pi-image" style="font-size: 2rem"></i>
            <span>Drag and drop the image here or click to upload</span>
            <span class="upload-hint"
              >Supported formats: JPG, PNG (max 5MB)</span
            >
          </div>
        </template>
      </FileUpload>
    </div>

    <!-- Right Side - Scrollable Content -->
    <div class="right-panel">
      <!-- Preview and results area -->
      <div v-if="selectedImage || analysisResult" class="result-container">
        <!-- Image Preview -->
        <div v-if="selectedImage" class="preview-section">
          <h2>Preview Image</h2>
          <img :src="selectedImage" alt="Preview" class="preview-image" />
        </div>

        <!-- Analysis Result -->
        <div v-if="analysisResult" class="analysis-section">
          <h2>Analysis Result</h2>
          <Card class="result-card">
            <template #content>
              <div class="result-content">
                <div
                  v-for="(section, index) in analysisResult"
                  :key="index"
                  class="result-section"
                  v-html="formatMarkdown(section)"
                ></div>
              </div>
            </template>
          </Card>
        </div>
      </div>
    </div>

    <!-- Loading Spinner -->
    <ProgressSpinner v-if="loading" class="spinner" />

    <!-- Toast Messages -->
    <Toast position="bottom-right" />
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import { useToast } from "primevue/usetoast";

//  Vue-state-management
const selectedImage = ref(null);
const analysisResult = ref(null);
const loading = ref(false);
const toast = useToast();

// What to do when selecting a file
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
        summary: "error",
        detail: "Image processing failed, please try again later",
        life: 3000,
      });
    }
  }
};

// Complete process for handling image upload and analysis
const processImage = async (file) => {
  try {
    loading.value = true;

    // Step 1: Upload the image and get the file id
    const uploadResult = await uploadImage(file);
    console.log("Upload result:", uploadResult); // For debugging

    if (!uploadResult || !uploadResult.id) {
      throw new Error("Failed to get upload file id");
    }

    // Step 2: Use the obtained file id to perform image analysis
    await analyzeImage(uploadResult.id);
  } catch (error) {
    console.error("Process error:", error);
    throw error;
  } finally {
    loading.value = false;
  }
};

// Upload image to Dify
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
    console.log("Upload response data:", data); // for debugging
    return data; // Return complete response data, including id
  } catch (error) {
    console.error("Error uploading image:", error);
    toast.add({
      severity: "error",
      summary: "error",
      detail: "Image upload failed: " + error.message,
      life: 3000,
    });
    throw error;
  }
};

// Analyze images using Dify Workflow
const analyzeImage = async (fileId) => {
  console.log("Analyzing with:", import.meta.env.VITE_DIFY_WORKFLOW_ENDPOINT);
  try {
    // Prepare the payload, using the file id obtained after uploading
    const payload = {
      inputs: {
        pic: {
          transfer_method: "local_file",
          upload_file_id: fileId, // Use the id obtained from the first API
          type: "image",
        },
      },
      response_mode: "blocking",
      user: "Ambre",
    };

    console.log("Analysis payload:", payload); // for debugging

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
    console.log("Analysis response data:", data); // for debugging

    console.log(
      "test!!!!!: data.data.outputs.result",
      data.data.outputs.result
    );

    // Process results according to actual API return format
    //analysisResult.value = data.answer || data.response || JSON.stringify(data)

    // fetch data.data.outputs.result
    if (data.data?.outputs?.result) {
      // Split the result into sections
      const formattedResult = data.data.outputs.result
        .split("\n\n")
        .map((section) => {
          return section.trim(); // remove leading/trailing spaces
        })
        .filter((section) => section); // remove empty sections

      analysisResult.value = formattedResult;
    } else {
      analysisResult.value = ["Unable to fetch analysis result"];
    }

    toast.add({
      severity: "success",
      summary: "Image Analysis Completed",
      detail: "Image Analysis Completed",
      life: 3000,
    });
  } catch (error) {
    console.error("Error analyzing image:", error);
    toast.add({
      severity: "error",
      summary: "error",
      detail: "Image Analysis Failed:" + error.message,
      life: 3000,
    });
    throw error;
  }
};

const formatMarkdown = (text) => {
  return (
    text
      // Bold
      .replace(/\*\*(.*?)\*\*/g, "<strong>$1</strong>")
      // Italic
      .replace(/\s*-\s+/g, "<br>• ")
      // Line break
      .replace(/\n/g, "<br>")
  );
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
.page-wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  padding: 20px;
  background-color: #f0f2f5;
}

.page-container {
  display: flex;
  width: 90vw;
  height: 80vh;
  max-width: 1400px;
  max-height: 900px;
  min-width: 900px;
  min-height: 600px;
  background-color: white;
  border-radius: 12px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  overflow: hidden;
}

.left-panel {
  width: 400px;
  height: 100%;
  padding: 2rem;
  background-color: #f8f9fa;
  border-right: 1px solid #dee2e6;
  display: flex;
  flex-direction: column;
  gap: 2rem;
}

.right-panel {
  flex-grow: 1;
  height: 100%;
  overflow-y: auto;
  padding: 2rem;
}

.header {
  text-align: center;
}

.header h1 {
  color: var(--primary-color);
  margin-bottom: 0.5rem;
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
  display: flex;
  flex-direction: column;
  gap: 2rem;
}

.preview-section,
.analysis-section {
  padding: 1rem;
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.preview-image {
  max-width: 100%;
  height: auto;
  border-radius: 8px;
}

.spinner {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 1000;
}

.result-card {
  background-color: #ffffff;
  border: 1px solid #e0e0e0;
}

.result-content {
  padding: 1.5rem;
}

.result-section {
  margin-bottom: 1.5rem;
  font-size: 1.1rem;
  line-height: 1.6;
  color: #333;
}

.result-section:last-child {
  margin-bottom: 0;
}

.result-section strong {
  color: #1a73e8;
}

.result-section br {
  display: block;
  margin: 0.5rem 0;
}

@media (max-width: 1200px) {
  .page-container {
    width: 100%;
    height: 100%;
    min-height: 600px;
  }
}

@media (max-width: 768px) {
  .page-container {
    flex-direction: column;
  }

  .left-panel {
    width: 100%;
    height: auto;
    min-height: 300px;
  }

  .right-panel {
    height: calc(100% - 300px);
  }
}
</style>
