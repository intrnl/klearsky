<script lang="ts" setup>
import { inject, onBeforeMount, onBeforeUnmount, reactive } from "vue"
import hotkeys from "hotkeys-js"
import SVGIcon from "@/components/SVGIcon.vue"
import Util from "@/composables/util"

const emit = defineEmits<{(event: string): void}>()

const props = defineProps<{
  images: Array<{
    largeUri: string
    smallUri: string
  }>
  index: number
}>()

const mainState = inject("state") as MainState

const state = reactive<{
  isBlob: boolean;
  loaded: boolean;
  mode: boolean;
  x: number;
  y: number;
}>({
  isBlob: props.images[props.index].largeUri.startsWith("blob:"),
  loaded: false,
  mode: false,
  x: 0.5,
  y: 0.5,
})

onBeforeMount(() => {
  hotkeys("esc", close)
})

onBeforeUnmount(() => {
  hotkeys.unbind("esc")
})

function startDrag (event: MouseEvent | TouchEvent) {
  const e = (event as TouchEvent).touches != null
    ? (event as TouchEvent).touches[0]
    : event as MouseEvent

  // 右クリックはスルー
  if ((e as MouseEvent).button != null && (e as MouseEvent).button !== 0) return

  state.mode = true
  state.x = (e.clientX / window.innerWidth)
  state.y = (e.clientY / window.innerHeight)
  // TODO: やや危険、別の方法を模索すること
  window.document.body.style.overflowX = "hidden"
  window.document.body.style.overflowY = "hidden"
}

function moveDrag (event: MouseEvent | TouchEvent) {
  if (!state.mode || (
    (event as MouseEvent).buttons != null &&
    (event as MouseEvent).buttons !== 1)
  ) return
  const e = (event as TouchEvent).touches != null
    ? (event as TouchEvent).touches[0]
    : event as MouseEvent
  state.x = (e.clientX / window.innerWidth)
  state.y = (e.clientY / window.innerHeight)
}

function endDrag () {
  state.x = 0.5
  state.y = 0.5
  state.mode = false
  // TODO: やや危険、別の方法を模索すること
  window.document.body.style.overflowX = "hidden"
  window.document.body.style.overflowY = "scroll"
}

function onLoadLargeImage () {
  state.loaded = true
}

function showImage (indexAdding: number) {
  mainState.imagePopupProps.index = props.index + indexAdding
}

function setBackgroundImage () {
  Util.blurElement()
  if (!state.loaded) return
  if (mainState.currentSetting == null) return
  mainState.currentSetting.backgroundImage = props.images[props.index].largeUri
  mainState.saveSettings()
}

function blurElement () {
  Util.blurElement()
}

function close () {
  Util.blurElement()
  emit("close")
}
</script>

<template>
  <div
    class="image-popup"
    :data-loaded="state.loaded"
    :data-mode="state.mode"
  >
    <!-- サムネイル画像 -->
    <div
      class="image"
      :style="`
        background-image: url(${state.loaded ? '' : images[index].smallUri});
        background-position: ${state.x * 100}% ${state.y * 100}%;
      `"
      @mousedown="startDrag"
      @touchstart.passive="startDrag"
      @mousemove="moveDrag"
      @touchmove.passive="moveDrag"
      @mouseup="endDrag"
      @touchend.passive="endDrag"
    >
      <!-- ラージ画像 -->
      <div
        class="image"
        :style="`
          background-image: url(${images[index].largeUri});
          background-position: ${state.x * 100}% ${state.y * 100}%;
        `"
      />
    </div>

    <!-- ラージ画像の読込判断用 img 要素 -->
    <img
      class="large-image-loader"
      :src="images[index].largeUri"
      alt=""
      @load="onLoadLargeImage"
    >

    <!-- 前の画像ボタン -->
    <button
      class="floating-button previous-image-button"
      :disabled="index === 0"
      @click.prevent="showImage(- 1)"
    >
      <SVGIcon name="cursorLeft" />
    </button>

    <!-- 次の画像ボタン -->
    <button
      class="floating-button next-image-button"
      :disabled="index + 1 === images.length"
      @click.prevent="showImage(1)"
    >
      <SVGIcon name="cursorRight" />
    </button>

    <!-- 壁紙設定ボタン -->
    <button
      v-if="!state.isBlob"
      class="floating-button background-image-button"
      @click.prevent="setBackgroundImage"
    >
      <SVGIcon name="wallpaper" />
    </button>

    <!-- 画像を別タブで開くボタン -->
    <a
      v-if="!state.isBlob"
      class="floating-button open-image-button"
      :href="images[index].largeUri"
      rel="noreferrer"
      target="_blank"
      @click="blurElement"
    >
      <SVGIcon name="openInNew" />
    </a>

    <!-- 閉じるボタン -->
    <button
      class="floating-button close-button"
      @click.prevent="close"
    >
      <SVGIcon name="cross" />
    </button>
  </div>
</template>

<style lang="scss" scoped>
.image-popup {
  background-color: rgba(0, 0, 0, 0.75);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  overscroll-behavior: none;
  position: fixed;
  left: 0;
  top: 0;
  z-index: 2;
  width: 100%;
  height: 100%;
  &[data-mode="false"] {
    padding: 1rem;
  }
}

// サムネイル画像 / ラージ画像
.image {
  background-position: 50% 50%;
  background-repeat: no-repeat;
  cursor: grabbing;
  overscroll-behavior: none;
  position: relative;
  width: 100%;
  height: 100%;
  transition: background-position 100ms ease-out;
  [data-mode="false"] & {
    background-size: contain;
    cursor: grab;
  }
}

// ラージ画像の読込判断用 img 要素
.large-image-loader {
  display: contents;
}

// 壁紙設定ボタン / 画像を別タブで開くボタン / 閉じるボタン
.floating-button {
  background-color: rgba(0, 0, 0, 0.25);
  border-radius: var(--border-radius);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  position: absolute;
  width: 4rem;
  min-height: 4rem;
  &[disabled] {
    opacity: 0.25;
  }
  [data-mode="true"] & {
    display: none;
  }

  & > .svg-icon {
    fill: rgba(255, 255, 255, 0.75);
    font-size: 2rem;
  }

  &:focus, &:hover {
    background-color: rgba(0, 0, 0, 0.5);

    & > .svg-icon {
      fill: rgb(255, 255, 255);
    }
  }
}

// 前の画像ボタン
.previous-image-button {
  // SP幅以上
  @media (min-width: $sp-width) {
    left: 1rem;
  }

  // SP幅未満
  @media not all and (min-width: $sp-width) {
    bottom: calc(1rem + env(safe-area-inset-bottom));
    left: 1rem;
  }
}

// 次の画像ボタン
.next-image-button {
  // SP幅以上
  @media (min-width: $sp-width) {
    right: 1rem;
  }

  // SP幅未満
  @media not all and (min-width: $sp-width) {
    bottom: calc(1rem + env(safe-area-inset-bottom));
    left: 6rem;
  }
}

// 壁紙設定ボタン / 画像を別タブで開くボタン
.background-image-button,
.open-image-button {
  [data-loaded="false"] & {
    & > .svg-icon {
      fill: rgba(255, 255, 255, 0.25);
    }
  }
}

// 壁紙設定ボタン
.background-image-button {
  top: 1rem;
  left: 1rem;
}

// 画像を別タブで開くボタン
.open-image-button {
  top: 1rem;
  left: 6rem;
}

// 閉じるボタン
.close-button {
  // SP幅以上
  @media (min-width: $sp-width) {
    top: 1rem;
    right: 1rem;
  }

  // SP幅未満
  @media not all and (min-width: $sp-width) {
    bottom: calc(1rem + env(safe-area-inset-bottom));
    right: 1rem;
  }
}
</style>
