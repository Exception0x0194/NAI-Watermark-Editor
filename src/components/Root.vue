<template>
  <h1>NAI隐匿水印修改器</h1>
  <p>修改NAI生成图像中的隐匿水印内容</p>

  <div style="margin: 0 auto">
    <el-upload class="upload-demo" drag multiple :before-upload="handleUpload">
      <el-icon class="el-icon--upload"><upload-filled /></el-icon>
      <div class="el-upload__text">拖动文件到这里或者点击选择文件</div>
    </el-upload>
  </div>

  <div v-if="imgFileRef && imgfileInfoRef" style="display: grid;">
    <div v-for="item in imgfileInfoRef" :key="item.key">
      <div style="border: solid grey 1px; margin-top:-1px; padding:5px">
        <span style="font: bold; align-self: flex-start;"> {{ item.key }}</span>
        <el-input v-model="item.value" type="textarea" style="white-space: pre-wrap;"
          :autosize="{ minRows: 1, maxRows: 20 }" />
      </div>
    </div>

    <div style="margin-top:10px; place-content:center right;">
      <el-button type="primary" @click="saveMetadata">保存元信息到隐匿水印</el-button>
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
import useClipboard from "vue-clipboard3";
import { UploadFilled } from "@element-plus/icons-vue";

import { asyncFileReaderAsDataURL, getStealthExif, embedStealthExif } from "../utils";

const imgFileRef = ref(null);
const imageRef = ref(null);
const exifRef = ref(null);
const imgfileInfoRef = ref(null);

const modelFileRef = ref(null);
const modelFileInfoRef = ref(null);

const jsonData = ref(null);
const imageMaxSizeRef = ref(0);
const { toClipboard } = useClipboard();

const availableImgExt = ["png", "jpeg", "jpg", "webp", "bmp"]

const copy = (value) => {
  try {
    toClipboard(value);
    ElMessage({
      message: "复制成功",
      type: "success",
    });
  } catch (e) {
    console.log(e);
    ElMessage({
      message: "复制失败",
      type: "warning",
    });
  }
};

const showCopyBtn = (title) => {
  if (!title) return false
  if (
    title == "Description" ||
    title == "Comment" ||
    title == "完整生成信息" ||
    title.indexOf("提示词") != -1
  ) {
    return true;
  }
  return false;
};

const cleanData = () => {
  imgFileRef.value = null
  modelFileRef.value = null
  imgfileInfoRef.value = null
  modelFileInfoRef.value = null
  exifRef.value = null
  jsonData.value = null
}

async function handleUpload(file) {
  console.log(file);
  cleanData()

  let fileExt = file.name.split(".").pop().toLowerCase();
  if (availableImgExt.indexOf(fileExt) != -1) {
    imgFileRef.value = file;
    inspectImage(file)
  } else {
    ElMessage({
      message: "解析失败，该文件可能不是一个正常的图片/模型文件。",
      type: "error",
    });
  }
  return false;
}

const inspectImage = async (file) => {
  await readImageBase64()
  // exifRef.value = await readExif(file)
  imgfileInfoRef.value = await readFileInfo(file)
}

async function readFileInfo(file) {
  jsonData.value = null
  let parsed = []

  let metadata = await getStealthExif(imageRef.value.src)
  if (metadata) {
    parsed = Object.keys(metadata).map((key) => {
      return {
        keyword: key,
        text: metadata[key],
      }
    });
  } else {
    return [{
      key: "提示",
      value: "无法读取到图像水印，这可能不是一张NAI生成的图片，或是经过压缩后丢失了水印的图片。",
    }];
  }

  let ok = []
  const commentJson = JSON.parse(metadata["Comment"]);
  ok.push({ key: "prompt", value: commentJson.prompt });
  ok.push({ key: "uc", value: commentJson.uc });
  ok.push({ key: "Title", value: metadata["Title"] });
  ok.push({ key: "Software", value: metadata["Software"] });
  ok.push({ key: "Source", value: metadata["Source"] });

  return ok
}

const readImageBase64 = async () => {
  imageRef.value = null;
  let result = await asyncFileReaderAsDataURL(imgFileRef.value)
  const image = new Image();
  image.src = result;
  await image.decode();
  const { width, height } = image;
  imageRef.value = {
    width,
    height,
    src: result,
  };
  imageMaxSizeRef.value = width;
}

const printableBytes = (size) => {
  const printable = (d, z) => {
    return `${d.toFixed(2)} ${z}`;
  };

  let kb = size / 1024;
  if (kb < 1024) {
    return printable(kb, "KB");
  }
  let mb = kb / 1024;
  if (mb < 1024) {
    return printable(mb, "MB");
  }

  let gb = mb / 1024;
  return printable(gb, "GB");
};

const saveMetadata = async () => {
  // 重新读取图片中的隐匿水印
  const existingMetadata = await getStealthExif(imageRef.value.src);

  // 从 imgfileInfoRef 中读取编辑后的数据
  const updatedMetadata = imgfileInfoRef.value.reduce((acc, item) => {
    acc[item.key] = item.value;
    return acc;
  }, {});

  // 更新隐匿水印中的信息
  if (existingMetadata) {
    // 更新 Comment 字段中的 prompt 和 uc 字段
    if (existingMetadata["Comment"]) {
      const commentJson = JSON.parse(existingMetadata["Comment"]);
      commentJson.prompt = updatedMetadata["prompt"];
      commentJson.uc = updatedMetadata["uc"];
      existingMetadata["Comment"] = JSON.stringify(commentJson);
    }

    // 更新水印中的 Description 字段
    existingMetadata["Description"] = updatedMetadata["prompt"];

    // 更新 Title、Software 和 Source 字段
    existingMetadata["Title"] = updatedMetadata["Title"];
    existingMetadata["Software"] = updatedMetadata["Software"];
    existingMetadata["Source"] = updatedMetadata["Source"];

    const imageDataUrl = imageRef.value.src;
    const outputImageDataUrl = await embedStealthExif(imageDataUrl, JSON.stringify(existingMetadata));

    // 替换页面中显示的图片
    imageRef.value.src = outputImageDataUrl;

    ElMessage({
      message: "元信息已保存并嵌入图片",
      type: "success",
    });
  } else {
    ElMessage({
      message: "无法读取原始水印信息，保存失败。",
      type: "error",
    });
  }
};

</script>
