<script setup lang="ts">
import { useRoute, useRouter } from 'vue-router'
import { ref, onMounted, nextTick, watch } from 'vue'
import { chat, feedback, getCharacter, getChat, newChat, regenerateChat } from '@/utils/api'
import CONFIG from '@/config'
import { useLogto } from '@logto/vue'
import { useChatStore } from '@/stores/chat'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { BookUser, SendHorizonal } from 'lucide-vue-next'
import { Loader2, ThumbsUp, ThumbsDown, RotateCcw, MessageCircleMore } from 'lucide-vue-next'
import { useToast } from '@/components/ui/toast/use-toast'
import { Avatar, AvatarFallback, AvatarImage } from '@/components/ui/avatar'
import { type UserResponse } from '@/types/response'
import { Dialog, DialogContent, DialogHeader, DialogTitle, DialogTrigger } from '@/components/ui/dialog'
import CreateCharacter from '@/components/create_character.vue'

const { toast } = useToast()
const router = useRouter()
const route = useRoute()
const { getAccessToken, getIdTokenClaims } = useLogto()

const chatStore = useChatStore()

const hid = ref('')
const ready = ref(false)
const userName = ref<string>()
const userId = ref<string>()

const inputText = ref('')
const disableInput = ref(false)
const inputEle = ref<any>()
const scroll = ref<any>()

const load = async () => {
  hid.value = ''
  ready.value = false
  userName.value = ''
  inputText.value = ''
  disableInput.value = false
  inputEle.value = null
  scroll.value = null

  const claims = await getIdTokenClaims()
  userName.value = claims!.username!!
  userId.value = claims!.sub
  if (route.name === 'NewChat') {
    const cid = (route.params['cid'] as string) || ''
    if (!cid) {
      await router.push({ name: 'Landing' })
    }
    const token = await getAccessToken(CONFIG.API.ENDPOINT)
    const newChatInfo = await newChat(token!, cid)
    const hid = newChatInfo.hid
    chatStore.addChat(hid, newChatInfo.cid, newChatInfo.logs)
    if (!chatStore.isCharacterExist(cid)) {
      const char = await getCharacter(token!, cid)
      char.id = cid
      chatStore.upsertCharacter(cid, char)
    }
    chatStore.appendHistory({
      hid,
      cid: newChatInfo.cid,
      name: chatStore.getCharacterFromChat(hid)['name'],
      logs: newChatInfo.logs
    })
    await router.push({ name: 'Chat', params: { hid } })
  }
  hid.value = (route.params['hid'] as string) || ''
  if (!chatStore.isChatExist(hid.value)) {
    const token = await getAccessToken(CONFIG.API.ENDPOINT)
    const theChat = await getChat(token!, hid.value)
    chatStore.addChat(hid.value, theChat.cid, theChat.logs, theChat.feedbacked ?? false)
    if (!chatStore.isCharacterExist(theChat.cid)) {
      const char = await getCharacter(token!, theChat.cid)
      char.id = theChat.cid
      chatStore.upsertCharacter(theChat.cid, char)
    }
  }
  ready.value = true
  await nextTick(() => {
    focus()
    keepBottom()
  })
}

onMounted(async () => {
  await load()
})

watch(route, async () => {
  await load()
})

const focus = () => {
  inputEle.value.$el.focus()
}

const keepBottom = () => {
  scroll.value!.scrollTop = scroll.value!.scrollHeight
}

const submit = async () => {
  inputText.value = inputText.value.trim()
  if (!inputText.value) {
    return
  }
  disableInput.value = true
  chatStore.addLog(hid.value, {
    from: 0,
    content: inputText.value
  })
  await nextTick(() => {
    keepBottom()
  })
  const token = await getAccessToken(CONFIG.API.ENDPOINT)
  const result = await chat(token!, hid.value, inputText.value)
  if (result.err) {
    toast({
      title: '嗯，有一些问题啊...',
      description: '不知道为什么? 请稍后再试一次',
      variant: 'destructive'
    })
    chatStore.popLog(hid.value)
    disableInput.value = false
    return
  }
  chatStore.addLog(hid.value, result.logs[0])

  inputText.value = ''
  disableInput.value = false
  chatStore.resetFeedback(hid.value)
  chatStore.setHistoryMsg(hid.value, result.logs[0])
  await nextTick(() => {
    focus()
    keepBottom()
  })
}

const regenerate = async () => {
  disableInput.value = true
  chatStore.popLog(hid.value)
  chatStore.resetFeedback(hid.value)
  const token = await getAccessToken(CONFIG.API.ENDPOINT)
  const result = await regenerateChat(token!, hid.value)
  if (result.err) {
    toast({
      title: '嗯，有一些问题啊...',
      description: '不知道为什么? 请稍后再试一次',
      variant: 'destructive'
    })
    disableInput.value = false
    return
  }
  chatStore.addLog(hid.value, result.logs[0])
  disableInput.value = false
  chatStore.setHistoryMsg(hid.value, result.logs[0])
  await nextTick(() => {
    focus()
    keepBottom()
  })
}

const feedBack = async (feed: boolean) => {
  const token = await getAccessToken(CONFIG.API.ENDPOINT)
  await feedback(token!, hid.value, feed)
  chatStore.setFeedback(hid.value)
}

const submitEnter = async (e: KeyboardEvent) => {
  if (!e.ctrlKey && e.key === 'Enter') {
    await submit()
  }
}
</script>

<template>
  <div class="flex flex-col h-screen justify-between" v-if="ready">
    <div class="flex flex-col gap-3 overflow-y-auto p-4" ref="scroll">
      <!-- Chat history -->
      <div class="flex flex-col items-center m-10">
        <div class="flex gap-3 items-center">
          <Avatar>
            <AvatarImage src="file://" />
            <AvatarFallback>{{ chatStore.getCharacterFromChat(hid).name[0] }}</AvatarFallback>
          </Avatar>
          <h1 class="text-2xl font-bold">{{ chatStore.getCharacterFromChat(hid).name }}</h1>
        </div>
        <div class="mt-2 text-center text-gray-600">
          <p class="text-sm mb-2">{{ chatStore.getCharacterFromChat(hid).description }}</p>
          <div class="flex gap-2 justify-center items-center text-sm">
            <RouterLink :to="{ name: 'User', params: { uid: chatStore.getCharacterFromChat(hid).creator.id }}">@{{ chatStore.getCharacterFromChat(hid).creator.nickname || (chatStore.getCharacterFromChat(hid).creator as UserResponse).name }}</RouterLink>
            <span class="flex items-center">
              <MessageCircleMore class="w-4 h-4" />{{ chatStore.getCharacterFromChat(hid).stat.chat_count }}
            </span>
            <span class="flex items-center">
              <ThumbsUp class="w-4 h-4" />{{ chatStore.getCharacterFromChat(hid).stat.liked }}
            </span>

            <Dialog v-if="chatStore.getCharacterFromChat(hid).creator.id == userId">
              <DialogTrigger as-child>
                <Button size="sm" variant="link">编辑</Button>
              </DialogTrigger>
              <DialogContent class="p-0">
                <DialogHeader class="pl-6 pr-6 pt-6">
                  <DialogTitle>编辑角色信息</DialogTitle>
                </DialogHeader>
                <div class="max-h-[calc(100vh-6.25rem)] pl-4 pr-4 pb-6 overflow-y-auto">
                  <CreateCharacter class="ml-2 mr-2" :preload="chatStore.getCharacterFromChat(hid)" />
                </div>
              </DialogContent>
            </Dialog>
          </div>
        </div>
      </div>


      <template v-for="(log, i) in chatStore.getLogs(hid)" :key="i">
        <div class="flex gap-3 justify-end" v-if="log.from === 0">
          <div class="flex flex-col items center gap-2">
            <span class="font-bold font-400 text-end">{{ userName }}</span>
            <div class="rounded-lg px-3 py-2 text-white bg-black w-fit">{{ log.content }}</div>
          </div>
          <Avatar>
            <AvatarImage src="files://" />
            <AvatarFallback>{{ userName![0] }}</AvatarFallback>
          </Avatar>
        </div>
        <div class="flex gap-3" v-else>
          <Avatar>
            <AvatarImage src="files://" />
            <AvatarFallback>{{ chatStore.getCharacterFromChat(hid)['name'][0] }}</AvatarFallback>
          </Avatar>
          <div class="flex flex-col items center gap-2">
            <span class="font-bold font-400">{{
              chatStore.getCharacterFromChat(hid)['name']
            }}</span>
            <div class="rounded-lg px-3 py-2 text-base bg-muted w-fit">{{ log.content }}</div>
            <div
              class="flex gap-1 justify-end items-center"
              v-if="i == chatStore.getLogs(hid).length - 1 && i > 1"
            >
              <template v-if="!chatStore.getFeedback(hid)">
                <ThumbsUp class="w-4 h-4 thumbs-up" @click="feedBack(true)" />
                <ThumbsDown class="w-4 h-4 thumbs-down" @click="feedBack(false)" />
              </template>
              <RotateCcw
                class="ml-1 w-4 h-4"
                @click="regenerate"
                :class="{ 'animate-spin': disableInput }"
              />
            </div>
          </div>
        </div>
      </template>

      <div class="flex gap-3" v-if="disableInput">
        <Avatar>
          <AvatarImage src="files://" />
          <AvatarFallback>{{ chatStore.getCharacterFromChat(hid)['name'][0] }}</AvatarFallback>
        </Avatar>
        <div class="flex flex-col items center gap-2">
          <span class="font-bold font-400">{{ chatStore.getCharacterFromChat(hid)['name'] }}</span>
          <div class="rounded-lg px-3 py-2 text-base bg-muted w-fit">
            <Loader2 v-if="disableInput" class="w-5 h-5 animate-spin" />
          </div>
        </div>
      </div>
    </div>
    <div class="flex w-full items-center gap-1.5 p-4 pt-1 relative">
      <p class="warning-inaccuracy text-sm text-gray-400 select-none">
        请注意：角色的对话均为捏造，请勿信以为真
      </p>
      <Input
        type="text"
        placeholder="说点什么..."
        :disabled="disableInput"
        v-model="inputText"
        @keydown="submitEnter"
        ref="inputEle"
      />
      <Button @click="submit" :disabled="disableInput || inputText.length < 1">
        <Loader2 v-if="disableInput" class="w-4 h-4 animate-spin mr-2" />
        <SendHorizonal v-else class="w-4 h-4 mr-2" />
        发送
      </Button>
    </div>
  </div>
</template>

<style scoped lang="scss">
.thumbs-up {
  &:hover {
    fill: cyan;
  }
  &:focus {
    fill: cyan;
  }
}

.thumbs-down {
  &:hover {
    fill: red;
  }
  &:focus {
    fill: red;
  }
}

.warning-inaccuracy {
  position: absolute;
  top: -22px;
  left: 0;
  right: 0;
  text-align: center;
  margin: auto;

}
</style>
