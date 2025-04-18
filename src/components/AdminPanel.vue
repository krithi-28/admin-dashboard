<template>
  <a-layout class="admin-container">
    <a-upload-dragger
      :beforeUpload="handleFile"
      :fileList="[]"
      accept=".zip"
      class="upload-area"
    >
      <p class="ant-upload-drag-icon">
        <inbox-outlined />
      </p>
      <p class="ant-upload-text">Click or drag ZIP file to this area to upload</p>
    </a-upload-dragger>

    <p v-if="uploadedFileName" class="uploaded-filename">
      ✅ Uploaded: {{ uploadedFileName }}
    </p>


    <a-spin :spinning="loading" class="spinner-area">
      <div v-if="videoUrl" class="video-area">
        <h3>Screen Recording</h3>
        <video :src="videoUrl" controls width="100%"></video>
      </div>

      <div v-if="parsed.consoleLogs.length" class="log-area">
        <h3>Console Logs</h3>
        <a-collapse>
          <a-collapse-panel key="1" header="Log">
            <pre><code>{{ getLogsByLevel('log') }}</code></pre>
          </a-collapse-panel>
          <a-collapse-panel key="2" header="Error">
            <pre class="error-log"><code>{{ getLogsByLevel('error') }}</code></pre>
          </a-collapse-panel>
          <a-collapse-panel key="3" header="Warning">
            <pre class="warn-log"><code>{{ getLogsByLevel('warning') }}</code></pre>
          </a-collapse-panel>
        </a-collapse>
      </div>


      <div v-if="parsed.networkLogs.length" class="network-area">
        <h3>Network Logs</h3>
        <a-list
          bordered
        >
          <!-- Use v-for to render network items -->
          <a-list-item v-for="(item, index) in requestSummaries" :key="index">
            <div style="display: flex; justify-content: space-between; width: 100%;">
              <a-tooltip :title="item.url">
                <a @click="showNetworkDetails(item)" style="cursor: pointer; color: #1890ff;">
                  {{ item.url }}
                </a>
              </a-tooltip>
              <span class="req-id">(ID: {{ item.requestId }})</span>
            </div>
          </a-list-item>
        </a-list>
      </div>

      <div v-if="parsed.systemInfo" class="info-area">
        <h3>System Info</h3>
        <pre>{{ parsed.systemInfo }}</pre>
      </div>

      <div v-if="parsed.ipAddress" class="ip-area">
        <h3>IP Address</h3>
        <p>{{ parsed.ipAddress }}</p>
      </div>
    </a-spin>

    <a-modal
      :open="networkModal.visible"
      :footer="null"
      :centered="true"
      :maskClosable="false"
      :closable="true"
      title="Network Details"
      width="80%"
      @cancel="handleclose"
    >
      <div class="network-detail">
        <div class="left">
          <h4>Request</h4>
          <pre v-if="networkModal.request">{{ networkModal.request }}</pre>
          <p v-else class="placeholder">⚠️ Request not available</p>
        </div>
        <div class="right">
          <h4>Response</h4>
          <pre v-if="networkModal.response">{{ networkModal.response }}</pre>
          <p v-else class="placeholder">⚠️ Response not available</p>
        </div>
      </div>
    </a-modal>

  </a-layout>
</template>

  <script>
  import JSZip from "jszip";
  import { InboxOutlined } from "@ant-design/icons-vue";
  import { message } from "ant-design-vue";
  // import { Tooltip } from "ant-design-vue";
  
  export default {
    name:"AdminPanel",
    components: { InboxOutlined },
    data() {
      return {
        uploadedFileName:null,
        loading: false,
        videoUrl: null,
        parsed: {
          consoleLogs: [],
          networkLogs: [],
          systemInfo: '',
          ipAddress: ''
        },
        networkModal: {
          visible: false,
          request: '',
          response: ''
        }
      };
    },
    computed: {
      requestSummaries() {
        return this.parsed.networkLogs
          .filter(log => log.type === 'request')
          .map(req => {
            return {
              requestId: req.requestId,
              url: req.url,
              request: req,
              response: this.parsed.networkLogs.find(res => res.type === 'response' && res.requestId === req.requestId)
            };
          });
      }
    },
    methods: {
      handleclose(){
        this.networkModal.visible=false;
      },

      stringifySafe(value) {
        try {
          if (typeof value === 'object') {
            const seen = new WeakSet();
            return JSON.stringify(value, function (key, val) {
              if (typeof val === 'object' && val !== null) {
                if (seen.has(val)) return '[Circular]';
                seen.add(val);
              }
              return val;
            }, 2); // pretty-print with 2 spaces
          } else {
            return String(value);
          }
        } catch (e) {
          return '[Unserializable Object]';
        }
      },



      async handleFile(file) {
        this.loading = true;
        this.uploadedFileName=file.name;
        const zip = await JSZip.loadAsync(file);
        const parsed = {
          consoleLogs: [],
          networkLogs: [],
          systemInfo: '',
          ipAddress: ''
        };
        let videoBlob = null;
  
        for (const filename of Object.keys(zip.files)) {
          const fileData = await zip.files[filename].async("string");
          if (filename.includes("Console")) {
            parsed.consoleLogs = JSON.parse(fileData);
          } else if (filename.includes("Network")) {
            parsed.networkLogs = JSON.parse(fileData);
          } else if (filename.includes("System")) {
            parsed.systemInfo = fileData;
          } else if (filename.includes("IP")) {
            parsed.ipAddress = fileData;
          } else if (filename.endsWith(".mp4")) {
            videoBlob = await zip.files[filename].async("blob");
          }
        }
  
        if (videoBlob) {
          this.videoUrl = URL.createObjectURL(videoBlob);
        }
  
        this.parsed = parsed;
        this.loading = false;
        message.success("ZIP processed successfully");
        return false;
      },
      getLogsByLevel(level) {
        return this.parsed.consoleLogs
          .filter(log => log.level === level)
          .map(log => {
            const formattedText = this.stringifySafe(log.text);
            return `[${log.timestamp}] ${formattedText}`;
          })
          .join('\n\n'); // Add spacing between logs
      },
      showNetworkDetails(item) {
        this.networkModal.request = item.request
          ? JSON.stringify(item.request, null, 2)
          : '⚠️ Request not available';

        this.networkModal.response = item.response
          ? JSON.stringify(item.response, null, 2)
          : '⚠️ Response not available';

        this.networkModal.visible = true;
      }

    }
  };
  </script>
  
  <style scoped>
  .admin-container {
    padding: 20px;
  }
  .upload-area {
    margin-bottom: 30px;
  }
  .spinner-area {
    min-height: 300px;
  }
  .video-area, .log-area, .network-area, .info-area, .ip-area {
    margin-bottom: 20px;
    background: #fff;
    padding: 20px;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  }
  .network-detail {
    display: flex;
    gap: 20px;
  }
  .network-detail .left, .network-detail .right {
    flex: 1;
    background: #f5f5f5;
    padding: 15px;
    border-radius: 6px;
    overflow: auto;
    max-height: 500px;
  }
  .req-id {
    margin-left: 10px;
    color: #aaa;
  }

  .log-area pre {
  white-space: pre-wrap;
  word-break: break-word;
  background-color: #f5f5f5;
  padding: 10px;
  border-radius: 6px;
  font-size: 13px;
  line-height: 1.6;
  font-family: monospace;
  max-height: 400px;
  overflow-y: auto;
}

.error-log {
  background-color: #ffeaea;
  color: #b00020;
}

.warn-log {
  background-color: #fff8e1;
  color: #8c6d00;
}

.network-detail {
  display: flex;
  gap: 20px;
  justify-content: space-between;
}

.network-detail .left,
.network-detail .right {
  flex: 1;
  max-width: 48%;
}

.network-detail pre {
  background: #f7f7f7;
  padding: 12px;
  border-radius: 6px;
  font-family: monospace;
  font-size: 13px;
  overflow-x: auto;
}

.network-detail .placeholder {
  color: #888;
  font-style: italic;
}


  </style>
  