<template>
  <div class="w-screen h-screen flex flex-col bg-gray-100">
    <header class="text-center p-4 font-bold text-xl bg-white border-b">보드게임 룰 설명 봇</header>

    <div class="p-4 flex justify-center">
      <div class="w-full max-w-4xl">
        <label class="block text-gray-700 text-sm font-medium mb-2">게임을 선택해주세요.</label>
        <select v-model="selectedGame" class="w-full p-3 rounded border text-base">
          <option value="카탄">카탄</option>
          <option value="스플렌더">스플렌더</option>
        </select>
      </div>
    </div>

    <div class="flex-1 min-h-0 flex justify-center px-4">
      <div ref="chatBox" class="chat-scroll w-full max-w-4xl space-y-4 overflow-y-auto pr-2 pb-4">
        <div v-for="(msg, i) in messages" :key="i" class="flex" :class="msg.isBot ? 'justify-start' : 'justify-end'">
          <div class="max-w-[70%] px-5 py-2.5 rounded-[1.5rem] break-words markdown-body" :class="msg.isBot
            ? 'bg-purple-100 text-black text-left'
            : 'bg-purple-300 text-white text-left'" v-html="msg.text"></div>
        </div>
      </div>
    </div>

    <form @submit.prevent="sendMessage" class="p-4 border-t bg-white w-full flex justify-center">
      <div class="w-full max-w-4xl flex gap-2">
        <input v-model="userInput" class="flex-1 p-3 border rounded text-base" placeholder="질문을 입력해주세요." />
        <button type="submit" class="px-4 py-2 bg-purple-500 text-white rounded text-base">전송</button>
      </div>
    </form>
  </div>
</template>

<script setup>
import { ref, nextTick } from 'vue'
import { marked } from 'marked'
import axios from 'axios' // 💥 axios 꼭 설치돼 있어야 함!! 없으면 `npm install axios`


marked.use({
  breaks: true
})

const selectedGame = ref('카탄')
const userInput = ref('')
const messages = ref([])

const chatBox = ref(null)

const scrollToBottom = () => {
  nextTick(() => {
    if (chatBox.value) {
      chatBox.value.scrollTop = chatBox.value.scrollHeight
    }
  })
}

const sendMessage = async () => {
  if (!userInput.value.trim()) return

  // 유저 메시지 먼저 표시
  messages.value.push({
    text: escapeHtml(userInput.value).replace(/\n/g, '<br>'),
    isBot: false
  })
  scrollToBottom()

  const questionText = userInput.value
  userInput.value = ''

  try {
    const response = await axios.post('http://localhost:8080/api/ask', {
      question: questionText,
      game: selectedGame.value
    })

    const parsed = marked.parse(response.data.answer || '답변을 불러올 수 없습니다.')
    messages.value.push({ text: parsed, isBot: true })
  } catch (error) {
    console.error('❌ API 요청 실패:', error)
    messages.value.push({
      text: marked.parse('❌ 서버와의 통신 중 오류가 발생했어요. 나중에 다시 시도해주세요!'),
      isBot: true
    })
  }

  scrollToBottom()
}


const escapeHtml = (text) =>
  text.replace(/[&<>"']/g, (match) => {
    const escape = {
      '&': '&amp;',
      '<': '&lt;',
      '>': '&gt;',
      '"': '&quot;',
      "'": '&#39;'
    }
    return escape[match]
  })
</script>

<style>
.chat-scroll {
  scrollbar-width: thin;
  scrollbar-color: rgba(216, 180, 254, 0.3) transparent;
  scrollbar-gutter: stable both-edges;
}

.chat-scroll::-webkit-scrollbar {
  width: 8px;
  transition: opacity 0.3s ease;
  visibility: hidden;
}

.chat-scroll:hover::-webkit-scrollbar {
  visibility: visible;
  opacity: 1;
}

.chat-scroll::-webkit-scrollbar-thumb {
  background-color: #d8b4fe;
  border-radius: 10px;
}

.chat-scroll::-webkit-scrollbar-track {
  background-color: transparent;
}

.markdown-body {
  line-height: 1.5;
  text-align: left;
}

.markdown-body p {
  margin: 0 !important;
  /* 💥 여백 제거 핵심 */
  padding: 0;
  line-height: 1.5;
  white-space: pre-wrap;
}

.markdown-body strong {
  font-weight: bold;
}
</style>
