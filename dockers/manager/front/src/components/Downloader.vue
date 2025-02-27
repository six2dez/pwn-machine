<template>
  <q-slide-transition>
    <q-card v-show="downloading.length" class="downloader" bordered>
      <q-card-section
        v-for="[idx, downloadInfo] of downloading.entries()"
        :key="idx"
        class="q-py-none q-pl-sm"
      >
        <q-circular-progress
          size="50px"
          show-value
          instant-feedback
          :value="downloadInfo.progress"
          :color="downloadInfo.progress === 100 ? 'green' : 'lime'"
          class="q-ma-sm q-py-none"
        />
        {{ downloadInfo.name }}
      </q-card-section>
    </q-card>
  </q-slide-transition>
</template>

<script>
import { GlobalBus } from "src/eventBus.js";

export default {
  data: () => ({ value: 0, downloading: [] }),
  created() {
    GlobalBus.$on("startFileDownload", this.downloadFile);
  },
  destroyed() {
    GlobalBus.$off("startFileDownload", this.downloadFile);
  },
  methods: {
    downloadBlob(blob, downloadInfo) {
      const url = URL.createObjectURL(blob);

      const a = document.createElement("a");
      a.href = url;
      a.download = downloadInfo.name;
      a.style.display = "none";
      document.body.appendChild(a);
      a.click();
      a.remove();
      setTimeout(() => {
        this.downloading = this.downloading.filter(d => d !== downloadInfo);
        URL.revokeObjectURL(url);
      });
    },

    async followProgress(reader, downloadInfo) {
      const chunks = [];
      let downloaded = 0;
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        downloaded += value.length;
        downloadInfo.progress = Math.floor(
          (downloaded / downloadInfo.size) * 100
        );
        chunks.push(value);
      }
      const blob = new Blob(chunks.flat(), {
        type: downloadInfo.mime
      });

      this.downloadBlob(blob, downloadInfo);
    },

    async downloadFile({volume, path, file}) {
      const params = new URLSearchParams({ volume, path });
      const token = localStorage.getItem("token");
      const response = await fetch(`/file/download?${params}`, {
        headers: {
          Authorization: `Bearer ${token}`
        }
      });
      const downloadInfo = {
        name: path.split("/").pop(),
        size: file.size,
        mime: file.mime,
        progress: 0,
        chunks: []
      };
      this.downloading.push(downloadInfo);
      const reader = response.body.getReader();
      this.followProgress(reader, downloadInfo);
    }
  }
};
</script>

<style lang="scss" scoped>
.downloader {
  position: fixed;
  bottom: 10px;
  right: 10px;
  z-index: 10000;
}
</style>
