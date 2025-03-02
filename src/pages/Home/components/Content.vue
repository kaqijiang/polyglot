<script setup lang="ts">
import Button from '@/components/Button.vue'
import { generatTranslate, generateText } from '@/server/api'
import { getAzureTranslateKey, verifyOpenKey } from '@/utils'
import { useConversationStore } from '@/stores'

interface Translates {
  [key: string]: {
    isShow: boolean
    result: string | boolean
  }
}

// hooks
const store = useConversationStore()
const { el, scrollToBottom } = useScroll()
const { selfAvatar, openKey, openProxy } = useGlobalSetting()

const {
  language,
  voiceName,
  rate,
  isRecognizing,
  isPlaying,
  startRecognizeSpeech,
  isRecognizReadying,
  stopRecognizeSpeech,
  ssmlToSpeak,
  isSynthesizing,
} = useSpeechService({ langs: store.allLanguage as any })

// states
const message = ref('') // input message
const text = ref('') // current select message
const translates = ref<Translates>({}) // translate result
const isTranslating = ref(false) // translate loading
const speakIndex = ref(0) // record speak
const translateIndex = ref(0) // record translate

const messageLength = computed(() => store.getConversationsByCurrentProps('chatMessages').length)
const chatMessages = computed(() => store.getConversationsByCurrentProps('chatMessages').slice(1))// 除去第一条系统设置的消息
const currentChatMessages = computed(() => store.getConversationsByCurrentProps('chatMessages'))
const currentKey = computed(() => store.currentKey)
const currentName = computed(() => store.getConversationsByCurrentProps('name'))
const currentAvatar = computed(() => store.getConversationsByCurrentProps('avatar'))
const currentLanguage = computed(() => store.getConversationsByCurrentProps('language'))
const currentVoice = computed(() => store.getConversationsByCurrentProps('voice'))
const currentRate = computed(() => store.getConversationsByCurrentProps('rate'))

useTitle(currentName)

// 设置空格快捷键
useEventListener(document, 'keydown', (e) => {
  if (isRecognizing.value || isRecognizReadying.value || e.code !== 'Space') return
  message.value = ''
  startRecognizeSpeech()
})

useEventListener(document, 'keyup', async (e) => {
  if ((!isRecognizing.value && !isRecognizReadying.value) || e.code !== 'Space') return
  await stopRecognizeSpeech((textSlice) => {
    message.value += textSlice || ''
  })
  onSubmit()
  console.log('submit', message.value)
})

// effects
watch(messageLength, () => nextTick(() => scrollToBottom()))
watch(currentKey, () => {
  language.value = currentLanguage.value as any
  voiceName.value = currentVoice.value
  rate.value = currentRate.value
})

// methods
const fetchResponse = async (key: string) => {
  let res
  try {
    res = await generateText(currentChatMessages.value, key, openProxy.value)
  }
  catch (error: any) {
    return alert('[Error] 网络请求超时, 请检查网络或代理')
  }
  if (res.error) return alert(res.error?.message)

  return res.choices[0].message.content
}

async function onSubmit() {
  if (!verifyOpenKey(openKey.value)) return alert('请输入正确的API-KEY')
  if (!message.value) return

  store.changeConversations([
    ...currentChatMessages.value,
    { content: message.value, role: 'user' },
  ])
  message.value = ''
  store.changeLoading(true)

  const content = await fetchResponse(openKey.value)
  if (content) {
    store.changeConversations([
      ...currentChatMessages.value,
      {
        content, role: 'assistant',
      },
    ])
    speak(content, chatMessages.value.length - 1)
  }
  else {
    store.changeConversations(currentChatMessages.value.slice(0, -1))
  }

  store.changeLoading(false)
}

function speak(content: string, index: number) {
  console.log(isPlaying.value)
  if (isPlaying.value) return
  speakIndex.value = index
  text.value = content
  ssmlToSpeak(content)
}

const recognize = async () => {
  try {
    console.log('isRecognizing', isRecognizing.value)
    if (isRecognizing.value) {
      await stopRecognizeSpeech((textSlice) => {
        message.value += textSlice || ''
      })
      onSubmit()
      console.log('submit', message.value)
      return
    }
    message.value = ''
    startRecognizeSpeech()
  }
  catch (error) {
    alert(error)
  }
}

const translate = async (text: string, i: number) => {
  translateIndex.value = i
  const key = text + i
  if (translates.value[key])
    return translates.value[key].isShow = !translates.value[key].isShow

  isTranslating.value = true
  const result = await generatTranslate({ text, translateKey: getAzureTranslateKey(), toLanguage: 'zh-Hans' })
  translates.value = {
    ...translates.value,
    [key]: { result, isShow: true },
  }
  isTranslating.value = false
}
</script>

<template>
  <div flex flex-col p-2 rounded-md bg-white dark="bg-#1e1e1e" shadow-sm>
    <div ref="el" class="hide-scrollbar flex-1 overflow-auto">
      <template v-if="chatMessages.length">
        <div
          v-for="item, i in chatMessages"
          :key="i"
          center-y odd:flex-row-reverse
        >
          <div class="w-10 h-10">
            <img w-full h-full object-fill rounded-full :src="item.role === 'user' ? selfAvatar : currentAvatar" alt="">
          </div>

          <div style="flex-basis:fit-content" mx-2>
            <p p-2 my-2 chat-box>
              {{ item.content }}
            </p>
            <p v-show="item.role === 'assistant' && translates[item.content + i]?.isShow " p-2 my-2 chat-box>
              {{ translates[item.content + i]?.result }}
            </p>

            <p v-if="item.role === 'assistant'" mt-2 flex>
              <template v-if="speakIndex !== i">
                <span class="chat-btn" @click="speak(item.content, i)">
                  <i icon-btn rotate-90 i-ic:sharp-wifi />
                </span>
              </template>
              <template v-else>
                <span v-if="isSynthesizing || isPlaying" class="chat-btn">
                  <i icon-btn rotate-90 i-svg-spinners:wifi-fade />
                </span>
                <span v-else class="chat-btn" @click="speak(item.content, i)">
                  <i icon-btn rotate-90 i-ic:sharp-wifi />
                </span>
              </template>
              <span v-if="!isTranslating || translateIndex !== i" ml-1 class="chat-btn" @click="translate(item.content, i)">
                <i icon-btn i-carbon:ibm-watson-language-translator />
              </span>
              <span v-else ml-1 class="chat-btn">
                <i icon-btn i-eos-icons:bubble-loading />
              </span>
            </p>
          </div>
        </div>
      </template>
      <template v-else>
        <div font-italic text-gray-500 center h-full>
          Haven't started the conversation yet, let's start now
        </div>
      </template>
    </div>

    <div class="flex h-10 w-[-webkit-fill-available] mt-1">
      <Button
        mr-1
        :disabled="isRecognizReadying"
        @click="recognize()"
      >
        <i v-if="isRecognizReadying" i-eos-icons:bubble-loading />
        <i v-else i-carbon:microphone />
      </Button>

      <div v-if="isRecognizing" class="loading-btn">
        识别中，请讲话
        <i icon-btn i-eos-icons:three-dots-loading />
      </div>
      <div v-else-if="isRecognizReadying" class="loading-btn">
        录音设备准备中
        <i icon-btn i-eos-icons:three-dots-loading />
      </div>
      <input
        v-else-if="!store.loading"
        v-model="message"
        type="text"
        placeholder="Type your message here..."
        input-box
        p-3 flex-1 @keyup.enter="onSubmit"
      >
      <div v-else class="loading-btn">
        AI Is Thinking
        <i icon-btn i-eos-icons:three-dots-loading />
      </div>
      <Button
        :disabled="store.loading"
        mx-1
        @click="onSubmit"
      >
        <i i-carbon:send-alt />
      </Button>
      <Button
        :disabled="store.loading"
        @click="store.cleanCurrentConversations()"
      >
        <i i-carbon:trash-can />
      </Button>
    </div>
  </div>
</template>
