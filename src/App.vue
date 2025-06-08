<template>
  <div class="container">
    <div v-if="isAuthModalOpen" class="modal">
      <div class="modal-content">
        <h2>{{ authMode === 'login' ? 'Login' : 'Register' }}</h2>
        <input v-model="authForm.email" placeholder="Email" />
        <input v-model="authForm.password" type="password" placeholder="Password" />
        <input v-if="authMode === 'register'" v-model="authForm.passwordConfirm" type="password" placeholder="Confirm Password" />
        <div v-if="authError" class="error">{{ authError }}</div>
        <button @click="authMode === 'login' ? login() : register()">
          {{ authMode === 'login' ? 'Login' : 'Register' }}
        </button>
        <a href="#" @click.prevent="authMode = authMode === 'login' ? 'register' : 'login'">
          {{ authMode === 'login' ? 'Need an account? Register' : 'Already have an account? Login' }}
        </a>
      </div>
    </div>
    <div class="qa-panel">
      <div class="conversation">
        <div v-for="(msg, idx) in conversation" :key="idx" :class="msg.role">
          <strong>{{ msg.role === 'user' ? 'You' : 'AI' }}:</strong> {{ msg.text }}
        </div>
      </div>
      <form @submit.prevent="askAI" class="input-row">
        <input v-model="question" placeholder="Ask me anything..." :disabled="loading" />
        <button type="submit" :disabled="loading || !question.trim()">Send</button>
      </form>
      <div v-if="loading" class="loading">Thinking...</div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive } from 'vue'
import PocketBase from 'pocketbase'

const pb = new PocketBase(import.meta.env.VITE_PB_URL || 'http://localhost:8090')
const isAuthModalOpen = ref(!pb.authStore.isValid)
const authMode = ref('login')
const authForm = reactive({ email: '', password: '', passwordConfirm: '' })
const authError = ref('')

const login = async () => {
  try {
    await pb.collection('users').authWithPassword(authForm.email, authForm.password)
    isAuthModalOpen.value = false
    authError.value = ''
  } catch (e) {
    authError.value = e.message
  }
}

const register = async () => {
  try {
    await pb.collection('users').create({
      email: authForm.email,
      password: authForm.password,
      passwordConfirm: authForm.passwordConfirm,
    })
    await login()
  } catch (e) {
    authError.value = e.message
  }
}

// Q&A logic
const question = ref('')
const conversation = ref([])
const loading = ref(false)

const askAI = async () => {
  if (!question.value.trim()) return
  const userMsg = { role: 'user', text: question.value }
  conversation.value.push(userMsg)
  loading.value = true
  try {
    const res = await fetch(`${import.meta.env.VITE_PB_URL || 'http://localhost:8090'}/api/ask/ai`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${pb.authStore.token}`,
      },
      body: JSON.stringify({ input: question.value }),
    })
    const data = await res.json()
    if (data.response) {
      conversation.value.push({ role: 'ai', text: data.response })
    } else {
      conversation.value.push({ role: 'ai', text: data.error || 'Error from AI' })
    }
  } catch (e) {
    conversation.value.push({ role: 'ai', text: 'Network error or server not available.' })
  }
  question.value = ''
  loading.value = false
}
</script>

<style scoped>
.container {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f7f9fb;
}
.qa-panel {
  background: #23242d;
  border-radius: 12px;
  padding: 2rem;
  min-width: 400px;
  max-width: 600px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.07);
  display: flex;
  flex-direction: column;
  gap: 1rem;
}
.conversation {
  background: #181920;
  color: #fff;
  border-radius: 8px;
  padding: 1rem;
  min-height: 200px;
  max-height: 300px;
  overflow-y: auto;
  font-family: 'Fira Mono', 'Consolas', monospace;
  margin-bottom: 1rem;
}
.user { color: #2ecc40; }
.ai { color: #39cccc; }
.input-row {
  display: flex;
  gap: 1rem;
}
.input-row input {
  flex: 1;
  padding: 0.7rem;
  border-radius: 6px;
  border: none;
  font-size: 1rem;
  background: #fff;
}
.input-row button {
  background: #2ecc40;
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 0.7rem 1.5rem;
  font-size: 1rem;
  cursor: pointer;
}
.loading {
  color: #fff;
  text-align: center;
  margin-top: 1rem;
}
.modal {
  position: fixed;
  top: 0; left: 0; right: 0; bottom: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}
.modal-content {
  background: #23242d;
  color: #fff;
  padding: 2rem 2.5rem;
  border-radius: 10px;
  min-width: 300px;
  text-align: center;
}
.modal-content button {
  margin-top: 1rem;
  background: #2ecc40;
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 0.5rem 1.5rem;
  font-size: 1rem;
  cursor: pointer;
}
.error {
  color: #ff4136;
  margin: 0.5rem 0;
}
</style>
