<script setup lang="ts">
import { Separator } from '@/components/ui/separator'
import { Home, User2, Search } from 'lucide-vue-next'
import { navigationMenuTriggerStyle } from '@/components/ui/navigation-menu'
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
  DialogClose
} from '@/components/ui/dialog'
import Login from '@/components/login.vue'
import { Button } from '@/components/ui/button'
import { useSharedState } from '@/stores/shared'
import { useLogto } from '@logto/vue'
import CONFIG from '@/config'
import { getChatHistory } from '@/utils/api'
import { onMounted, ref } from 'vue'
import { Avatar, AvatarFallback, AvatarImage } from '@/components/ui/avatar'
import { useChatStore } from '@/stores/chat'
import { useRoute, useRouter } from 'vue-router'

const router = useRouter()
const route = useRoute()
const sharedState = useSharedState()
const chatStore = useChatStore()

const { getAccessToken, getIdTokenClaims } = useLogto()

const loginShow = ref(false)

const getHistoryChats = async () => {
  const token = await getAccessToken(CONFIG.API.ENDPOINT)
  const hist = await getChatHistory(token!)
  chatStore.setHistory(hist)
}

const jumpMe = async () => {
  const claims = await getIdTokenClaims()
  if (route.name === 'User' && route.params['uid'] === claims!.sub) {
    return
  }
  await router.push({ name: 'User', params: { uid: claims!.sub }})
  loginShow.value = false
}

const jumpEdit = async () => {
  // not implemented
}

onMounted(async () => {
  await getHistoryChats()
})
</script>

<template>
  <aside class="w-64 p-2 shrink-0 dark:border-r-zinc-800 border-r-2 flex flex-col justify-between">
    <nav class="space-y-2 mt-1">
      <div class="flex items-center mb-3">
        <!--<img src="https://placehold.co/40x40" alt="Logo" class="mr-2" />-->
        <span class="text-xl font-bold"> {{ sharedState.getTitle }} </span>
        <span class="text-xs text-gray-500 ml-2">测试版本</span>
      </div>
      <Separator />
      <RouterLink
        class="flex items-center p-2 pl-3 transition-all rounded hover:bg-zinc-100 dark:hover:bg-zinc-800"
        :to="{ name: 'Landing' }"
        ><Home class="w-4 h-4 mr-2" />主页</RouterLink
      >
      <RouterLink
        class="flex items-center p-2 pl-3 transition-all rounded hover:bg-zinc-100 dark:hover:bg-zinc-800"
        :to="{ name: 'Search' }"
        ><Search class="w-4 h-4 mr-2" />搜索</RouterLink
      >
      <Separator />
    </nav>
    <div class="grow overflow-y-auto space-y-2 mb-2 mt-2">
      <RouterLink
        class="flex items-center p-2 transition-all rounded hover:bg-zinc-100 dark:hover:bg-zinc-800"
        :to="'/chat/' + data.hid"
        v-for="(data, i) in chatStore.getHistory"
        :key="i"
      >
        <Avatar class="w-8 h-8 mr-1">
          <AvatarImage src="files://" />
          <AvatarFallback>{{ data.name[0] }}</AvatarFallback>
        </Avatar>
        <span class="text-sm whitespace-nowrap">{{ data.name }}</span>
        <p class="text-sm text-gray-600 whitespace-nowrap overflow-hidden text-ellipsis">
          : {{ data.logs[0].content }}
        </p>
      </RouterLink>
    </div>
    <nav class="space-y-2 mb-3">
      <div class="flex items-center justify-center">
        <Dialog v-model:open="loginShow">
          <DialogTrigger as-child>
            <Button variant="link" size="sm" :class="navigationMenuTriggerStyle()">
              <User2 class="w-4 h-4 mr-2" />
              我的
            </Button>
          </DialogTrigger>
          <DialogContent>
            <DialogHeader>
              <DialogTitle>用户信息</DialogTitle>
            </DialogHeader>
            <div class="flex flex-col gap-5">
              <Login />
              <div class="flex gap-2">
                <Button size="xs" variant="outline" @click="jumpMe">查看</Button>
                <Button size="xs" variant="secondary" @click="jumpEdit">编辑</Button>
              </div>
            </div>
          </DialogContent>
        </Dialog>
      </div>
    </nav>
  </aside>
</template>

<style scoped lang="scss">
.text-white {
  p {
    color: white;
  }
}
</style>
