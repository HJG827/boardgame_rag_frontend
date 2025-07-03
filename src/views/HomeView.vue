<template>
  <div class="w-screen h-screen flex flex-col bg-gray-100">
    <header class="text-center p-4 font-bold text-xl bg-white border-b">보드게임 룰 설명 봇</header>

    <!-- 🧠 자동완성 게임 선택 -->
    <div class="p-4 flex justify-center">
      <div class="w-full max-w-4xl">
        <label class="block text-gray-700 text-sm font-medium mb-2">게임을 입력하거나 선택해주세요.</label>
        <div class="relative">
          <input
            ref="inputEl"
            v-model="customGame"
            @input="filterGameOptions"
            @keydown.down.prevent="highlightNext"
            @keydown.up.prevent="highlightPrev"
            @keydown.enter.prevent="selectHighlightedOption"
            @focus="showSuggestions = true; filterGameOptions()"
            @blur="hideSuggestions"
            class="w-full py-3 px-3 rounded border text-base"
            placeholder="게임명을 입력하세요 (예: 할리갈리)"
          />
          <!-- 🔽 자동완성 목록 -->
          <ul
  v-if="showSuggestions && filteredOptions.length"
  class="absolute z-10 bg-white border w-full mt-1 rounded shadow-md auto-suggest-scroll"
>

            <li
              v-for="(option, index) in filteredOptions"
              :key="option.eng"
              @mousedown.prevent="selectSuggestion(option)"
              :class="[
                'px-4 py-2 hover:bg-gray-100 cursor-pointer',
                index === highlightedIndex ? 'bg-gray-200' : ''
              ]"
            >
              {{ option.kor }}
            </li>
          </ul>
          <!-- ⚠️ 안내 문구 -->
          <p v-if="customGame && isUnknownGame" class="text-sm text-gray-500 mt-1">
            ※ 보유하고 있지 않은 게임을 입력해서 정확도가 떨어질 수 있습니다.
          </p>
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
        <button
          type="submit"
          class="px-4 py-2 rounded text-base transition duration-200"
          :class="isLoading
            ? 'bg-purple-300 text-white cursor-not-allowed'
            : 'bg-purple-500 text-white hover:bg-purple-600'"
          :disabled="isLoading">
          전송
        </button>
      </div>
    </form>
  </div>
</template>

<script setup>
import { ref, nextTick, watch, computed } from 'vue'
import { marked } from 'marked'
import axios from 'axios'
import Hangul from 'hangul-js'

marked.use({ breaks: true })

const customGame = ref('')
const userInput = ref('')
const messages = ref([])
const allMessages = ref({})
const chatBox = ref(null)
const inputEl = ref(null)
const loadingBotIndex = ref(null)
const isLoading = ref(false)
const highlightedIndex = ref(-1)

const isUnknownGame = computed(() => {
  return customGame.value && !gameOptions.some(opt => opt.kor === customGame.value)
})

const gameOptions = [
  { kor: "낫쏘", eng: "Nassau" },
  { kor: "더 마인드", eng: "TheMind" },
  { kor: "달무티", eng: "Dalmuti" },
  { kor: "라비린스", eng: "Labyrinth" },
  { kor: "루미큐브", eng: "Rummikub" },
  { kor: "메이플 벨리", eng: "MapleValley" },
  { kor: "블로커스", eng: "Blokus" },
  { kor: "스플렌더", eng: "Splendor" },
  { kor: "카탄", eng: "Katan" },
  { kor: "퀀텀", eng: "Qwantum" },
  { kor: "할리갈리", eng: "HalliGalli" }
]

const filteredOptions = ref([])
const showSuggestions = ref(false)

const matchKorean = (input, target) => Hangul.search(target, input) > -1

const filterGameOptions = () => {
  const keyword = customGame.value.trim()
  if (!keyword) {
    filteredOptions.value = [...gameOptions] // 처음에 전체 보여주기
    highlightedIndex.value = -1
    return
  }
  filteredOptions.value = gameOptions.filter(game => matchKorean(keyword, game.kor))
  highlightedIndex.value = -1
}

const highlightNext = () => {
  if (!filteredOptions.value.length) return
  highlightedIndex.value = (highlightedIndex.value + 1) % filteredOptions.value.length
}

const highlightPrev = () => {
  if (!filteredOptions.value.length) return
  highlightedIndex.value = (highlightedIndex.value - 1 + filteredOptions.value.length) % filteredOptions.value.length
}

const selectHighlightedOption = () => {
  if (highlightedIndex.value >= 0) {
    selectSuggestion(filteredOptions.value[highlightedIndex.value])
  }
}

const hideSuggestions = () => {
  setTimeout(() => { showSuggestions.value = false }, 200)
}

const selectSuggestion = (option) => {
  customGame.value = option.kor
  showSuggestions.value = false
}

const getCurrentGameKey = () => customGame.value.trim()
const getGameKorName = (eng) => customGame.value

watch(customGame, () => {
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
  if (!customGame.value.trim()) {
    messages.value.push({
      text: marked.parse('⚠️ 먼저 게임을 선택해주세요!'),
      isBot: true,
      game: null
    })
    scrollToBottom()
    return
  }

  if (!userInput.value.trim() || isLoading.value) return

  isLoading.value = true

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

  const loadingMsg = {
    text: `<div class="typing-indicator"><span></span><span></span><span></span></div>`,
    isBot: true,
    game: gameKey
  }
  messages.value.push(loadingMsg)
  loadingBotIndex.value = messages.value.length - 1
  scrollToBottom()

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
    messages.value.splice(loadingBotIndex.value, 1, {
      text: parsed,
      isBot: true,
      game: gameKey
    })
    loadingBotIndex.value = null
    scrollToBottom()
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
  } finally {
    isLoading.value = false
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


.typing-indicator {
  display: flex;
  gap: 4px;
  align-items: center;
  height: 20px;
  padding: 0 6px;
}

.typing-indicator span {
  display: inline-block;
  width: 8px;
  height: 8px;
  background-color: #7c3aed; /* 보라색 점 */
  border-radius: 50%;
  animation: typingBounce 1.2s infinite ease-in-out;
}

.typing-indicator span:nth-child(1) {
  animation-delay: 0s;
}
.typing-indicator span:nth-child(2) {
  animation-delay: 0.2s;
}
.typing-indicator span:nth-child(3) {
  animation-delay: 0.4s;
}

@keyframes typingBounce {
  0%, 80%, 100% {
    transform: translateY(0);
    opacity: 0.4;
  }
  40% {
    transform: translateY(-6px);
    opacity: 1;
  }
}

.auto-suggest-scroll {
  max-height: calc(2.5rem * 6); /* 각 항목 높이 약 2.5rem × 6개 */
  overflow-y: auto;
  scrollbar-width: thin;
  scrollbar-color: rgba(216, 180, 254, 0.3) transparent;
}

.auto-suggest-scroll::-webkit-scrollbar {
  width: 8px;
  visibility: hidden;
}

.auto-suggest-scroll:hover::-webkit-scrollbar {
  visibility: visible;
}

.auto-suggest-scroll::-webkit-scrollbar-thumb {
  background-color: #d8b4fe;
  border-radius: 10px;
}

.auto-suggest-scroll::-webkit-scrollbar-track {
  background-color: transparent;
}

</style>
