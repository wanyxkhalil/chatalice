<script setup>
import { ref, onMounted, computed, nextTick } from 'vue';
import { Greet, Call } from '../wailsjs/go/main/App';
import { EventsOn } from '../wailsjs/runtime';

const chats = ref([]);
const currentChatIndex = ref(0);
const userInput = ref('');
const messageContainer = ref(null);
const menuItems = ref([]);
const showSettings = ref(false);
const showAbout = ref(false);
const showSettingsList = ref(false);
const settings = ref({ name: '', key: '', baseUrl: '' });
const submittedSettings = ref([]);
const currentSettingName = ref('Setting Models');
const currentModelId = ref(0);

const submitSettings = () => {
  submittedSettings.value.push({ ...settings.value });
  currentSettingName.value = settings.value.name;
  Call("insertModel", JSON.stringify(settings.value));
  settings.value = { name: '', key: '', baseUrl: '', model: '' };
};

const toggleSettingsList = () => {
  if (submittedSettings.value.length > 0) {
    showSettingsList.value = !showSettingsList.value;
  } else {
    showSettings.value = true;
  }
};

const selectSetting = (name, id) => {
  console.log(name, id)
  currentSettingName.value = name
  currentModelId.value = id
  showSettingsList.value = false;
};

const deleteSetting = (index) => {
  submittedSettings.value.splice(index, 1);
  if (submittedSettings.value.length === 0) {
    currentSettingName.value = 'gpt-4.0';
  } else if (currentSettingName.value === submittedSettings.value[index]?.name) {
    currentSettingName.value = submittedSettings.value[0].name;
  }
};

const currentChat = computed(() => chats.value[currentChatIndex.value]);

const scrollToBottom = () => {
  nextTick(() => {
    if (messageContainer.value) {
      messageContainer.value.scrollTop = messageContainer.value.scrollHeight;
    }
  });
};

const sendMessage = () => {
  if (userInput.value.trim() !== '') {
    currentChat.value.messages.push({ text: userInput.value.trim(), isUser: true });
    Call("hello", JSON.stringify({
      Content: userInput.value.trim(),
      ChatID: chats.value[currentChatIndex.value].id,
      ModelID: currentModelId.value,
    })).then(response => {
      response = JSON.parse(response);
      currentChat.value.messages.push({ text: response.text, isUser: false, id: response.message_id });
      scrollToBottom();
    }).catch(error => {
      console.error('Error:', error);
    });
    userInput.value = '';
  }
};

const handleKeyDown = (event) => {
  if ((event.key === 'Enter' && event.ctrlKey && !navigator.platform.toUpperCase().includes('MAC')) ||
    (event.key === 'Enter' && event.metaKey && navigator.platform.toUpperCase().includes('MAC'))) {
    sendMessage();
  }
};

const refreshModelList = () => {
  Call("getModelList", "").then(response => {
    let modelList = JSON.parse(response);
    if (modelList.length > 0) {
      submittedSettings.value = modelList.map(item => ({ name: item.name, key: item.key, baseUrl: item.baseUrl, model: item.model, id: item.ID }));
      currentSettingName.value = modelList[0].name;
      currentModelId.value = modelList[0].ID;
    }
    console.log(modelList, submittedSettings.value)
  })
}

const getChats = () => {
  Call("getChats", "").then(data => {
    let response = JSON.parse(data);
    chats.value = response.map(item => ({ title: item.title, messages: item.messages, id: item.id, modelId: item.modelId }));
    if (chats.value.length == 0) {
      chats.value = [{ title: "Untitled", messages: [], id: new Date().getTime(), modelId: submittedSettings.value[0].id }];
    }
    currentChatIndex.value = 0
    currentModelId.value = chats.value[0].modelId
    currentSettingName.value = submittedSettings.value[0].name
  })
}

onMounted(() => {
  menuItems.value = [
    { icon: 'fas fa-plus', text: 'New Chat', onClickMethod: newChat },
    {
      icon: 'fas fa-cog', text: 'Settings', onClickMethod: () => {
        showSettings.value = true;
      }
    },
    {
      icon: 'fas fa-info-circle', text: 'About', onClickMethod: () => {
        showAbout.value = true;
      }
    },
  ];
  refreshModelList();
  getChats();
  window.addEventListener('keydown', handleKeyDown);

  // Clean up the event listener on unmount
  return () => {
    window.removeEventListener('keydown', handleKeyDown);
  };
});

const newChat = () => {
  chats.value.unshift({ title: "Untitled", messages: [], id: new Date().getTime() });
  currentChatIndex.value = 0;
};

const selectChat = (index) => {
  currentChatIndex.value = index;
  currentModelId.value = chats.value[currentChatIndex.value].modelId || currentModelId.value
  currentSettingName.value = submittedSettings.value.find(item => item.id === currentModelId.value)?.name
};

const markdownToHtml = (markdownText) => {
  // TODO 使用库将 Markdown 转换为 HTML
  return markdownText;
};

// 使用 window.wails.Events.On 监听事件 (需要根据实际情况调整)
EventsOn("addMessage", (message) => {
  currentChat.value.messages.push({ text: message, isUser: false });
});

EventsOn("updateChatTitle", (data) => {
  let chat = JSON.parse(data);
  chats.value.find(({ id }) => id === chat.id).title = chat.title;
});


EventsOn("appendMessage", (data) => {
  console.log("appendMessage", data);
  let message = JSON.parse(data);
  console.log("message", message, currentChat.value.messages);
  // loop currentChat.messages to find the id===message.message_id and upate the text+=text
  const msg = currentChat.value.messages.find(({ id }) => id === message.message_id);
  if (msg) {
    msg.text += message.text;
    scrollToBottom();
  }
});
</script>

<template>
  <div class="flex h-screen">
    <!-- Sidebar -->
    <div class="flex-col w-64 bg-gray-100 border-r border-gray-200">
      <div class="flex items-center justify-center h-16 border-b border-gray-200">
        <img src="./assets/images/appicon.png" alt="ChatAlice logo" class="h-6 w-6">
        <span class="text-xl font-semibold ps-2">ChatAlice</span>
      </div>
      <div class="flex-col-1 overflow-y-auto h-[calc(2/3*100vh-64px)]">
        <div class="p-4">
          <ul>
            <li v-for="(chat, index) in chats" :key="index" class="mb-2">
              <div
                :class="['flex items-center p-2 cursor-pointer rounded', currentChatIndex === index ? 'bg-gray-200' : '']"
                @click="selectChat(index)">
                <i class="fas fa-file-alt mr-2"></i>
                <span>{{ chat.title }}</span>
              </div>
            </li>
          </ul>
        </div>
      </div>
      <div class="p-4 border-t border-gray-200 h-[33%] flex flex-col justify-end">
        <div v-for="(item, index) in menuItems" :key="index" class="flex items-center mb-2 p-2 cursor-pointer"
          @click="item.onClickMethod()">
          <i :class="item.icon" class="mr-2"></i>
          <span>{{ item.text }}</span>
        </div>
      </div>
    </div>
    <!-- Main Content -->
    <div class="flex-1 flex flex-col">
      <div class="flex items-center justify-between h-16 px-4 border-b border-gray-200">
        <div class="flex items-center">
          <span class="text-lg font-medium">{{ currentChat?.title }}</span>
          <div class="relative pl-2">
            <span @click="toggleSettingsList"
              class="bg-orange-200 text-orange-800 px-2 py-1 rounded text-sm cursor-pointer">{{ currentSettingName
              }}</span>
            <div v-if="showSettingsList" class="absolute top-full left-0 mt-1 bg-white border rounded shadow-lg z-10">
              <div v-for="setting in submittedSettings" :key="setting.name"
                @click="selectSetting(setting.name, setting.id)" class="px-4 py-2 hover:bg-gray-100 cursor-pointer">
                {{ setting.name }}
              </div>
            </div>
          </div>
        </div>

      </div>
      <div ref="messageContainer" class="flex-1 p-4 overflow-y-auto message-scroll">
        <div v-for="(msg, index) in currentChat?.messages" :key="index" class="mb-4">
          <div class="flex items-start ">
            <div class="flex-shrink-0 mr-3">
              <i
                :class="['fas w-6', msg.isUser ? 'fa-user' : 'fa-robot', 'text-2xl', msg.isUser ? 'text-blue-500' : 'text-green-500']"></i>
            </div>
            <div class="flex-grow">
              <!-- <p class="text-gray-800 text-left">{{ markdown.render(msg.text) }}</p> -->
              <p class="text-gray-800 text-left" v-html="markdownToHtml(msg.text)"></p>
              <i class="fas fa-spinner fa-1x fa-spin text-gray-800" v-if="!msg.text"></i>
            </div>
          </div>
        </div>
      </div>
      <div class="flex items-center p-4 border-t border-gray-200" style="height: 33.33%;">
        <div class="flex-1 flex items-center relative" style="height: 100%;">
          <textarea v-model="userInput" placeholder="Type your question here..."
            class="w-full h-full px-4 py-2 border border-gray-300 rounded-md focus:outline-none"></textarea>
          <div class="floating-icons">
            <div class="absolute inset-y-0 right-0 flex items-center space-x-4 pr-4">
              <i class="fas fa-plus-square text-gray-500 cursor-pointer"></i>
              <i class="fas fa-image text-gray-500 cursor-pointer"></i>
              <i class="fas fa-file-alt text-gray-500 cursor-pointer"></i>
              <i class="fas fa-folder-open text-gray-500 cursor-pointer"></i>
              <i class="fas fa-paperclip text-gray-500 cursor-pointer"></i>
              <i :class="['fas', 'fa-paper-plane', 'text-blue-500', 'cursor-pointer', { 'disabled': !userInput.trim() }]"
                @click="sendMessage"></i>
            </div>
          </div>
        </div>
      </div>
    </div>


    <!-- Settings Modal -->
    <div v-if="showSettings" class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center">
      <div class="bg-white rounded-lg p-6 w-96">
        <h2 class="text-2xl font-bold mb-4">Settings</h2>
        <form @submit.prevent="submitSettings">
          <div class="mb-4">
            <label class="block text-sm font-medium text-gray-700">Name</label>
            <input v-model="settings.name" type="text"
              class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50">
          </div>
          <div class="mb-4">
            <label class="block text-sm font-medium text-gray-700">Key</label>
            <input v-model="settings.key" type="text"
              class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50">
          </div>
          <div class="mb-4">
            <label class="block text-sm font-medium text-gray-700">Base URL</label>
            <input v-model="settings.baseUrl" type="text"
              class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50">
          </div>
          <div class="mb-4">
            <label class="block text-sm font-medium text-gray-700">Model</label>
            <input v-model="settings.model" type="text"
              class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50">
          </div>
          <button type="submit" class="w-full bg-blue-500 text-white rounded-md py-2 hover:bg-blue-600">Submit</button>
        </form>
        <div class="mt-4">
          <h3 class="font-bold mb-2">Submitted Settings:</h3>
          <ul class="list-disc pl-5">
            <li v-for="(setting, index) in submittedSettings" :key="index" class="flex justify-between items-center">
              <span>Name: {{ setting.name }}, Key: {{ setting.key }}, Base URL: {{ setting.baseUrl }},Model:{{
                setting.model }}</span>
              <button @click="deleteSetting(index)" class="text-red-500 hover:text-red-700">
                <i class="fas fa-trash-alt"></i>
              </button>
            </li>
          </ul>
        </div>
        <button @click="showSettings = false"
          class="mt-4 w-full bg-gray-300 text-gray-800 rounded-md py-2 hover:bg-gray-400">Close</button>
      </div>
    </div>

    <!-- About Model-->
    <div v-if="showAbout"
      class="fixed inset-0 mx-auto p-8 w-96 bg-gradient-to-r from-blue-100 to-purple-100 rounded-lg shadow-lg">
      <h1 class="text-3xl font-bold text-center text-blue-600 mb-6">欢迎来到ChatAlice的奇幻世界！🐰🍄</h1>

      <p class="text-lg text-gray-600 mb-3">
        ChatAlice 是一位充满好奇心的AI聊天伙伴，如同闯入科技奇境的现代爱丽丝。每次对话都是一场穿越想象力边界的冒险！
      </p>

      <p class="text-lg text-gray-600 mb-3">
        深奥的技术话题，天马行空的想象，Alice都会以温暖而智慧的方式陪伴左右。跟随我们的白兔，跳入这个充满惊喜的数字兔子洞吧？
      </p>

      <p class="text-lg text-gray-600 mb-6">
        有趣的是，在计算机世界里，Alice常常和她的好朋友Bob一起出现在各种思想实验中。而在我们的项目里，Alice决定独自前行，成为你的专属对话伙伴！
      </p>

      <div class="text-center">
        <a href="https://github.com/JessonChan/chatalice"
          class="inline-block bg-blue-500 hover:bg-blue-600 text-white font-bold py-2 px-4 rounded transition duration-300">
          在GitHub上探索我们的仙境
        </a>
      </div>

      <p class="text-sm text-gray-500 mt-6 text-center italic">
        "在这里，我们都有点疯狂。你疯狂，我疯狂。但我会告诉你一个秘密，最棒的人都是疯狂的。" - 柴郡猫（可能是程序员）
      </p>
      <button @click="showAbout = false"
        class="mt-4 w-full bg-gray-300 text-gray-800 rounded-md py-2 hover:bg-gray-400">Close</button>
    </div>
  </div>
</template>

<style>
/* 使用 @import 引入外部 CSS */
@import 'https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.3/css/all.min.css';
@import 'https://fonts.googleapis.com/css2?family=Roboto:wght@400;500;700&display=swap';


body {
  font-family: 'Roboto', sans-serif;
}

.disabled {
  pointer-events: none;
  opacity: 0.5;
}

.floating-icons {
  position: absolute;
  bottom: 0.5rem;
  right: 0.5rem;
  display: flex;
  gap: 0.5rem;
}

.message-scroll {
  scrollbar-width: thin;
  scrollbar-color: #CBD5E0 #EDF2F7;
}

.message-scroll::-webkit-scrollbar {
  width: 8px;
}

.message-scroll::-webkit-scrollbar-track {
  background: #EDF2F7;
}

.message-scroll::-webkit-scrollbar-thumb {
  background-color: #CBD5E0;
  border-radius: 4px;
}
</style>