<template>
  <div class="w-screen min-h-screen flex flex-col bg-gray-100 overflow-hidden">
    <!-- 헤더 -->
    <header class="text-center p-4 font-bold text-xl bg-white border-b w-full">보드게임 룰 설명 봇</header>

    <!-- 게임 선택 -->
    <div class="p-4 w-full flex justify-center">
      <div class="w-full max-w-4xl">
        <label class="block text-gray-700 text-sm font-medium mb-2">게임을 선택해주세요.</label>
        <select v-model="selectedGame" class="w-full p-3 rounded border text-base">
          <option value="카탄">카탄</option>
          <option value="스플렌더">스플렌더</option>
        </select>
      </div>
    </div>

    <!-- 채팅 내용 -->
    <div class="flex-1 overflow-y-auto w-full flex justify-center px-4 pb-6">
      <div class="w-full max-w-4xl space-y-4">
        <div v-for="(msg, i) in messages" :key="i" class="flex" :class="msg.isBot ? 'justify-start' : 'justify-end'">
          <div class="max-w-[70%] p-4 rounded-2xl whitespace-pre-wrap break-words" :class="msg.isBot
            ? 'bg-purple-100 text-black rounded-bl-none'
            : 'bg-purple-300 text-white rounded-br-none'">
            {{ msg.text }}
          </div>
        </div>
      </div>
    </div>

    <!-- 입력창 -->
    <form @submit.prevent="sendMessage" class="p-4 border-t bg-white w-full flex justify-center">
      <div class="w-full max-w-4xl flex gap-2">
        <input v-model="userInput" class="flex-1 p-3 border rounded text-base" placeholder="질문을 입력해주세요." />
        <button type="submit" class="px-4 py-2 bg-purple-500 text-white rounded text-base">전송</button>
      </div>
    </form>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const selectedGame = ref('카탄')
const userInput = ref('')
const messages = ref([])

const sendMessage = () => {
  if (!userInput.value.trim()) return

  const question = userInput.value
  messages.value.push({ text: question, isBot: false })
  userInput.value = ''

  setTimeout(() => {
    messages.value.push({
      text: `${selectedGame.value}에 대한 질문을 인식했어요! 곧 자세한 답변을 드릴게요.`,
      isBot: true
    })
  }, 1000)
}
</script>

<style scoped>
body {
  font-family: 'Pretendard', sans-serif;
}
</style>