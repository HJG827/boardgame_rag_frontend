<template>
  <div class="w-screen h-screen flex flex-col bg-gray-100">
    <header class="text-center p-4 font-bold text-xl bg-white border-b">보드게임 룰 설명 봇</header>

    <!-- 🧠 드롭다운 + 직접 입력 통합 -->
    <div class="p-4 flex justify-center">
      <div class="w-full max-w-4xl">
        <label class="block text-gray-700 text-sm font-medium mb-2">게임을 선택해주세요.</label>
        <select v-model="selectedGame" class="w-full p-3 rounded border text-base">
          <option disabled value="">게임 선택</option>
          <option v-for="game in gameOptions" :key="game.eng" :value="game.eng">
            {{ game.kor }}
          </option>
          <option value="custom">직접 입력</option>
        </select>

        <!-- ✏️ 직접 입력창은 선택 시에만 나타남 -->
        <div v-if="selectedGame === 'custom'" class="mt-2">
          <input v-model="customGame" class="w-full p-3 border rounded text-base" placeholder="게임명을 직접 입력하세요 (예: 할리갈리)" />
        </div>
      </div>
    </div>

    <div class="flex-1 min-h-0 flex justify-center px-4">
      <div ref="chatBox" class="chat-scroll w-full max-w-4xl space-y-4 overflow-y-auto pr-2 pb-4">
        <div v-for="(msg, i) in messages" :key="i" class="flex flex-col" :class="msg.isBot ? 'items-start' : 'items-end'">
          <!-- 🎲 게임명 표시 -->
          <div v-if="msg.isBot && msg.game" class="text-sm text-[#7c3aed] font-semibold mb-1">
            🎲 {{ getGameKorName(msg.game) }}
          </div>
          <!-- 💬 말풍선 -->
          <div class="max-w-[70%] px-5 py-2.5 rounded-[1.5rem] break-words markdown-body"
               :class="msg.isBot ? 'bg-purple-100 text-black text-left' : 'bg-purple-300 text-white text-left'"
               v-html="msg.text"></div>
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
import { ref, nextTick, watch } from 'vue'
import { marked } from 'marked'
import axios from 'axios'

marked.use({ breaks: true })

const selectedGame = ref('')
const customGame = ref('')
const userInput = ref('')
const messages = ref([])
const allMessages = ref({})
const chatBox = ref(null)

const gameOptions = [
  { kor: "카탄", eng: "katan" },
  { kor: "스플렌더", eng: "splendor" },
]

const getCurrentGameKey = () => selectedGame.value === 'custom' ? customGame.value.trim() : selectedGame.value

const getGameKorName = (eng) => {
  if (eng === 'custom') return customGame.value
  const found = gameOptions.find(g => g.eng === eng)
  return found ? found.kor : eng
}

watch([selectedGame, customGame], () => {
  const key = getCurrentGameKey()
  messages.value = allMessages.value[key] || []
})

const scrollToBottom = () => {
  nextTick(() => {
    if (chatBox.value) {
      chatBox.value.scrollTop = chatBox.value.scrollHeight
    }
  })
}

const sendMessage = async () => {
  if (!userInput.value.trim()) return

  const gameKey = getCurrentGameKey()
  const questionText = userInput.value

  const userMsg = {
    text: escapeHtml(userInput.value).replace(/\n/g, '<br>'),
    isBot: false,
    game: gameKey
  }
  messages.value.push(userMsg)
  userInput.value = ''
  scrollToBottom()

  // history 구성 (현재 질문 포함 X)
  const filtered = messages.value.filter(m => m.game === gameKey)
  const history = []
  for (let i = 0; i < filtered.length - 1; i++) {
    const user = filtered[i]
    const bot = filtered[i + 1]
    if (!user.isBot && bot?.isBot) {
      history.push({ role: 'user', content: user.text.replace(/<br>/g, '\n') })
      history.push({ role: 'assistant', content: bot.text.replace(/<br>/g, '\n') })
    }
  }

  try {

    const res = await axios.post('https://boardgame-client.fly.dev/api/ask', {
      question: questionText,
      game: gameKey,
      history: history
    })

    const parsed = marked.parse(res.data.answer || '답변을 불러올 수 없습니다.')
    const botMsg = {
      text: parsed,
      isBot: true,
      game: gameKey
    }
    messages.value.push(botMsg)
    scrollToBottom()

    // 메시지 백업
    allMessages.value[gameKey] = [...messages.value]

  } catch (err) {
    console.error('❌ 서버 오류:', err)
    const errorMsg = {
      text: marked.parse('❌ 서버와의 통신 오류입니다. 다시 시도해주세요!'),
      isBot: true,
      game: gameKey
    }
    messages.value.push(errorMsg)
    allMessages.value[gameKey] = [...messages.value]
  }
}

const escapeHtml = (text) =>
  text.replace(/[&<>"']/g, (match) => ({
    '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;'
  }[match]))
</script>

<style>
.chat-scroll {
  scrollbar-width: thin;
  scrollbar-color: rgba(216, 180, 254, 0.3) transparent;
  scrollbar-gutter: stable both-edges;
}
.chat-scroll::-webkit-scrollbar {
  width: 8px;
  visibility: hidden;
}
.chat-scroll:hover::-webkit-scrollbar {
  visibility: visible;
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
  padding: 0;
  line-height: 1.5;
  white-space: pre-wrap;
}
.markdown-body strong {
  font-weight: bold;
}
</style>
