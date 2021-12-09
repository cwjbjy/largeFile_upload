<template>
  <div>
    <input type="file" @change="handleFileChange" :disabled="status !== Status.wait" />
    <el-button @click="handleUpload" :disabled="uploadDisabled">上传</el-button>
    <div>计算文件 hash</div>
    <el-progress :percentage="hashPercentage"></el-progress>
    <div>总进度</div>
    <el-progress :percentage="fakeUploadPercentage"></el-progress>
    <el-button @click="handleResume" v-if="status === Status.pause">恢复</el-button>
    <el-button
      v-else
      :disabled="status !== Status.uploading || !container.hash"
      @click="handlePause"
      >暂停</el-button
    >
    <el-table :data="data">
      <el-table-column prop="hash" label="切片hash" align="center"></el-table-column>
      <el-table-column label="大小(KB)" align="center" width="120">
        <template v-slot="{ row }">
          {{ row.size | transformByte }}
        </template>
      </el-table-column>
      <el-table-column label="进度" align="center">
        <template v-slot="{ row }">
          <el-progress :percentage="row.percentage" color="#909399"></el-progress>
        </template>
      </el-table-column>
    </el-table>
  </div>
</template>

<script>
const SIZE = 10 * 1024 * 1024; // 切片大小
const Status = {
  wait: "wait",
  pause: "pause",
  uploading: "uploading",
};
export default {
  data() {
    return {
      //存储文件
      container: {
        file: null,
        hash: "",
        worker: null,
      },
      //存储切片后的文件
      data: [],
      //计算hash进度
      hashPercentage: 0,
      // 当暂停时会取消 xhr 导致进度条后退
      // 为了避免这种情况，需要定义一个假的进度条
      fakeUploadPercentage: 0,
      Status,
      status: Status.wait,
      requestList: [], //存储文件切片HTTP请求
    };
  },
  computed: {
    uploadDisabled() {
      return (
        !this.container.file || [Status.pause, Status.uploading].includes(this.status)
      );
    },
    //真实进度条
    uploadPercentage() {
      if (!this.container.file || !this.data.length) return 0;
      const loaded = this.data
        .map((item) => item.size * item.percentage)
        .reduce((acc, cur) => acc + cur);
      return parseInt((loaded / this.container.file.size).toFixed(2));
    },
  },
  watch: {
    uploadPercentage(now) {
      if (now > this.fakeUploadPercentage) {
        //呈现的进度条
        this.fakeUploadPercentage = now;
      }
    },
  },
  methods: {
    handlePause() {
      this.status = Status.pause;
      this.resetData();
    },
    resetData() {
      this.requestList.forEach((xhr) => xhr?.abort());
      this.requestList = [];
      if (this.container.worker) {
        this.container.worker.onmessage = null;
      }
    },
    async handleResume() {
      this.status = Status.uploading;
      //uploadedList：已上传文件切片列表
      const { uploadedList } = await this.verifyUpload(
        this.container.file.name,
        this.container.hash
      );
      await this.uploadChunks(uploadedList);
    },
    handleFileChange(e) {
      const [file] = e.target.files;
      if (!file) return;
      this.resetData();
      Object.assign(this.$data, this.$options.data());
      this.container.file = file;
    },
    request({
      url,
      method = "post",
      data,
      headers = {},
      onProgress = (e) => e,
      requestList,
    }) {
      return new Promise((resolve) => {
        const xhr = new XMLHttpRequest();
        xhr.upload.onprogress = onProgress;
        xhr.open(method, url);
        Object.keys(headers).forEach((key) => xhr.setRequestHeader(key, headers[key]));
        xhr.send(data);
        xhr.onload = (e) => {
          // 将请求成功的 xhr 从列表中删除
          if (requestList) {
            const xhrIndex = requestList.findIndex((item) => item === xhr);
            requestList.splice(xhrIndex, 1);
          }
          resolve({
            data: e.target.response,
          });
        };
        //将当前请求存入this.requestList中
        requestList?.push(xhr);
      });
    },
    async handleUpload() {
      if (!this.container.file) return;
      this.status = Status.uploading;
      const fileChunkList = this.createFileChunk(this.container.file);
      this.container.hash = await this.calculateHash(fileChunkList);
      const { shouldUpload, uploadedList } = await this.verifyUpload(
        this.container.file.name,
        this.container.hash
      );
      if (!shouldUpload) {
        this.status = Status.wait;
        this.$message.success("秒传：上传成功");
        return;
      }
      this.data = fileChunkList.map(({ file }, index) => ({
        chunk: file,
        index,
        hash: this.container.hash + "-" + index, // 文件名 + 数组下标
        percentage: uploadedList.includes(index) ? 100 : 0, //已上传的切片的进度为100%
        size: file.size,
      }));
      await this.uploadChunks(uploadedList);
    },
    // 生成文件切片
    createFileChunk(file, size = SIZE) {
      const fileChunkList = [];
      let cur = 0;
      while (cur < file.size) {
        fileChunkList.push({ file: file.slice(cur, cur + size) });
        cur += size;
      }
      return fileChunkList;
    },
    // 上传切片，同时过滤已上传的切片（通过内容生成的hash名判断）
    async uploadChunks(uploadedList = []) {
      const requestList = this.data
        .filter(({ hash }) => !uploadedList.includes(hash))
        .map(({ chunk, hash, index }) => {
          const formData = new FormData();
          formData.append("chunk", chunk);
          formData.append("hash", hash);
          formData.append("filename", this.container.file.name);
          formData.append("fileHash", this.container.hash);
          return { formData, index };
        })
        .map(async ({ formData, index }) =>
          this.request({
            url: "http://localhost:3001",
            data: formData,
            onProgress: this.createProgressHandler(this.data[index]),
            requestList: this.requestList,
          })
        );
      await Promise.all(requestList); // 并发切片
      // 之前上传的切片数量 + 本次上传的切片数量 = 所有切片数量时
      // 合并切片
      if (uploadedList.length + requestList.length === this.data.length) {
        await this.mergeRequest();
      }
    },
    //生成合并文件请求
    async mergeRequest() {
      await this.request({
        url: "http://localhost:3001/merge",
        headers: {
          "content-type": "application/json",
        },
        data: JSON.stringify({
          filename: this.container.file.name,
          fileHash: this.container.hash,
          size: SIZE,
        }),
      });
      this.$message.success("上传成功");
      this.status = Status.wait;
    },
    // 根据 hash 验证文件是否曾经已经被上传过
    // 没有才进行上传
    async verifyUpload(filename, fileHash) {
      const { data } = await this.request({
        url: "http://localhost:3001/verify",
        headers: {
          "content-type": "application/json",
        },
        data: JSON.stringify({
          filename,
          fileHash,
        }),
      });
      return JSON.parse(data);
    },
    // 生成文件 hash（web-worker）
    calculateHash(fileChunkList) {
      return new Promise((resolve) => {
        this.container.worker = new Worker("/hash.js");
        this.container.worker.postMessage({ fileChunkList });
        this.container.worker.onmessage = (e) => {
          const { percentage, hash } = e.data;
          //接收计算文件hash进度
          this.hashPercentage = percentage;
          if (hash) {
            resolve(hash);
          }
        };
      });
    },
    // 用闭包保存每个 chunk 的进度数据
    createProgressHandler(item) {
      return (e) => {
        item.percentage = parseInt(String((e.loaded / e.total) * 100));
      };
    },
  },
};
</script>
