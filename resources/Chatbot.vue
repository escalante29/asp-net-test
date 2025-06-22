<template>
  <div>
    <button class="fixed bottom-4 right-4 p-4 bg-blue-500 text-white rounded-full shadow-lg" @click="toggleChatbot">
      <i class="pi pi-comment"></i>
    </button>

    <div
      v-if="isOpen"
      class="fixed bottom-20 right-4 w-full max-w-sm h-[80vh] max-h-96 bg-white rounded-lg shadow-xl flex flex-col overflow-hidden sm:bottom-4"
    >
      <div class="p-4 bg-blue-600 text-white rounded-t-lg">
        Chatbot
      </div>
      <div class="flex-grow p-4 overflow-y-auto">
        <div v-for="(message, index) in messages" :key="index" class="mb-2">
          <div class="flex" :class="{ 'justify-end': message.sender === 'user' }">
            <div
              :class="{
                'inline-block p-2 rounded-lg': true,
                'bg-blue-100': message.sender === 'user',
                'bg-gray-200': message.sender === 'bot',
                'text-gray-800': true, // Ensure text color has contrast
 'max-w-[70%]': true, // Limit message width
              }"
            >
              {{ message.text }}
            </div>
          </div>
        </div>
      </div>
      <div class="p-4 border-t border-gray-200 flex">
        <input
 v-model="newMessage"
 @keyup.enter="sendMessage"
 class="flex-grow px-3 py-2 border rounded-l-lg focus:outline-none focus:ring focus:border-blue-300 bg-white text-gray-800 mr-2"
 placeholder="Type your message..." />
        <button
 @click="sendMessage"
 class="px-4 py-2 bg-blue-500 text-white rounded-r-lg hover:bg-blue-600 focus:outline-none focus:ring focus:ring-blue-300">
          Send
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const isOpen = ref(false);
const messages = ref([]);
const newMessage = ref('');

const toggleChatbot = () => {
  isOpen.value = !isOpen.value;
 if (!isOpen.value) {
 messages.value = [];
  }
};

const sendMessage = () => {
  if (newMessage.value.trim() === '') {
    return;
  }

  // Add user message
  messages.value.push({
    sender: 'user',
    text: newMessage.value,
  });

  // Simulate bot auto-reply
 simulateChatbotReply();

  newMessage.value = '';
};

const simulateChatbotReply = () => {
  setTimeout(() => {
 messages.value.push({
 sender: 'bot',
 text: 'This is a simulated reply from the chatbot.',
 });
  }, 500); // Delay for a more natural feel
};
</script>

<style scoped>
/* Scoped styles can be added here if needed, but Tailwind provides most styling */
</style>
