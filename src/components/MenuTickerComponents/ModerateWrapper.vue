<script lang="ts" setup>
import { reactive } from "vue"
import MenuTicker from "@/components/MenuTicker.vue"
import MenuTickerSendAccountReport from "@/components/MenuTickerComponents/SendAccountReport.vue"
import MenuTickerSendPostReport from "@/components/MenuTickerComponents/SendPostReport.vue"
import MenuTickerToggleBlock from "@/components/MenuTickerComponents/ToggleBlock.vue"
import MenuTickerToggleMute from "@/components/MenuTickerComponents/ToggleMute.vue"
import SVGIcon from "@/components/SVGIcon.vue"

const emit = defineEmits<{(event: string): void}>()

defineProps<{
  isUser?: boolean;
  user?: TTUser;
  post?: TTPost;
}>()

const state = reactive<{
  display: boolean;
}>({
  display: false,
})
</script>

<template>
  <button
    class="menu-ticker__sub-trigger"
    @click.prevent.stop
    @mouseenter="state.display = true"
    @mouseleave="state.display = false"
  >
    <SVGIcon name="cursorLeft" />
    <span>{{ $t("moderate") }}</span>

    <!-- モデレーションメニュー -->
    <MenuTicker
      :display="state.display"
      class="menu-ticker__sub"
    >
      <!-- ミュートのトグル -->
      <MenuTickerToggleMute
        v-if="!isUser"
        :user="user"
        @close="emit('close')"
      />

      <!-- ブロックのトグル -->
      <MenuTickerToggleBlock
        v-if="!isUser"
        :user="user"
        @close="emit('close')"
      />

      <!-- アカウントレポート送信ポップアップを開く -->
      <MenuTickerSendAccountReport
        v-if="!isUser && user != null"
        :user="user"
        @close="emit('close')"
      />

      <!-- ポストレポート送信ポップアップを開く -->
      <MenuTickerSendPostReport
        v-if="!isUser && post != null"
        :post="post"
        @close="emit('close')"
      />
    </MenuTicker>
  </button>
</template>
