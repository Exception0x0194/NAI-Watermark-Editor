<template>
  <h1>NAI 隐匿水印编辑器</h1>
  <p>批量编辑 NAI 生成图像中的隐匿水印内容</p>

  <div v-if="firstImageRef">
    <div style="border: solid gray 1px; margin-bottom: 10px; max-width: 720px; height: 40vh">
      <img v-bind="firstImageRef" alt=""
        style="display: block; width: auto; max-width:720px; height: 40vh; margin:auto" />
    </div>
  </div>

  <div style="margin: 0 auto">
    <el-upload class="upload-demo" drag multiple :before-upload="handleUpload">
      <el-icon class="el-icon--upload"><upload-filled /></el-icon>
      <div class="el-upload__text">拖动文件到这里或者<em>点击导入文件</em></div>
    </el-upload>
  </div>

  <div v-if="filesRef.length > 0"
    style="margin:10px; display: flex; justify-content: space-between; align-items: center;">
    <span style="">已导入文件数量：{{ filesRef.length }}</span>
    <div style="">
      <el-button type="primary" @click="saveMetadata">导出图片</el-button>
    </div>
  </div>

  <div v-if="firstFileInfoRef" style="display: grid;">
    <div v-for="item in firstFileInfoRef" :key="item.key">
      <div
        style="border: solid grey 1px; border-radius: 5px; margin-top:5px; padding:5px; display: flex; flex-direction: column;justify-content: flex-start;">
        <span style="font-weight: bold; align-self: start; margin-left:3px; margin-bottom:3px">
          {{ item.key }}</span>
        <el-input v-model="item.value" type="textarea" style="white-space: pre-wrap;"
          :autosize="{ minRows: 1, maxRows: 20 }" />
      </div>
    </div>
  </div>


  <p>*运算完全在本地运行</p>
  <div>
    <a href="https://github.com/Exception0x0194/NAI-Watermark-Editor">GitHub</a>
    <br />
  </div>
</template>

<script setup lang="ts">
import { ElMessage } from "element-plus";
import { ref } from "vue";
import { UploadFilled } from "@element-plus/icons-vue";
import { embed_stealth_watermark, decode_stealth_watermark } from "stealth-watermark-editor";
import { ZipWriter, Uint8ArrayReader } from '@zip.js/zip.js';
import { createWriteStream } from 'streamsaver';

const filesRef = ref<File[]>([]);
const firstImageRef = ref<{ width: number, height: number, src: string } | null>(null);
const firstFileInfoRef = ref<{ key: string, value: string }[] | null>(null);

const availableImgExt = ["png", "webp", "bmp"];

const prioritizedCommentKeys = [
  "prompt",
  "uc"
];
const filteredMetadataKeys = [
  "Description",
  "Comment"
];
let metadataKeys: string[] = [], commentKeys: string[] = [];

async function handleUpload(file: File) {
  console.log(file);

  let fileExt = file.name.split(".").pop()!.toLowerCase();
  if (availableImgExt.indexOf(fileExt) != -1) {
    // Load image / metadata
    await loadImage(file);
    await loadImageInfo(file);
    filesRef.value = [...filesRef.value, file];
  } else {
    ElMessage({
      message: "不支持的文件类型：" + file.name,
      type: "error",
    });
  }
  return false;
}

async function loadImage(file: File) {
  if (firstImageRef.value !== null) return;
  // Read file
  let fileBuffer: string | null = null;
  const loadPromise = new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = (e) => {
      fileBuffer = e.target!.result as string;
      resolve(e);
    };
    reader.onerror = (e) => {
      reject(e);
    };
    reader.readAsDataURL(file);
  });
  await loadPromise;

  try {
    const image = new Image();
    image.src = fileBuffer!;
    await image.decode();

    const { width, height } = image;
    firstImageRef.value = { width, height, src: fileBuffer! };
  } catch (error) {
    console.error("Error loading image: ", error);
    ElMessage({
      message: "无法解码的图片：" + file.name,
      type: "error",
    });
  }
}

async function loadImageInfo(file: File) {
  if (firstFileInfoRef.value !== null) return;
  // Read file
  let fileBuffer: ArrayBuffer | null = null;
  const loadPromise = new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = (e) => {
      fileBuffer = e.target!.result as ArrayBuffer;
      resolve(e);
    };
    reader.onerror = (e) => {
      reject(e);
    };
    reader.readAsArrayBuffer(file);
  });
  await loadPromise;

  // Read metadata to be displayed
  try {
    const u8array = new Uint8Array(fileBuffer!);
    const decodedString = decode_stealth_watermark(u8array);

    const metadataJson = JSON.parse(decodedString);
    const commentJson = JSON.parse(metadataJson["Comment"]);

    metadataKeys = Object.keys(metadataJson);
    commentKeys = Object.keys(commentJson);

    const ret: { key: string, value: string }[] = [];
    for (const key of prioritizedCommentKeys) {
      // Push prompt and UC (from comments) first
      ret.push({ key: key, value: String(commentJson[key]) });
    }
    for (const key of metadataKeys) {
      // Remaining values in metadata
      if (filteredMetadataKeys.indexOf(key) !== -1) continue;
      ret.push({ key: key, value: String(metadataJson[key]) });
    }
    for (const key of commentKeys) {
      // Remaining values in comments
      if (prioritizedCommentKeys.indexOf(key) !== -1) continue;
      // Use "[]" to init if obj is empty array
      const obj = Array.isArray(commentJson[key]) && commentJson[key].length === 0 ? "[]" : String(commentJson[key]);
      ret.push({ key: key, value: obj });
    }
    console.log(ret);

    firstFileInfoRef.value = ret;
  }
  catch (error) {
    console.error("Error loading metadata: ", error);
    ElMessage({
      message: "没有元信息的图片：" + file.name,
      type: "warning",
    });
  }
}

const saveMetadata = async () => {
  ElMessage({
    message: "正在写入水印...",
    type: "info",
  });
  const metadata = getMetadata();
  if (filesRef.value.length == 1) {
    // Single file: download as Blob
    const file = filesRef.value[0];
    const u8ArrayWithWatermark = await getEmbeddedImageBytes(file, metadata);
    // Convert the modified Uint8Array back to Blob
    const blob = new Blob([u8ArrayWithWatermark], { type: "application/octet-stream" });
    // Create a link and trigger the download
    const url = window.URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = "👻-" + file.name;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    window.URL.revokeObjectURL(url); // Clean up URL object
  }
  else {
    // Multiple files: create a ZIP stream
    const fileStream = createWriteStream("images_with_edited_watermark.zip");
    const writer = fileStream.getWriter();
    const zipWriter = new ZipWriter(new WritableStream({
      write(chunk) {
        return writer.write(chunk);
      },
      close() {
        writer.close();
      }
    }));
    for (const file of filesRef.value) {
      // Get file bytes
      const u8ArrayWithWatermark = await getEmbeddedImageBytes(file, metadata);
      // Write into ZIP stream
      await zipWriter.add("👻-" + file.name, new Uint8ArrayReader(u8ArrayWithWatermark));
    }
    // Close ZIP stream
    await zipWriter.close();
  }
  ElMessage({
    message: "已保存图片",
    type: "success",
  });
};

async function getEmbeddedImageBytes(file: File, metadata: Object) {
  // Read files
  let fileBuffer: ArrayBuffer | null = null;
  const loadPromise = new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = (e) => {
      fileBuffer = e.target!.result as ArrayBuffer;
      resolve(e);
    };
    reader.onerror = (e) => {
      reject(e);
    };
    reader.readAsArrayBuffer(file);
  });
  await loadPromise;
  const u8Array = new Uint8Array(fileBuffer!);
  let u8ArrayWithWatermark = new Uint8Array;
  try {
    // Embed metadata into image
    u8ArrayWithWatermark = embed_stealth_watermark(u8Array, JSON.stringify(metadata));
  } catch (error) {
    console.log("Error writing watermark: ", error);
  }
  return u8ArrayWithWatermark;
}

function getMetadata() {
  if (firstFileInfoRef.value === null) {
    return {};
  }
  const metadataJson = {}, commentJson = {};
  for (const item of firstFileInfoRef.value) {
    const tryParse = (value: string) => {
      try {
        return JSON.parse(value);
      } catch (error) {
        return value;
      }
    }
    const obj = tryParse(item.value);
    if (metadataKeys.indexOf(item.key) !== -1) {
      // Metadata item
      metadataJson[item.key] = obj;
    } else {
      // Comment item
      commentJson[item.key] = obj;
      // Description
      if (item.key === "prompt") metadataJson["Description"] = obj;
    }
  }
  // Stringified comment
  metadataJson["Comment"] = JSON.stringify(commentJson);
  return metadataJson;
}

</script>
