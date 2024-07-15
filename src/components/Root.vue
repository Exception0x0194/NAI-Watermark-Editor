<template>
  <h1>NAI 隐匿水印修改器</h1>
  <p>批量修改 NAI 生成图像中的隐匿水印内容</p>

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

  <div v-if="filesRef.length > 0" style="margin-left:3px; margin-bottom:5px;display: flex;">
    已导入文件数量：{{ filesRef.length }}
  </div>

  <div v-if="firstFileInfoRef" style="display: grid;">
    <div v-for="item in firstFileInfoRef" :key="item.key">
      <div
        style="border: solid grey 1px; margin-top:-1px; padding:5px; display: flex; flex-direction: column;justify-content: flex-start;">
        <span style="font-weight: bold; align-self: start; margin-left:3px; margin-bottom:3px"> {{ item.key }}</span>
        <el-input v-model="item.value" type="textarea" style="white-space: pre-wrap;"
          :autosize="{ minRows: 1, maxRows: 20 }" />
      </div>
    </div>

    <div style="margin-top:10px; display: flex; justify-content: flex-end;">
      <el-button type="primary" @click="saveMetadata">保存图片到压缩包</el-button>
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

const availableImgExt = ["png", "jpeg", "jpg", "webp", "bmp"]

async function handleUpload(file: File) {
  console.log(file);

  let fileExt = file.name.split(".").pop()!.toLowerCase();
  if (availableImgExt.indexOf(fileExt) != -1) {
    // 加载文件信息
    loadImage(file);
    loadImageInfo(file);
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
  // 读取文件内容
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
      message: "无法加载或解码图片。",
      type: "error",
    });
  }
}

async function loadImageInfo(file: File) {
  // 读取文件内容
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

  // 读取需要显示的 metadata
  try {
    const u8array = new Uint8Array(fileBuffer!);
    const decodedString = decode_stealth_watermark(u8array);
    const metadata = JSON.parse(decodedString);

    let ok: { key: string, value: string }[] = [];
    const commentJson = JSON.parse(metadata["Comment"]);
    ok.push({ key: "prompt", value: commentJson.prompt });
    ok.push({ key: "uc", value: commentJson.uc });
    ok.push({ key: "Software", value: metadata["Software"] });
    ok.push({ key: "Source", value: metadata["Source"] });

    console.log(metadata);
    console.log(commentJson);

    firstFileInfoRef.value = ok;
  }
  catch (error) {
    console.error("Error loading metadata: ", error);
    ElMessage({
      message: "无法加载图片信息。",
      type: "warning",
    });
  }
}

const saveMetadata = async () => {
  ElMessage({
    message: "正在写入水印并打包...",
    type: "info",
  });

  // 创建 ZIP 文件的写入流
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
    // 读取文件内容
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
    // 重新读取隐匿水印；如果没有水印，读取预设内容
    const u8Array = new Uint8Array(fileBuffer!);
    let metadata = {};
    try {
      const decodedString = decode_stealth_watermark(u8Array);
      metadata = JSON.parse(decodedString);
    } catch (error) {

    }
    // 用编辑后的信息覆盖原水印中的内容
    metadata = updateMetadata(metadata);
    // 写入水印到图片中
    const u8ArrayWithWatermark = embed_stealth_watermark(u8Array, JSON.stringify(metadata));
    // 写入文件到 zip 流
    await zipWriter.add("👻-" + file.name, new Uint8ArrayReader(u8ArrayWithWatermark));
  }

  // 完成写入
  await zipWriter.close();

  ElMessage({
    message: "已保存压缩包",
    type: "success",
  });
};

function updateMetadata(metadata: Object) {
  const ret = metadata;
  // 读取编辑后的信息
  const updatedMetadata = firstFileInfoRef.value!.reduce((acc, item) => {
    acc[item.key] = item.value;
    return acc;
  }, {});
  // 更新 Comment 字段中的 prompt 和 uc 字段
  if (ret["Comment"]) {
    const commentJson = JSON.parse(ret["Comment"]);
    commentJson.prompt = updatedMetadata["prompt"];
    commentJson.uc = updatedMetadata["uc"];
    ret["Comment"] = JSON.stringify(commentJson);
  }
  // 更新水印中的 Description 字段
  ret["Description"] = updatedMetadata["prompt"];
  // 更新 Software 和 Source 字段
  ret["Software"] = updatedMetadata["Software"];
  ret["Source"] = updatedMetadata["Source"];

  return ret;
}

</script>
