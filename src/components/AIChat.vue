<template>
  <div class="chat-layout">
    <!-- 侧边栏 -->
    <div :class="['sidebar', { collapsed: isSidebarCollapsed }]">
      <div class="sidebar-header">
        <el-button
          type="primary"
          class="new-chat-btn"
          @click="handleNewChat"
        >
          <el-icon><Plus /></el-icon>
          <span v-if="!isSidebarCollapsed">新对话</span>
        </el-button>
        <el-button
          text
          class="collapse-btn"
          @click="toggleSidebar"
        >
          <el-icon><component :is="isSidebarCollapsed ? Expand : Fold" /></el-icon>
        </el-button>
      </div>

      <div class="conversation-list">
        <div
          v-for="conv in conversations"
          :key="conv.id"
          :class="['conversation-item', { active: currentConversationId === conv.id }]"
          @click="switchConversation(conv.id)"
        >
          <el-icon class="conv-icon"><ChatDotRound /></el-icon>
          <div v-if="!isSidebarCollapsed" class="conv-content">
            <div class="conv-title">{{ conv.title }}</div>
            <div class="conv-time">{{ formatConversationTime(conv.updateTime) }}</div>
          </div>
          <el-icon
            v-if="!isSidebarCollapsed"
            class="delete-icon"
            @click.stop="deleteConversation(conv.id)"
          >
            <Close />
          </el-icon>
        </div>

        <div v-if="conversations.length === 0" class="empty-conversations">
          <el-icon :size="40"><ChatLineRound /></el-icon>
          <p v-if="!isSidebarCollapsed">暂无对话</p>
        </div>
      </div>
    </div>

    <!-- 主聊天区域 -->
    <div class="chat-main">
      <div class="chat-container">
        <div class="chat-header">
          <div class="header-content">
            <el-button
              text
              class="mobile-menu-btn"
              @click="toggleSidebar"
            >
              <el-icon><Fold /></el-icon>
            </el-button>
            <el-icon :size="28" color="#667eea"><ChatDotRound /></el-icon>
            <h2>AI维修 智能助手</h2>
          </div>
          <div class="header-actions">
            <el-button text @click="handleClearChat">
              <el-icon><Delete /></el-icon>
              清空对话
            </el-button>
          </div>
        </div>

        <div class="messages-container" ref="messagesContainer">
          <div v-if="messages.length === 0" class="welcome-screen">
            <div class="welcome-content">
              <div class="logo-animation">
                <el-icon :size="80" color="#667eea"><ChatDotRound /></el-icon>
              </div>
              <h3>你好！我是 维修AI 助手</h3>
              <p>我可以帮你解答查维修资料,维修过程新增领料单等功能</p>
              <div class="suggestion-chips">
                <div
                  v-for="suggestion in suggestions"
                  :key="suggestion"
                  class="suggestion-chip"
                  @click="handleSuggestionClick(suggestion)"
                >
                  {{ suggestion }}
                </div>
              </div>
            </div>
          </div>

          <div
              v-for="(message, index) in messages"
              :key="index"
              :class="['message-item', message.role]"
          >
            <div class="message-avatar">
              <el-icon v-if="message.role === 'user'" :size="24"><User /></el-icon>
              <el-icon v-else :size="24" color="#fff"><ChatDotRound /></el-icon>
            </div>
            <div class="message-wrapper">
              <div class="message-info">
                <span class="message-role">{{ message.role === 'user' ? '你' : 'AI 助手' }}</span>
                <span class="message-time">{{ formatTime(message.timestamp) }}</span>
              </div>
              <div class="message-bubble">
                <div v-if="message.content" class="message-text" v-html="renderMarkdown(message.content)"></div>
                <div v-else-if="isLoading && index === messages.length - 1 && message.role === 'ai'" class="typing-indicator">
                  <span></span>
                  <span></span>
                  <span></span>
                </div>
              </div>
            </div>
          </div>

          <div v-if="false" class="message-item ai">
            <div class="message-avatar">
              <el-icon :size="24" color="#fff"><ChatDotRound /></el-icon>
            </div>
            <div class="message-wrapper">
              <div class="message-info">
                <span class="message-role">AI 助手</span>
              </div>
              <div class="message-bubble">
                <div class="typing-indicator">
                  <span></span>
                  <span></span>
                  <span></span>
                </div>
              </div>
            </div>
          </div>
        </div>

        <div class="input-area">
          <div class="input-wrapper">
            <el-input
              v-model="inputMessage"
              type="textarea"
              :rows="3"
              placeholder="输入消息... (Shift + Enter 换行，Enter 发送)"
              @keydown.enter.exact.prevent="handleSendMessage"
              :disabled="isLoading"
              resize="none"
              class="message-input"
            />
            <div class="input-actions">
              <span class="input-tip">按 Enter 发送</span>
              <el-button
                type="primary"
                @click="handleSendMessage"
                :loading="isLoading"
                :disabled="!inputMessage.trim()"
                class="send-button"
              >
                <el-icon><Promotion /></el-icon>
                发送
              </el-button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick, onMounted, watch } from 'vue'
import { ChatDotRound, User, Promotion, Delete, Plus, Close, Fold, Expand, ChatLineRound } from '@element-plus/icons-vue'
import MarkdownIt from 'markdown-it'
import hljs from 'highlight.js'
import axios from 'axios'
import 'highlight.js/styles/atom-one-dark.css'

const messages = ref([])
const inputMessage = ref('')
const isLoading = ref(false)
const messagesContainer = ref(null)
const memoryId = ref(null)
const isSidebarCollapsed = ref(false)

const conversations = ref([])
const currentConversationId = ref(null)

const md = new MarkdownIt({
  html: true,
  linkify: true,
  typographer: true,
  highlight: function (str, lang) {
    if (lang && hljs.getLanguage(lang)) {
      try {
        return '<pre class="hljs"><code>' +
               hljs.highlight(str, { language: lang, ignoreIllegals: true }).value +
               '</code></pre>'
      } catch (__) {}
    }
    return '<pre class="hljs"><code>' + md.utils.escapeHtml(str) + '</code></pre>'
  }
})

const suggestions = [
  '你能做什么',
  '帮我查资料',
  '我需要领XXX料',
]

const renderMarkdown = (content) => {
  return md.render(content)
}

const formatTime = (timestamp) => {
  const date = new Date(timestamp)
  return date.toLocaleTimeString('zh-CN', { hour: '2-digit', minute: '2-digit' })
}

const formatConversationTime = (timestamp) => {
  const date = new Date(timestamp)
  const now = new Date()
  const diff = now - date

  if (diff < 60000) return '刚刚'
  if (diff < 3600000) return `${Math.floor(diff / 60000)}分钟前`
  if (diff < 86400000) return `${Math.floor(diff / 3600000)}小时前`

  return date.toLocaleDateString('zh-CN', { month: '2-digit', day: '2-digit' })
}

const scrollToBottom = async () => {
  await nextTick()
  if (messagesContainer.value) {
    messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight
  }
}

const generateMemoryId = () => {
  return Math.floor(Math.random() * 1000000)
}

const createNewConversation = () => {
  const newId = generateMemoryId()
  const newConv = {
    id: newId,
    title: '新对话',
    messages: [],
    createTime: Date.now(),
    updateTime: Date.now()
  }

  conversations.value.unshift(newConv)
  currentConversationId.value = newId
  memoryId.value = newId
  messages.value = []

  saveConversationsToStorage()

  return newId
}

const switchConversation = (convId) => {
  if (currentConversationId.value === convId) return

  const conv = conversations.value.find(c => c.id === convId)
  if (!conv) return

  currentConversationId.value = convId
  memoryId.value = convId
  messages.value = [...conv.messages]

  scrollToBottom()
}

const deleteConversation = (convId) => {
  const index = conversations.value.findIndex(c => c.id === convId)
  if (index === -1) return

  conversations.value.splice(index, 1)

  if (currentConversationId.value === convId) {
    if (conversations.value.length > 0) {
      switchConversation(conversations.value[0].id)
    } else {
      createNewConversation()
    }
  }

  saveConversationsToStorage()
}

const updateCurrentConversation = () => {
  const conv = conversations.value.find(c => c.id === currentConversationId.value)
  if (!conv) return

  conv.messages = [...messages.value]
  conv.updateTime = Date.now()

  if (messages.value.length > 0) {
    const firstUserMsg = messages.value.find(m => m.role === 'user')
    if (firstUserMsg) {
      conv.title = firstUserMsg.content.substring(0, 20) + (firstUserMsg.content.length > 20 ? '...' : '')
    }
  }

  saveConversationsToStorage()
}

const saveConversationsToStorage = () => {
  try {
    localStorage.setItem('ai-conversations', JSON.stringify(conversations.value))
  } catch (e) {
    console.error('Failed to save conversations:', e)
  }
}

const loadConversationsFromStorage = () => {
  try {
    const saved = localStorage.getItem('ai-conversations')
    if (saved) {
      conversations.value = JSON.parse(saved)

      if (conversations.value.length > 0) {
        switchConversation(conversations.value[0].id)
      } else {
        createNewConversation()
      }
    } else {
      createNewConversation()
    }
  } catch (e) {
    console.error('Failed to load conversations:', e)
    createNewConversation()
  }
}
// ... existing code ...
const handleSendMessage = async () => {
  if (!inputMessage.value.trim() || isLoading.value) return

  if (!memoryId.value) {
    memoryId.value = generateMemoryId()
  }

  const userMessage = {
    role: 'user',
    content: inputMessage.value.trim(),
    timestamp: Date.now()
  }

  messages.value.push(userMessage)
  updateCurrentConversation()

  const currentInput = inputMessage.value
  inputMessage.value = ''

  await scrollToBottom()
  isLoading.value = true

  const aiMessage = {
    role: 'ai',
    content: '',
    timestamp: Date.now()
  }
  messages.value.push(aiMessage)

  const aiMessageIndex = messages.value.length - 1
  console.log('AI消息索引:', aiMessageIndex, '当前消息数量:', messages.value.length)
  console.log('AI消息对象:', messages.value[aiMessageIndex])

  try {
    console.log('开始请求流式接口...')
    const response = await fetch('http://localhost:8080/AI-Qwen/api/chat/send/stream', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        memoryId: memoryId.value,
        message: currentInput
      })
    })

    console.log('响应状态:', response.status)
    console.log('响应头 Content-Type:', response.headers.get('content-type'))
    console.log('响应体是否存在:', response.body !== null)

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }

    const reader = response.body.getReader()
    const decoder = new TextDecoder('utf-8')
    let buffer = ''
    let isDone = false
    let currentEvent = ''
    let currentData = ''
    let chunkCount = 0
    let totalChars = 0

    console.log('开始读取流式数据...')

    while (!isDone) {
      console.log('等待读取数据块...')
      const { done, value } = await reader.read()

      console.log('读取结果 - done:', done, 'value长度:', value?.length || 0)

      if (done) {
        console.log('✅ 读取完成，总共接收', chunkCount, '个数据块')
        break
      }

      chunkCount++
      const text = decoder.decode(value, { stream: true })
      console.log(`📦 第${chunkCount}个数据块原始内容:`, JSON.stringify(text))
      console.log(`📦 第${chunkCount}个数据块解码后:`, text)

      buffer += text

      const lines = buffer.split('\n')
      buffer = lines.pop() || ''

      console.log('解析到', lines.length, '行，剩余buffer:', JSON.stringify(buffer))

      for (const line of lines) {
        const trimmedLine = line.trim()
        console.log('处理行:', JSON.stringify(trimmedLine))

        if (trimmedLine.startsWith('event:')) {
          currentEvent = trimmedLine.substring(6).trim()
          console.log('🏷️ 事件类型:', currentEvent)
        } else if (trimmedLine.startsWith('data:')) {
          currentData = trimmedLine.substring(5).trim()
          console.log('📝 数据内容:', currentData, '事件:', currentEvent)

          if (currentEvent === 'message') {
            if (messages.value[aiMessageIndex]) {
              messages.value[aiMessageIndex].content += currentData
              totalChars += currentData.length

              if (totalChars <= 10 || totalChars % 10 === 0) {
                console.log(`✨ 已接收${totalChars}个字符，内容:`, messages.value[aiMessageIndex].content)
              }

              await scrollToBottom()
            } else {
              console.error('❌ AI消息对象不存在！索引:', aiMessageIndex, '数组长度:', messages.value.length)
            }
          } else if (currentEvent === 'done') {
            console.log('✅ 收到完成信号，总字符数:', totalChars)
            isDone = true
            break
          } else if (currentEvent === 'error') {
            console.error('❌ 收到错误:', currentData)
            if (messages.value[aiMessageIndex]) {
              messages.value[aiMessageIndex].content = `错误: ${currentData}`
            }
            isDone = true
            break
          }

          currentEvent = ''
          currentData = ''
        } else if (trimmedLine === '') {
          currentEvent = ''
          currentData = ''
        }
      }
    }

    console.log('🏁 最终内容长度:', messages.value[aiMessageIndex]?.content?.length || 0)
    console.log('🏁 最终内容:', messages.value[aiMessageIndex]?.content)
    updateCurrentConversation()
    await scrollToBottom()
  } catch (error) {
    console.error('❌ Error:', error)
    let errorMessage = '抱歉，出现了一些问题。请稍后重试。'

    if (error.message.includes('Failed to fetch')) {
      errorMessage = '无法连接到服务器，请检查后端服务是否启动。\n\n确保后端服务运行在 http://localhost:8080'
    } else {
      errorMessage = `请求失败: ${error.message}`
    }

    if (messages.value[aiMessageIndex]) {
      messages.value[aiMessageIndex].content = errorMessage
    } else {
      messages.value.push({
        role: 'ai',
        content: errorMessage,
        timestamp: Date.now()
      })
    }

    updateCurrentConversation()
  } finally {
    isLoading.value = false
  }
}



// ... existing code ...


const handleSuggestionClick = (suggestion) => {
  inputMessage.value = suggestion
  handleSendMessage()
}

const handleClearChat = () => {
  messages.value = []
  updateCurrentConversation()
}

const handleNewChat = () => {
  createNewConversation()
}

const toggleSidebar = () => {
  isSidebarCollapsed.value = !isSidebarCollapsed.value
}

onMounted(() => {
  loadConversationsFromStorage()
  scrollToBottom()
})

watch(messages, () => {
  updateCurrentConversation()
}, { deep: true })
</script>

<style scoped>
.chat-layout {
  display: flex;
  height: 100vh;
  background: #f5f5f5;
}

.sidebar {
  width: 260px;
  background: linear-gradient(180deg, #667eea 0%, #764ba2 100%);
  display: flex;
  flex-direction: column;
  transition: all 0.3s ease;
  overflow: hidden;
  box-shadow: 2px 0 8px rgba(0, 0, 0, 0.1);
}

.sidebar.collapsed {
  width: 60px;
}

.sidebar-header {
  padding: 16px;
  display: flex;
  gap: 8px;
  align-items: center;
}

.new-chat-btn {
  flex: 1;
  background: rgba(255, 255, 255, 0.2);
  border: 1px solid rgba(255, 255, 255, 0.3);
  color: white;
  backdrop-filter: blur(10px);
  transition: all 0.3s;
}

.new-chat-btn:hover {
  background: rgba(255, 255, 255, 0.3);
  transform: translateY(-2px);
}

.collapse-btn {
  color: white;
  padding: 8px;
}

.conversation-list {
  flex: 1;
  overflow-y: auto;
  padding: 8px;
}

.conversation-list::-webkit-scrollbar {
  width: 4px;
}

.conversation-list::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.3);
  border-radius: 2px;
}

.conversation-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px;
  margin-bottom: 4px;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
  color: white;
  position: relative;
}

.conversation-item:hover {
  background: rgba(255, 255, 255, 0.15);
}

.conversation-item.active {
  background: rgba(255, 255, 255, 0.25);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.conv-icon {
  font-size: 18px;
  flex-shrink: 0;
}

.conv-content {
  flex: 1;
  min-width: 0;
}

.conv-title {
  font-size: 14px;
  font-weight: 500;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  margin-bottom: 4px;
}

.conv-time {
  font-size: 11px;
  opacity: 0.7;
}

.delete-icon {
  font-size: 16px;
  opacity: 0;
  transition: opacity 0.2s;
  padding: 4px;
}

.conversation-item:hover .delete-icon {
  opacity: 0.8;
}

.delete-icon:hover {
  opacity: 1 !important;
  color: #ff6b6b;
}

.empty-conversations {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 40px 20px;
  color: rgba(255, 255, 255, 0.6);
  text-align: center;
}

.empty-conversations p {
  margin-top: 12px;
  font-size: 14px;
}

.chat-main {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.chat-container {
  display: flex;
  flex-direction: column;
  height: 100%;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

.chat-header {
  background: rgba(255, 255, 255, 0.98);
  backdrop-filter: blur(20px);
  padding: 16px 24px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.08);
  display: flex;
  justify-content: space-between;
  align-items: center;
  z-index: 10;
}

.header-content {
  display: flex;
  align-items: center;
  gap: 12px;
}

.mobile-menu-btn {
  display: none;
}

.header-content h2 {
  margin: 0;
  font-size: 20px;
  font-weight: 600;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.header-actions {
  display: flex;
  gap: 8px;
}

.messages-container {
  flex: 1;
  overflow-y: auto;
  padding: 24px;
  display: flex;
  flex-direction: column;
  gap: 20px;
  scroll-behavior: smooth;
}

.messages-container::-webkit-scrollbar {
  width: 8px;
}

.messages-container::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.1);
}

.messages-container::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.3);
  border-radius: 4px;
}

.messages-container::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.5);
}

.welcome-screen {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  animation: fadeIn 0.6s ease-out;
}

.welcome-content {
  text-align: center;
  color: white;
  max-width: 600px;
  padding: 40px;
}

.logo-animation {
  animation: pulse 2s ease-in-out infinite;
}

.welcome-content h3 {
  font-size: 32px;
  margin: 24px 0 12px;
  font-weight: 600;
}

.welcome-content p {
  font-size: 16px;
  opacity: 0.95;
  margin-bottom: 32px;
  line-height: 1.6;
}

.suggestion-chips {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  justify-content: center;
}

.suggestion-chip {
  cursor: pointer;
  padding: 10px 18px;
  font-size: 14px;
  background: rgba(255, 255, 255, 0.15);
  border: 1px solid rgba(255, 255, 255, 0.3);
  color: white;
  border-radius: 20px;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  backdrop-filter: blur(10px);
}

.suggestion-chip:hover {
  background: rgba(255, 255, 255, 0.25);
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.message-item {
  display: flex;
  gap: 12px;
  max-width: 85%;
  animation: slideIn 0.3s ease-out;
}

.message-item.user {
  align-self: flex-end;
  flex-direction: row-reverse;
}

.message-item.ai {
  align-self: flex-start;
}

.message-avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
}

.message-item.user .message-avatar {
  background: white;
  color: #667eea;
}

.message-item.ai .message-avatar {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

.message-wrapper {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.message-info {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 12px;
  color: rgba(255, 255, 255, 0.9);
}

.message-item.user .message-info {
  justify-content: flex-end;
}

.message-role {
  font-weight: 600;
}

.message-time {
  opacity: 0.7;
}

.message-bubble {
  background: white;
  padding: 16px 20px;
  border-radius: 16px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
  max-width: 100%;
}

.message-item.user .message-bubble {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

.message-text {
  font-size: 14px;
  line-height: 1.8;
  word-wrap: break-word;
}

.message-text :deep(h1),
.message-text :deep(h2),
.message-text :deep(h3),
.message-text :deep(h4) {
  margin: 20px 0 12px;
  font-weight: 600;
  line-height: 1.4;
}

.message-text :deep(h1) {
  font-size: 24px;
}

.message-text :deep(h2) {
  font-size: 20px;
}

.message-text :deep(h3) {
  font-size: 17px;
}

.message-text :deep(p) {
  margin: 10px 0;
}

.message-text :deep(code) {
  background: rgba(0, 0, 0, 0.06);
  padding: 2px 6px;
  border-radius: 4px;
  font-family: 'Consolas', 'Monaco', 'Courier New', monospace;
  font-size: 13px;
}

.message-item.user .message-text :deep(code) {
  background: rgba(255, 255, 255, 0.2);
}

.message-text :deep(pre) {
  background: #282c34;
  padding: 16px;
  border-radius: 8px;
  overflow-x: auto;
  margin: 16px 0;
  position: relative;
}

.message-text :deep(pre code) {
  background: none;
  padding: 0;
  color: #abb2bf;
  font-size: 13px;
  line-height: 1.6;
}

.message-text :deep(ul),
.message-text :deep(ol) {
  margin: 12px 0;
  padding-left: 24px;
}

.message-text :deep(li) {
  margin: 6px 0;
}

.message-text :deep(table) {
  border-collapse: collapse;
  width: 100%;
  margin: 16px 0;
  font-size: 13px;
}

.message-text :deep(th),
.message-text :deep(td) {
  border: 1px solid #e0e0e0;
  padding: 10px 14px;
  text-align: left;
}

.message-text :deep(th) {
  background: #f5f5f5;
  font-weight: 600;
}

.message-text :deep(blockquote) {
  border-left: 4px solid #667eea;
  padding-left: 16px;
  margin: 16px 0;
  color: #666;
  font-style: italic;
}

.message-text :deep(a) {
  color: #667eea;
  text-decoration: none;
}

.message-text :deep(a):hover {
  text-decoration: underline;
}

.typing-indicator {
  display: flex;
  gap: 6px;
  padding: 8px 0;
}

.typing-indicator span {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #667eea;
  animation: typing 1.4s infinite;
}

.typing-indicator span:nth-child(2) {
  animation-delay: 0.2s;
}

.typing-indicator span:nth-child(3) {
  animation-delay: 0.4s;
}

.input-area {
  background: rgba(255, 255, 255, 0.98);
  backdrop-filter: blur(20px);
  padding: 20px 24px;
  box-shadow: 0 -2px 12px rgba(0, 0, 0, 0.08);
}

.input-wrapper {
  max-width: 1200px;
  margin: 0 auto;
}

.message-input :deep(.el-textarea__inner) {
  border-radius: 12px;
  border: 2px solid #e8e8e8;
  transition: all 0.3s;
  font-size: 14px;
  line-height: 1.6;
  padding: 12px 16px;
  resize: none;
}

.message-input :deep(.el-textarea__inner):focus {
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.message-input :deep(.el-textarea__inner):disabled {
  background: #f5f5f5;
  cursor: not-allowed;
}

.input-actions {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 12px;
}

.input-tip {
  font-size: 12px;
  color: #999;
}

.send-button {
  border-radius: 10px;
  padding: 10px 28px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border: none;
  font-weight: 500;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.send-button:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(102, 126, 234, 0.4);
}

.send-button:active:not(:disabled) {
  transform: translateY(0);
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes pulse {
  0%, 100% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.05);
  }
}

@keyframes typing {
  0%, 60%, 100% {
    transform: translateY(0);
    opacity: 0.7;
  }
  30% {
    transform: translateY(-10px);
    opacity: 1;
  }
}

@media (max-width: 768px) {
  .sidebar {
    position: fixed;
    left: 0;
    top: 0;
    bottom: 0;
    z-index: 1000;
    transform: translateX(-100%);
  }

  .sidebar:not(.collapsed) {
    transform: translateX(0);
  }

  .mobile-menu-btn {
    display: block;
  }

  .chat-header {
    padding: 12px 16px;
  }

  .header-content h2 {
    font-size: 18px;
  }

  .messages-container {
    padding: 16px;
  }

  .message-item {
    max-width: 95%;
  }

  .welcome-content {
    padding: 20px;
  }

  .welcome-content h3 {
    font-size: 24px;
  }

  .suggestion-chips {
    padding: 0 10px;
  }

  .input-area {
    padding: 16px;
  }
}
</style>
