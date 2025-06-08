<template>
  <div class="container">
    <div class="editor-panel">
      <input v-model="fileName" placeholder="Filename..." class="filename-input" />
      <textarea v-model="code" placeholder="Start typing your Go code here..." class="code-editor"></textarea>
      <div class="button-row">
        <button @click="runCode" :disabled="!isWsConnected">Run Code (Go Agent)</button>
        <button @click="saveFile">Save File (PocketBase)</button>
        <button @click="loadFile">Load File (PocketBase)</button>
        <button @click="clearOutput">Clear Console</button>
      </div>
    </div>
    <div class="output-panel">
      <h3>Agent Output Console</h3>
      <div class="output-console">
        <div v-for="(line, idx) in agentOutput" :key="idx">{{ line }}</div>
      </div>
    </div>
    <div v-if="messageBox.show" class="modal">
      <div class="modal-content">
        <p>{{ messageBox.text }}</p>
        <button @click="messageBox.show = false">OK</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted } from 'vue'
import PocketBase from 'pocketbase'

const pb = new PocketBase(import.meta.env.VITE_PB_URL || 'http://localhost:8090')
const code = ref('')
const fileName = ref('my_agent_code.go')
const agentOutput = ref([])
const ws = ref(null)
const isWsConnected = ref(false)
const messageBox = reactive({ show: false, text: '' })

const showMessage = (text) => {
  messageBox.text = text
  messageBox.show = true
}

const connectWebSocket = () => {
  const wsUrl = import.meta.env.VITE_WS_URL || 'ws://localhost:8090/ws'
  ws.value = new WebSocket(wsUrl)
  ws.value.onopen = () => {
    isWsConnected.value = true
    showMessage('Connected to Go Agent!')
  }
  ws.value.onmessage = (event) => {
    agentOutput.value.push(event.data)
  }
  ws.value.onclose = () => {
    isWsConnected.value = false
    showMessage('Disconnected from Go Agent. Please ensure the agent is running.')
    setTimeout(connectWebSocket, 5000)
  }
  ws.value.onerror = () => {
    isWsConnected.value = false
    showMessage('WebSocket error. Is the Go agent running?')
  }
}

const runCode = () => {
  if (ws.value && ws.value.readyState === WebSocket.OPEN && code.value) {
    ws.value.send(code.value)
    agentOutput.value.push(`> Sending code to agent: ${fileName.value}`)
  } else {
    showMessage('WebSocket is not connected or code is empty.')
  }
}

const clearOutput = () => {
  agentOutput.value = []
}

const saveFile = async () => {
  // Prompt for login if not authenticated
  if (!pb.authStore.isValid) {
    // You can show a login modal here
    showMessage('Please log in to save files.')

  }
  // Save or update file in PocketBase
  // ... (same as your previous logic)
}

const loadFile = async () => {
  // Prompt for login if not authenticated
  if (!pb.authStore.isValid) {
    showMessage('Please log in to load files.')

  }
  // Load file from PocketBase
  // ... (same as your previous logic)
}

onMounted(() => {
  connectWebSocket()
})
</script>

<style scoped>
/* Add styles to match your screenshot */
</style>
