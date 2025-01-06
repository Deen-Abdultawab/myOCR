<template>
  <div class="p-6 bg-gray-100 min-h-screen flex items-center justify-center">
    <div class="max-w-xl w-full">
      <div
        class="border-2 border-dashed border-gray-300 rounded-lg p-6 text-center bg-white"
        @dragover.prevent="dragOver = true"
        @dragleave.prevent="dragOver = false"
        @drop.prevent="handleDrop"
        :class="dragOver ? 'border-blue-500 bg-blue-50' : ''"
      >
        <p class="text-gray-500">Drag and drop your images here</p>
        <p class="text-gray-400 mt-2">or</p>
        <label
          for="file-input"
          class="cursor-pointer text-blue-500 hover:underline"
        >
          browse to upload
        </label>
        <input
          id="file-input"
          type="file"
          class="hidden"
          multiple
          accept="image/*"
          @change="handleFileInput"
        />
      </div>

      <!-- Preview Section -->
      <div v-if="images.length" class="mt-4">
        <h3 class="text-gray-700 font-semibold">Preview:</h3>
        <div class="grid grid-cols-2 sm:grid-cols-3 gap-4 mt-2">
          <div
            v-for="(image, index) in images"
            :key="index"
            class="relative"
          >
            <img
              :src="image.preview"
              alt="Uploaded image"
              class="w-full h-32 object-cover rounded-lg"
            />
            <button
              @click="removeImage(index)"
              class="absolute top-1 right-1 bg-red-500 text-white text-xs px-2 py-1 rounded-full"
            >
              X
            </button>
          </div>
        </div>
      </div>

      <!-- Process Button -->
      <button
        v-if="images.length"
        @click="processOCR"
        class="mt-4 bg-blue-500 text-white py-2 px-4 rounded-lg"
      >
        <span v-if="isLoading">Loading...</span>
        <span v-else>Process OCR</span>
      </button>

      <!-- OCR Results -->
      <div v-if="ocrResults.length" class="mt-6">
        <h3 class="text-gray-700 font-semibold">OCR Results:</h3>
        <div
          v-for="(result, index) in ocrResults"
          :key="index"
          class="bg-white p-4 rounded-lg shadow mt-4"
        >
          <h4 class="font-bold mb-2">Image {{ index + 1 }}</h4>
          <h3 class="text-gray-700">{{ result }}</h3>
          <!-- <div>{{ extractedDetails }}</div> -->
        </div>
      </div>

      <button v-if="showExportBtn" @click="processOCRAndExport" class="bg-blue-500 text-white px-4 py-2 rounded mt-4">
        Export to Excel
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue";
import ExcelJS from 'exceljs';

const images = ref([]);
const dragOver = ref(false);
const ocrResults = ref([]);
const isLoading = ref(false)
const extractedDetails = ref({})
const showExportBtn = ref(false)

const handleFileInput = (event) => {
  const files = event.target.files;
  processFiles(files);
};

const handleDrop = (event) => {
  dragOver.value = false;
  const files = event.dataTransfer.files;
  processFiles(files);
};

const processFiles = (files) => {
  Array.from(files).forEach((file) => {
    if (file.type.startsWith("image/")) {
      const reader = new FileReader();
      reader.onload = (e) => {
        images.value.push({
          file,
          preview: e.target.result,
        });
      };
      reader.readAsDataURL(file);
    }
  });
};

const removeImage = (index) => {
  images.value.splice(index, 1);
};

// Process OCR for multiple images
const processOCR = async () => {
  if (!images.value.length) return;

  ocrResults.value = []; // Reset previous results
  const apiKey = "K81563776888957"; // Replace with your actual API key
  const apiUrl = "https://api.ocr.space/parse/image";

  for (const image of images.value) {
    const formData = new FormData();
    formData.append("file", image.file);
    formData.append("apikey", apiKey);
    formData.append("language", "eng");
    formData.append("detectOrientation", "true");
    formData.append("isTable", "true");

    try {
      isLoading.value = true
      const response = await fetch(apiUrl, {
        method: "POST",
        body: formData,
      });
      const result = await response.json();
      console.log(result)

      if (result.IsErroredOnProcessing) {
        ocrResults.value.push("Error: " + result.ErrorMessage.join(", "));
        isLoading.value = false
      } else {
        const parsedText = result.ParsedResults[0]?.ParsedText || "No text found";
        extractedDetails.value = extractDetails(parsedText)
        ocrResults.value.push(extractedDetails);

        console.log(ocrResults.value)
        isLoading.value = false
        showExportBtn.value = true
      }
    } catch (error) {
      ocrResults.value.push("Error processing the image: " + error.message);
      isLoading.value = false
    }
  }
};

const extractDetails = (ocrText) => {
  const details = {
    gross: null,
    tare: null,
    issueDate: null,
  };

  // Regular expressions to match the specific details
  const grossRegex = /Gross(?:\s*\([^)]+\))?\s*([\d]+)/i;
  const tareRegex = /Tare(?:\s*\([^)]+\))?\s*([\d]+)/i;
  const issueDateRegex = /Issue Date\s*:\s*([\d/]+)/i;

  // Extracting Gross
  const grossMatch = ocrText.match(grossRegex);
  if (grossMatch) {
    details.gross = grossMatch[1];
  }

  // Extracting Tare
  const tareMatch = ocrText.match(tareRegex);
  if (tareMatch) {
    details.tare = tareMatch[1];
  }

  // Extracting Issue Date
  const issueDateMatch = ocrText.match(issueDateRegex);
  if (issueDateMatch) {
    details.issueDate = issueDateMatch[1];
  }

  return details;
};

const exportToExcel = async (groupedData) => {
  const workbook = new ExcelJS.Workbook();

  for (const date in groupedData) {
    // Sanitize the worksheet name
    const sanitizedDate = date.replace(/[\/\\*?[\]:]/g, "-");

    // Add a new worksheet with the sanitized name
    const worksheet = workbook.addWorksheet(sanitizedDate);

    // Add Headers
    worksheet.columns = [
      { header: "Gross (Kg)", key: "gross", width: 15 },
      { header: "Tare (Kg)", key: "tare", width: 15 },
    ];

    // Ensure the original `date` key is used to retrieve data
    const entries = groupedData[date];
    if (entries && Array.isArray(entries)) {
      entries.forEach((entry) => {
        worksheet.addRow(entry);
      });
    } else {
      console.warn(`No data found for date: ${date}`);
    }
  }

  // Save Excel File
  const buffer = await workbook.xlsx.writeBuffer();
  const blob = new Blob([buffer], { type: "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet" });
  const link = document.createElement("a");
  link.href = URL.createObjectURL(blob);
  link.download = "OCR_Report.xlsx";
  link.click();
};


const processOCRAndExport = async () => {
  // await processOCR();

  const groupedData = ocrResults.value.reduce((acc, result) => {
    const { issueDate, gross, tare } = result;
    if (!acc[issueDate]) acc[issueDate] = [];
    acc[issueDate].push({ gross, tare });
    return acc;
  }, {});

  await exportToExcel(groupedData);
};

</script>

<style scoped>
/* Additional optional styling */
</style>
