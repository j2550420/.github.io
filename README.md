<!DOCTYPE html>
<html lang="zh-TW">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>演講即時 Q&A</title>
  <script src="https://cdn.tailwindcss.com"></script>
  
  <!-- 1. 定義邏輯函式 (移至 Head 以確保優先載入) -->
  <script>
    function chatWidget() {
      return {
        isOpen: false,
        isLoading: false,
        hasNewMessage: false,
        userInput: '',
        messages: [], 
        
        // ★★★ 請替換為您的 N8N Webhook URL ★★★
        config: {
          questionWebhook: 'https://YOUR_N8N_INSTANCE/webhook/chat/ask',
          feedbackWebhook: 'https://YOUR_N8N_INSTANCE/webhook/chat/feedback'
        },

        userId: localStorage.getItem('chat_user_id') || 'user_' + Math.random().toString(36).substr(2, 9),

        init() {
          if (!localStorage.getItem('chat_user_id')) {
            localStorage.setItem('chat_user_id', this.userId);
          }
        },

        toggleChat() {
          this.isOpen = !this.isOpen;
          if(this.isOpen) {
            this.hasNewMessage = false;
            this.$nextTick(() => this.scrollToBottom());
          }
        },

        scrollToBottom() {
          const container = document.getElementById('chat-messages');
          if (container) {
            container.scrollTo({ top: container.scrollHeight, behavior: 'smooth' });
          }
        },

        async sendMessage() {
          if (!this.userInput.trim()) return;
          
          const text = this.userInput;
          this.userInput = '';
          
          this.messages.push({ text: text, isUser: true });
          this.isLoading = true;
          this.$nextTick(() => this.scrollToBottom());

          try {
            const response = await fetch(this.config.questionWebhook, {
              method: 'POST',
              headers: { 'Content-Type': 'application/json' },
              body: JSON.stringify({
                question: text,
                user_id: this.userId,
                timestamp: new Date().toISOString()
              })
            });
            
            if (!response.ok) throw new Error('API Error');
            const data = await response.json();
            
            this.messages.push({
              text: data.answer || "抱歉，系統正忙，請稍後再試。",
              isUser: false,
              requiresFeedback: data.requires_feedback === true,
              feedbackGiven: false,
              questionId: data.question_id 
            });
            
            if (!this.isOpen) this.hasNewMessage = true;

          } catch (error) {
            console.error('Error:', error);
            this.messages.push({ text: "連線錯誤，請檢查網路狀態。", isUser: false });
          } finally {
            this.isLoading = false;
            this.$nextTick(() => this.scrollToBottom());
          }
        },

        async submitFeedback(questionId, isSolved, msgIndex) {
          if (this.messages[msgIndex].feedbackGiven) return;
          
          this.messages[msgIndex].feedbackGiven = true;
          const replyText = isSolved 
            ? "感謝您的回饋！(系統標記：已解決)" 
            : "已幫您轉交給講者，將於簡報後回覆。";
            
          this.messages.push({ text: replyText, isUser: false });
          this.$nextTick(() => this.scrollToBottom());

          try {
            await fetch(this.config.feedbackWebhook, {
              method: 'POST',
              headers: { 'Content-Type': 'application/json' },
              body: JSON.stringify({ question_id: questionId, is_solved: isSolved })
            });
          } catch (e) { console.error(e); }
        }
      }
    }
  </script>

  <!-- 2. 載入 Alpine.js (加上 defer) -->
  <script src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>
  
  <style>
    [x-cloak] { display: none !important; }
    .scrollbar-hide::-webkit-scrollbar { display: none; }
    .scrollbar-hide { -ms-overflow-style: none; scrollbar-width: none; }
    .animate-fade-in-up { animation: fadeInUp 0.5s ease-out; }
    @keyframes fadeInUp {
      from { opacity: 0; transform: translateY(10px); }
      to { opacity: 1; transform: translateY(0); }
    }
  </style>
</head>
<body class="bg-gradient-to-br from-indigo-50 via-white to-purple-50 min-h-screen font-sans text-gray-700 overflow-hidden">

  <!-- Main Landing Content (Background) -->
  <div class="flex flex-col items-center justify-center h-screen p-6 text-center z-0 relative">
    <div class="mb-8 p-6 bg-white/60 backdrop-blur-sm rounded-3xl shadow-xl border border-white/50 animate-fade-in-up max-w-md w-full">
      <div class="w-16 h-16 bg-gradient-to-tr from-violet-600 to-indigo-600 rounded-2xl flex items-center justify-center mx-auto mb-4 shadow-lg text-white">
        <svg class="w-8 h-8" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 12h.01M12 12h.01M16 12h.01M21 12c0 4.418-4.03 8-9 8a9.863 9.863 0 01-4.255-.949L3 20l1.395-3.72C3.512 15.042 3 13.574 3 12c0-4.418 4.03-8 9-8s9 3.582 9 8z"></path></svg>
      </div>
      <h1 class="text-2xl font-bold text-gray-800 mb-2">歡迎參加本次演講</h1>
      <p class="text-gray-600 mb-6 text-sm leading-relaxed">
        如果您在聽講過程中有任何疑問，<br>請點擊右下角的按鈕與 <b>AI 助理</b> 互動。
      </p>
      <div class="flex items-center justify-center gap-2 text-xs text-gray-400">
        <span class="w-2 h-2 bg-green-400 rounded-full animate-pulse"></span>
        系統線上運作中
      </div>
    </div>
  </div>
  
  <!-- Chat Widget Container -->
  <div x-data="chatWidget()" x-cloak class="fixed bottom-0 right-0 p-4 z-50 flex flex-col items-end space-y-4 w-full md:w-auto pointer-events-none">
    
    <!-- Chat Window -->
    <div x-show="isOpen" 
         x-transition:enter="transition ease-out duration-300 transform"
         x-transition:enter-start="opacity-0 translate-y-10 scale-95"
         x-transition:enter-end="opacity-100 translate-y-0 scale-100"
         x-transition:leave="transition ease-in duration-200 transform"
         x-transition:leave-start="opacity-100 translate-y-0 scale-100"
         x-transition:leave-end="opacity-0 translate-y-10 scale-95"
         class="pointer-events-auto w-full md:w-[400px] bg-white rounded-2xl shadow-2xl overflow-hidden flex flex-col max-h-[80vh] h-[600px] border border-gray-200 ring-1 ring-black/5">
      
      <!-- Header -->
      <div class="bg-gradient-to-r from-violet-600 to-indigo-600 p-4 flex justify-between items-center text-white shrink-0 shadow-sm">
        <div class="flex items-center gap-3">
          <div class="relative">
             <div class="w-2.5 h-2.5 bg-green-400 rounded-full border-2 border-indigo-600 absolute bottom-0 right-0"></div>
             <div class="w-8 h-8 bg-white/20 rounded-full flex items-center justify-center backdrop-blur-sm">
                <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9.75 17L9 20l-1 1h8l-1-1-.75-3M3 13h18M5 17h14a2 2 0 002-2V5a2 2 0 00-2-2H5a2 2 0 00-2 2v10a2 2 0 002 2z"></path></svg>
             </div>
          </div>
          <div>
            <h3 class="font-bold text-base">演講 Q&A 助理</h3>
            <p class="text-[10px] opacity-80 uppercase tracking-wider">AI Powered</p>
          </div>
        </div>
        <button @click="isOpen = false" class="hover:bg-white/20 rounded-full p-2 transition focus:outline-none">
          <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"></path></svg>
        </button>
      </div>

      <!-- Messages Area -->
      <div class="flex-1 overflow-y-auto p-4 space-y-4 bg-slate-50 scrollbar-hide" id="chat-messages">
        <!-- Welcome Message -->
        <div class="flex justify-start animate-fade-in-up">
          <div class="w-8 h-8 bg-indigo-100 rounded-full flex items-center justify-center mr-2 shrink-0 border border-indigo-200 text-indigo-600">
             <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path></svg>
          </div>
          <div class="bg-white border border-gray-100 text-gray-800 rounded-2xl rounded-tl-none py-3 px-4 max-w-[85%] shadow-sm text-sm">
            <p class="font-semibold text-indigo-600 mb-1 text-xs">AI Assistant</p>
            <p>👋 嗨！我是這場演講的 AI 助理。</p>
            <p class="mt-2 text-gray-600">您可以隨時輸入問題，我會盡力回答。若我無法回答，也會幫您記錄下來轉交給講者喔！</p>
          </div>
        </div>

        <template x-for="(msg, index) in messages" :key="index">
          <div class="flex w-full animate-fade-in-up" :class="msg.isUser ? 'justify-end' : 'justify-start'">
            
            <!-- Bot Avatar -->
            <div x-show="!msg.isUser" class="w-8 h-8 bg-indigo-100 rounded-full flex items-center justify-center mr-2 shrink-0 border border-indigo-200 text-indigo-600">
                <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 10V3L4 14h7v7l9-11h-7z"></path></svg>
            </div>

            <div class="flex flex-col max-w-[85%]" :class="msg.isUser ? 'items-end' : 'items-start'">
                <!-- Message Bubble -->
                <div :class="msg.isUser ? 'bg-indigo-600 text-white rounded-tr-none' : 'bg-white border border-gray-100 text-gray-800 rounded-tl-none'"
                    class="rounded-2xl py-3 px-4 shadow-sm text-sm break-words relative leading-relaxed">
                    <p x-text="msg.text" class="whitespace-pre-wrap"></p>
                </div>

                <!-- Feedback UI -->
                <div x-show="!msg.isUser && msg.requiresFeedback && !msg.feedbackGiven" 
                    class="mt-2 flex items-center gap-2 text-xs text-gray-500 bg-white/50 p-1.5 rounded-lg border border-gray-100 backdrop-blur-sm">
                    <span>是否解決問題？</span>
                    <button @click="submitFeedback(msg.questionId, true, index)" 
                            class="px-2 py-1 bg-white border border-green-200 text-green-600 rounded hover:bg-green-50 transition shadow-sm font-medium">
                        是
                    </button>
                    <button @click="submitFeedback(msg.questionId, false, index)" 
                            class="px-2 py-1 bg-white border border-red-200 text-red-600 rounded hover:bg-red-50 transition shadow-sm font-medium">
                        否
                    </button>
                </div>
                
                <div x-show="msg.feedbackGiven" class="mt-1 text-[10px] text-gray-400 italic px-1">
                    (感謝您的回饋)
                </div>
            </div>

          </div>
        </template>
        
        <!-- Loading -->
        <div x-show="isLoading" class="flex justify-start animate-pulse">
            <div class="w-8 h-8 bg-gray-200 rounded-full mr-2 shrink-0"></div>
            <div class="bg-white border border-gray-100 rounded-2xl rounded-tl-none py-3 px-4 shadow-sm flex items-center gap-1">
                <span class="w-1.5 h-1.5 bg-indigo-400 rounded-full animate-bounce"></span>
                <span class="w-1.5 h-1.5 bg-indigo-400 rounded-full animate-bounce delay-75"></span>
                <span class="w-1.5 h-1.5 bg-indigo-400 rounded-full animate-bounce delay-150"></span>
            </div>
        </div>
      </div>

      <!-- Input Area -->
      <div class="p-3 bg-white border-t border-gray-100 shrink-0">
        <form @submit.prevent="sendMessage" class="flex gap-2 relative">
          <input type="text" x-model="userInput" placeholder="輸入您的問題..." 
                 class="flex-1 bg-gray-50 border border-gray-200 rounded-full pl-5 pr-12 py-3 text-sm focus:outline-none focus:border-indigo-500 focus:ring-2 focus:ring-indigo-100 focus:bg-white transition shadow-inner"
                 :disabled="isLoading">
          <button type="submit" 
                  class="absolute right-1.5 top-1.5 p-1.5 bg-indigo-600 text-white rounded-full hover:bg-indigo-700 transition disabled:opacity-50 disabled:cursor-not-allowed shadow-md hover:shadow-lg transform active:scale-95 duration-200 h-9 w-9 flex items-center justify-center"
                  :disabled="!userInput.trim() || isLoading">
            <svg class="w-4 h-4 ml-0.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 19l9 2-9-18-9 18 9-2zm0 0v-8"></path></svg>
          </button>
        </form>
        <div class="text-center mt-2 flex justify-center items-center gap-1 opacity-40 hover:opacity-100 transition duration-300">
            <span class="text-[10px] text-gray-500">Powered by N8N & AI</span>
        </div>
      </div>
    </div>

    <!-- Floating Toggle Button -->
    <button @click="toggleChat" 
            class="pointer-events-auto group bg-gradient-to-br from-violet-600 to-indigo-600 hover:from-violet-700 hover:to-indigo-700 text-white rounded-full p-4 shadow-xl shadow-indigo-500/30 transition duration-300 transform hover:scale-105 flex items-center justify-center relative ring-4 ring-white focus:outline-none z-50">
      <span x-show="!isOpen" class="absolute transition-transform duration-300 group-hover:rotate-12">
        <svg class="w-8 h-8" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 10h.01M12 10h.01M16 10h.01M9 16H5a2 2 0 01-2-2V6a2 2 0 012-2h14a2 2 0 012 2v8a2 2 0 01-2 2h-5l-5 5v-5z"></path></svg>
      </span>
      <span x-show="isOpen" x-cloak class="absolute">
        <svg class="w-8 h-8" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"></path></svg>
      </span>
      <span class="absolute top-0 right-0 block h-4 w-4 rounded-full ring-2 ring-white bg-red-500 transform translate-x-1 -translate-y-1 shadow-sm animate-pulse" x-show="!isOpen && hasNewMessage"></span>
    </button>
  </div>
</body>
</html>
