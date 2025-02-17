  <!-- filepath: /C:/Users/Regal/Desktop/niks/kimi-k1.5/frontend/src/App.vue -->
  <template>
    <div class="h-screen flex justify-center bg-[#DAD2FF]">
      <div class="w-full max-w-4xl flex flex-col bg-white/90 shadow-lg relative backdrop-blur-sm">
        <!-- Fixed Header -->
        <header class="bg-gradient-to-r from-[#493D9E] to-[#B2A5FF] p-4 shadow-md fixed top-0 w-full max-w-4xl z-10">
          <h1 class="text-2xl font-bold text-white">Kimi k-1.5</h1>
        </header>

        <!-- Messages container with padding for header and footer -->
        <div 
          class="flex-1 overflow-y-auto space-y-4 px-4 scroll-smooth bg-gradient-to-b from-white/50 to-[#DAD2FF]/30" 
          ref="messagesContainer"
          style="margin-top: 64px; margin-bottom: 80px;"
        >
          <div
            v-for="(message, index) in messages"
            :key="index"
            class="flex"
            :class="[message.role === 'user' ? 'justify-end' : 'justify-start']"
          >
            <div
              class="rounded-2xl px-4 py-2 max-w-[85%] shadow-sm"
              :class="[
                message.role === 'user'
                  ? 'bg-[#493D9E] text-white rounded-br-sm'
                  : 'bg-[#FFF2AF] text-gray-800 rounded-bl-sm'
              ]"
            >
              <div
                v-if="message.role === 'assistant'" 
                class="relative"
              >
                <!-- Think toggle button -->
                <button 
                  v-if="hasThinkContent(message.content)"
                  @click="toggleThink(index)"
                  class="absolute -top-6 -right-6 p-0 rounded-full transition-transform hover:scale-110"
                  :class="showThinkMap[index] ? 'opacity-100' : 'opacity-70'"
                >
                  <div class="relative drop-shadow-lg">
                    <span class="text-lg filter drop-shadow-md">
                      {{ showThinkMap[index] ? '🐵' : '🙈 ' }}
                    </span>
                  </div>
                </button>
                
                <!-- Message content -->
                <div 
                  v-html="formatMessage(message.content, showThinkMap[index])"
                  class="prose prose-sm max-w-none whitespace-pre-wrap"
                ></div>
              </div>
              <div v-else>{{ message.content }}</div>
            </div>
          </div>
          
          <!-- Streaming content -->
          <div v-if="streamingContent" class="flex justify-start">
            <div 
              class="bg-[#FFF2AF] rounded-2xl rounded-bl-sm px-4 py-2 max-w-[85%] shadow-sm prose prose-sm"
              v-html="formatMessage(streamingContent)"
            ></div>
          </div>

          <!-- Loading indicator -->
          <div v-if="isLoading && !streamingContent" class="flex justify-start">
            <div class="bg-[#FFF2AF] rounded-2xl rounded-bl-sm px-4 py-2 shadow-sm flex items-center space-x-2">
              <div class="w-2 h-2 bg-[#493D9E] rounded-full animate-bounce"></div>
              <div class="w-2 h-2 bg-[#493D9E] rounded-full animate-bounce" style="animation-delay: 0.2s"></div>
              <div class="w-2 h-2 bg-[#493D9E] rounded-full animate-bounce" style="animation-delay: 0.4s"></div>
            </div>
          </div>
        </div>

        <!-- Fixed Footer Form -->
        <form 
          @submit.prevent="handleSubmit" 
          class="fixed bottom-0 w-full max-w-4xl bg-white/80 border-t border-[#B2A5FF] p-4 shadow-lg z-10 backdrop-blur-sm"
        >
          <div class="flex gap-2">
            <input
              type="text"
              v-model="input"
              class="flex-1 p-3 border border-[#B2A5FF] rounded-xl focus:ring-2 focus:ring-[#493D9E] focus:border-[#493D9E] outline-none transition-all bg-white/90"
              placeholder="Type your message..."
              :disabled="isLoading"
            />
            <button
              type="submit"
              :disabled="isLoading && !canStopGeneration"
              class="px-6 py-3 rounded-xl transition-colors disabled:bg-gray-300 disabled:cursor-not-allowed flex items-center gap-2 text-white"
              :class="isLoading ? 'bg-red-500 hover:bg-red-600' : 'bg-[#493D9E] hover:bg-[#B2A5FF]'"
            >
              <span>{{ isLoading ? 'Stop' : 'Send' }}</span>
              <svg v-if="!isLoading" xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
                <path d="M10.894 2.553a1 1 0 00-1.788 0l-7 14a1 1 0 001.169 1.409l5-1.429A1 1 0 009 15.571V11a1 1 0 112 0v4.571a1 1 0 00.725.962l5 1.428a1 1 0 001.17-1.408l-7-14z" />
              </svg>
              <svg v-else xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
                <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM8 7a1 1 0 00-1 1v4a1 1 0 102 0V8a1 1 0 00-1-1zm4 0a1 1 0 00-1 1v4a1 1 0 102 0V8a1 1 0 00-1-1z" clip-rule="evenodd" />
              </svg>
            </button>
          </div>
        </form>
      </div>
    </div>
  </template>

  <script setup>
  import { ref, watch, nextTick } from 'vue'
  import { marked } from 'marked'
  import DOMPurify from 'dompurify'
  import katex from 'katex'
  import 'katex/dist/katex.min.css'

  const input = ref('')
  const messages = ref([])
  const isLoading = ref(false)
  const streamingContent = ref('')
  const messagesContainer = ref(null)
  const canStopGeneration = ref(false)
  const controller = ref(null)
  const showThinkMap = ref({})

  // Scroll to bottom when messages or streaming content changes
  watch([messages, streamingContent], async () => {
    await nextTick()
    if (messagesContainer.value) {
      messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight
    }
  }, { deep: true })

  const handleSubmit = async () => {
    if (isLoading.value) {
      // Stop generation
      controller.value?.abort()
      isLoading.value = false
      canStopGeneration.value = false
      return
    }

    if (!input.value.trim()) return

    const userMessage = input.value.trim()
    input.value = ''
    isLoading.value = true
    streamingContent.value = ''
    canStopGeneration.value = true
    controller.value = new AbortController()

    // Add user message immediately
    messages.value.push({ role: 'user', content: userMessage })

    try {
      const response = await fetch('http://localhost:3000/api/chat', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          message: userMessage,
          conversationHistory: []
        }),
        signal: controller.value.signal
      })

      const reader = response.body.getReader()
      const decoder = new TextDecoder()
      let fullContent = ''

      while (true) {
        const { done, value } = await reader.read()
        if (done) break

        const chunk = decoder.decode(value)
        const lines = chunk.split('\n')

        for (const line of lines) {
          if (line.trim() === '') continue
          if (line === 'data: [DONE]') continue

          if (line.startsWith('data: ')) {
            try {
              const jsonData = JSON.parse(line.slice(6))
              if (jsonData.content) {
                fullContent += jsonData.content
                streamingContent.value = fullContent
                // Scroll to bottom on each update
                await nextTick()
                if (messagesContainer.value) {
                  messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight
                }
              }
            } catch (e) {
              console.error('Error parsing JSON:', e)
            }
          }
        }
      }

      messages.value.push({
        role: 'assistant',
        content: fullContent
      })
      streamingContent.value = ''

    } catch (error) {
      if (error.name === 'AbortError') {
        messages.value.push({
          role: 'assistant',
          content: 'Message generation stopped.'
        })
      } else {
        console.error('Error:', error)
        messages.value.push({
          role: 'assistant',
          content: 'Sorry, there was an error processing your request.'
        })
      }
    } finally {
      isLoading.value = false
      canStopGeneration.value = false
      controller.value = null
    }
  }

  // Check if message has think content
  const hasThinkContent = (content) => {
    return content.includes('◁think▷')
  }

  // Toggle think content visibility
  const toggleThink = (messageIndex) => {
    showThinkMap.value[messageIndex] = !showThinkMap.value[messageIndex]
  }

  // Modified formatMessage function
  const formatMessage = (content, showThink = false) => {
    // Handle ASCII art blocks
    content = content.replace(
      /```ascii\n([\s\S]*?)```/g, 
      '<div class="ascii-art">$1</div>'
    )
    
    // Handle think markers
    if (!showThink) {
      content = content.replace(/◁think▷.*?◁\/think▷/gs, '')
    } else {
      content = content.replace(
        /◁think▷(.*?)◁\/think▷/gs, 
        '<div class="think-content"><span class="think-label">Thinking:</span>$1</div>'
      )
    }
    
    // Helper function to safely render math
    const renderMath = (formula, displayMode = false) => {
      try {
        // Clean the formula - remove any invisible characters and fix common issues
        let cleanFormula = formula.trim()
          .replace(/[\u200B-\u200D\uFEFF]/g, '')
          .replace(/\\int\s+([^,]+),\s*([^,]+)/g, '\\int $1 \\, $2') // Fix integral notation
          .replace(/,\s*(dx|dy|dz|dt)/g, '\\, $1') // Fix differential notation
          .replace(/\s+/g, ' '); // Normalize spaces

        // If formula is just a simple expression without LaTeX commands
        if (!/\\[a-zA-Z]/.test(cleanFormula)) {
          cleanFormula = cleanFormula
            .replace(/\b([a-zA-Z]+)\b/g, '\\text{$1}') // Convert words to text mode
            .replace(/(\d*[a-zA-Z])\^(\d+|[a-zA-Z])/g, '{$1}^{$2}') // Fix exponents
            .replace(/([a-zA-Z0-9])\s*=\s*([a-zA-Z0-9])/g, '$1 = $2'); // Fix equals signs
        }

        const result = katex.renderToString(cleanFormula, {
          displayMode,
          throwOnError: false,
          strict: false,
          trust: true,
          macros: {
            "\\n": "n",
            "\\x": "x",
            "\\b": "b",
            "\\a": "a"
          }
        });

        // If KaTeX returned an error span, return the original formula
        if (result.includes('katex-error')) {
          return formula;
        }

        return result;
      } catch (e) {
        console.error('KaTeX error:', e);
        // Return the original text if there's an error
        return formula;
      }
    }

    // Handle LaTeX math expressions with different delimiters
    // First handle display math with parentheses
    content = content.replace(/\(([\s\S]*?)\)/g, (match, formula) => {
      // Don't process if it looks like regular text in parentheses
      if (!/[\\$_{}^=]/.test(formula) && !/\d+[a-zA-Z]/.test(formula)) return match;
      return renderMath(formula, true);
    });

    // Handle display math with square brackets
    content = content.replace(/\[([\s\S]*?)\]/g, (match, formula) => {
      return renderMath(formula, true);
    });

    // Handle inline math with dollars
    content = content.replace(/\$(.*?)\$/g, (match, formula) => {
      return renderMath(formula, false);
    });
    
    // Convert markdown to HTML and sanitize
    const html = marked(content)
    return DOMPurify.sanitize(html, {
      ADD_TAGS: ['math', 'annotation', 'semantics', 'mrow', 'mstyle', 'mn', 'mi', 'mo'],
      ADD_ATTR: ['xmlns', 'xmlns:xlink', 'aria-hidden', 'data-math-type', 'class', 'style', 'mathvariant']
    })
  }
  </script>

  <style>
  /* Add any custom styles here */
  html, body {
    height: 100%;
    margin: 0;
    background-color: #DAD2FF;
    overflow-y: hidden;
  }

  #app {
    height: 100%;
    width: 100%;
    padding: 0;
  }

  header {
    background-color:rgb(241, 186, 186);
    border-bottom: 1px solid #e5e7eb;
  }

  input {
    border: 1px solid #e5e7eb;
  }

  .fitContent{
    width: fit-content;
  }

  button {
    transition: background-color 0.3s;
  }

  button:hover {
    background-color: #2563eb;
  }

  /* Style markdown elements */
  .prose pre {
    background: #DAD2FF;
    padding: 1rem;
    border-radius: 0.5rem;
    margin: 0.5rem 0;
    overflow-x: auto;
  }

  .prose code {
    background: #DAD2FF;
    padding: 0.2rem 0.4rem;
    border-radius: 0.25rem;
    font-size: 0.875rem;
    color: #493D9E;
  }

  .prose p {
    margin: 0.5rem 0;
  }

  /* Smooth scrolling */
  .overflow-y-auto {
    scroll-behavior: smooth;
  }

  /* Custom scrollbar */
  .overflow-y-auto::-webkit-scrollbar {
    width: 8px;
  }

  .overflow-y-auto::-webkit-scrollbar-track {
    background: #DAD2FF;
  }

  .overflow-y-auto::-webkit-scrollbar-thumb {
    background: #B2A5FF;
    border-radius: 4px;
  }

  .overflow-y-auto::-webkit-scrollbar-thumb:hover {
    background: #493D9E;
  }

  /* Ensure content doesn't go under fixed elements */
  .overflow-y-auto {
    scroll-padding-top: 64px;
    scroll-padding-bottom: 80px;
  }

  /* Remove default outline on button focus */
  button:focus {
    outline: none;
  }

  /* Math formula styles */
  .katex {
    font-size: 1.1em;
    padding: 0.2em 0;
  }

  .katex-display {
    margin: 1em 0;
    overflow-x: auto;
    overflow-y: hidden;
  }

  /* Improve code block spacing when mixed with math */
  .prose pre {
    margin: 1.5em 0;
  }

  /* Ensure math formulas don't overflow on mobile */
  @media (max-width: 640px) {
    .katex-display {
      font-size: 0.9em;
    }
    
    .katex {
      font-size: 1em;
    }
  }

  /* ASCII Art and Diagram Styles */
  .ascii-art {
    font-family: 'Courier New', Courier, monospace;
    white-space: pre;
    line-height: 1.2;
    font-size: 14px;
    padding: 1rem;
    background: #f8f9fa;
    border-radius: 0.5rem;
    overflow-x: auto;
  }

  /* Think content styles */
  .think-content {
    margin-top: 0.5rem;
    padding: 0.5rem;
    background-color: rgba(73, 61, 158, 0.1);
    border-left: 3px solid #493D9E;
    border-radius: 0.25rem;
    font-style: italic;
  }

  .think-label {
    display: inline-block;
    font-weight: 600;
    color: #493D9E;
    margin-right: 0.5rem;
    font-style: normal;
  }

  /* Update think toggle button styles */
  button.absolute {
    transition: all 0.2s ease;
    z-index: 20;
  }

  button.absolute:hover {
    transform: scale(1.1);
  }
  </style>