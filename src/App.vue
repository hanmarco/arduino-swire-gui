<script setup>
import { ref, onMounted, onUnmounted, watch, nextTick } from "vue";
// src/App.vue 또는 사용하는 파일
import { SerialPort } from "tauri-plugin-serialplugin";
import pkg from '../package.json';

const ports = ref([]);
const portDetails = ref([]);
const selectedPort = ref("");
const isConnected = ref(false);
const dutyValue = ref(1);
const pulseValue = ref(5);
const frequencyValue = ref("100");
const frequencies = ref(["100", "200", "400"]);
const receivedMessages = ref([]);
let serialPortInstance = null;
let unlisten = null;
const messagesContainer = ref(null);

// 메시지가 추가될 때마다 스크롤을 최하단으로 이동
watch(
  () => receivedMessages.value.length,
  async () => {
    await nextTick() // v-for 렌더 완료 대기
    const el = messagesContainer.value
    if (!el) return
    el.scrollTop = el.scrollHeight // 최하단으로 이동
  },
  { flush: 'post' } // DOM 업데이트 이후 콜백 실행
);

async function listPorts() {
  try {
    const availablePorts = await SerialPort.available_ports();
    console.log('Available ports:', availablePorts); // 데이터 구조 확인
    
    // 포트 데이터가 배열이 아닌 경우를 처리
    const portsArray = Array.isArray(availablePorts) ? availablePorts : [availablePorts];
    // 각 포트의 전체 정보 저장
    const portDetails = portsArray.map(p => ({
      name: p.port_name || p.portName || p,
      product: p.product || ''
    }));
    // 포트 이름만 따로 저장
    ports.value = portDetails.map(p => p.name);
    // 포트 목록 디버깅
    console.log('Port details:', portDetails);
    
    if (ports.value.length > 0) {
      selectedPort.value = ports.value[0];
    } else {
      receivedMessages.value.push('No available ports found');
    }
  } catch (error) {
    console.error("Error listing ports:", error);
    receivedMessages.value.push(`Error listing ports: ${error.message}`);
  }
}

async function toggleConnection() {
  if (isConnected.value) {
    // Disconnect
    if (unlisten) {
      unlisten();
      unlisten = null;
    }
    if (serialPortInstance) {
      await serialPortInstance.close();
      serialPortInstance = null;
      receivedMessages.value.push(`Disconnected from ${selectedPort.value}`);
    }
    isConnected.value = false;
  } else {
    // Connect
    if (!selectedPort.value) {
      receivedMessages.value.push("Please select a COM port.");
      return;
    }
    try {
      serialPortInstance = new SerialPort(selectedPort.value, {
        baud_rate: 9600, // Default baud rate, can be made configurable
      });
      await serialPortInstance.open();
      unlisten = await serialPortInstance.listen((data) => {
        const decoder = new TextDecoder();
        receivedMessages.value.push(`Received: ${decoder.decode(data.data)}`);
      });
      isConnected.value = true;
      receivedMessages.value.push(`Connected to ${selectedPort.value}`);
    } catch (error) {
      console.error("Error connecting:", error);
      receivedMessages.value.push(`Error connecting: ${error.message}`);
      isConnected.value = false;
      serialPortInstance = null;
    }
  }
}

async function sendMessage(message) {
  if (!isConnected.value || !serialPortInstance) {
    receivedMessages.value.push("Not connected to a COM port.");
    return;
  }
  try {
    const encoder = new TextEncoder();
    await serialPortInstance.write(encoder.encode(message + '\n')); // Add newline for command termination
    receivedMessages.value.push(`Sent: ${message}`);
  } catch (error) {
    console.error("Error sending message:", error);
    receivedMessages.value.push(`Error sending: ${error.message}`);
  }
}

function setDuty() {
  const duty = parseInt(dutyValue.value);
  if (isNaN(duty) || duty < 1 || duty > 10) {
    receivedMessages.value.push("Duty value must be between 1 and 10.");
    return;
  }
  sendMessage(`d ${duty}`);
}

function sendS9() {
  sendMessage(`s9 ${pulseValue.value}`);
}

function sendS10() {
  sendMessage(`s10 ${pulseValue.value}`);
}

function setFrequency() {
  const freq = frequencyValue.value;
  sendMessage(`f${freq}`);
}

onMounted(() => {
  listPorts();
  // 초기 진입 시에도 스크롤을 최하단으로 이동
  nextTick(() => {
    const el = messagesContainer.value;
    if (!el) return;
    el.scrollTop = el.scrollHeight;
  });
});

onUnmounted(() => {
  if (isConnected.value) {
    toggleConnection(); // Ensure disconnection on component unmount
  }
});
</script>

<template>
  <main class="container">
    <h1 class="title-row">
      <a href="https://github.sec.samsung.net/sss-han/arduino-swire-gui" target="_blank" class="github-link" title="View source on GitHub">
        <svg height="24" width="24" viewBox="0 0 16 16" class="github-icon">
          <path fill="currentColor" d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"></path>
        </svg>
      </a>
      Arduino Swire GUI 
      <span class="version">v{{ pkg.version }}</span>
    </h1>

    <div class="row">
      <label for="com-port-select">COM Port:</label>
      <select id="com-port-select" v-model="selectedPort">
        <option v-for="port in portDetails" :key="port.name" :value="port.name">
          {{ port.product ? `${port.name} (${port.product})` : port.name }}
        </option>
      </select>
      <button class="icon-button" @click="listPorts" :disabled="isConnected" title="Refresh port list">
        🔄
      </button>
      <button @click="toggleConnection">
        {{ isConnected ? "Disconnect" : "Connect" }}
      </button>
    </div>

    <div class="row">
      <label for="duty-input">Duty (1-10):</label>
      <input id="duty-input" type="number" v-model.number="dutyValue" min="1" max="10" :disabled="!isConnected" />
      <button @click="setDuty" :disabled="!isConnected">Set Duty</button>
    </div>

    <div class="row">
      <label for="pulse-input">Pulse:</label>
      <input id="pulse-input" type="number" v-model.number="pulseValue" :disabled="!isConnected" />
    </div>

    <div class="row">
      <label for="frequency-select">Frequency (KHz):</label>
      <select id="frequency-select" v-model="frequencyValue" :disabled="!isConnected">
        <option v-for="freq in frequencies" :key="freq" :value="freq">{{ freq }}</option>
      </select>
      <button @click="setFrequency" :disabled="!isConnected">Set Frequency</button>
    </div>

    <div class="row">
      <button class="half-width" @click="sendS9" :disabled="!isConnected">s9</button>
      <button class="half-width" @click="sendS10" :disabled="!isConnected">s10</button>
    </div>

    <div class="messages" ref="messagesContainer">
      <h2>Messages:</h2>
      <div v-for="(msg, index) in receivedMessages" :key="index" class="message-item">
        {{ msg }}
      </div>
    </div>

    <footer class="footer">
      <div class="author">
        <span class="powered-by">POWERED BY</span>
        <a href="knoxim://sss.han" class="author-link" target="_blank">
          <svg class="user-icon" width="20" height="20" viewBox="0 0 24 24">
            <path fill="currentColor" d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm0 3c1.66 0 3 1.34 3 3s-1.34 3-3 3-3-1.34-3-3 1.34-3 3-3zm0 14.2c-2.5 0-4.71-1.28-6-3.22.03-1.99 4-3.08 6-3.08 1.99 0 5.97 1.09 6 3.08-1.29 1.94-3.5 3.22-6 3.22z"/>
          </svg>
          <span class="author-name">sss.han</span>
        </a>
      </div>
    </footer>
  </main>
</template>

<style scoped>
.container {
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding: 10px;
  max-width: 600px;
  margin: auto;
}

.row {
  display: flex;
  align-items: center;
  gap: 10px;
}

.row label {
  min-width: 80px;
  text-align: right;
}

.row input[type="number"],
.row select {
  flex-grow: 1;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.row button {
  padding: 8px 15px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.row button:hover:not(:disabled) {
  background-color: #0056b3;
}

.row button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
  opacity: 0.7;
}

.icon-button {
  padding: 8px 10px !important;
  min-width: 36px;
  font-size: 16px;
}

.row input:disabled,
.row select:disabled {
  background-color: #f5f5f5;
  cursor: not-allowed;
  opacity: 0.7;
}

.half-width {
  flex: 1 1 50%;
  width: 50%;
  margin: 0;
}

.messages {
  margin-top: 20px;
  border: 1px solid #eee;
  padding: 10px;
  max-height: 200px;
  overflow-y: auto;
  background-color: #f9f9f9;
  border-radius: 4px;
}

.message-item {
  font-family: monospace;
  white-space: pre-wrap;
  word-break: break-all;
  border-bottom: 1px dashed #eee;
  padding: 5px 0;
}

.message-item:last-child {
  border-bottom: none;
}

h1, h2 {
  text-align: center;
  color: #333;
}

.title-row {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
  margin-top: 0px
}

.version {
  font-size: 0.6em;
  color: #666;
  font-weight: normal;
  margin-right: 4px;
}

.footer {
  margin-top: 20px;
  padding-top: 20px;
  border-top: 1px solid #eee;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #666;
}

.github-link {
  color: #666;
  transition: all 0.3s ease;
  padding: 8px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.1);
  display: flex;
  align-items: center;
  justify-content: center;
}

.github-link:hover {
  color: #333;
  background: rgba(0, 0, 0, 0.05);
  transform: translateY(-2px);
}

.github-icon {
  display: block;
  filter: drop-shadow(0 2px 4px rgba(0, 0, 0, 0.1));
}

.author {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  font-size: 0.9em;
}

.user-icon {
  opacity: 0.8;
}

.powered-by {
  font-size: 0.7em;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #999;
  font-weight: 500;
}

.author-name {
  font-weight: 500;
  color: #666;
  margin-left: 4px;
}

.author-link {
  text-decoration: none;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  padding: 4px 8px;
  border-radius: 20px;
  color: #666;
}

.author-link:hover {
  color: #333;
  background: rgba(0, 0, 0, 0.05);
}
</style>