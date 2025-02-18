<script lang="ts" setup>
import { inject, onBeforeUnmount, onMounted, reactive, ref } from "vue"
import GloballineSettingsPopup from "@/components/GloballineSettingsPopup.vue"
import Loader from "@/components/Loader.vue"
import Post from "@/components/Post.vue"
import SubscribeRepos from "@/composables/atp-wrapper/atp-utils/subscribe-repos"
import SVGIcon from "@/components/SVGIcon.vue"
import Util from "@/composables/util"

const mainState = inject("state") as MainState

const state = reactive<{
  subscriber?: SubscribeRepos;
  globallineSettingsPopupDisplay: boolean;
}>({
  subscriber: undefined,
  globallineSettingsPopupDisplay: false,
})

onMounted(() => {
  startControlToScroll()
  connect()
})

onBeforeUnmount(() => {
  endControlToScroll()
  disconnect()
  destroyProfileTimer()
})

// subscribeRepo

function connect () {
  const domain = mainState.atp.session?.__service?.replace(/^\w+:\/+/, "") ?? ""
  state.subscriber = new SubscribeRepos(onError, undefined, undefined, undefined, onPost)
  state.subscriber.connect(`wss://${domain}/xrpc/com.atproto.sync.subscribeRepos`)
  createProfileTimer()
}

function disconnect () {
  state.subscriber?.disconnect()
}

function onError () {
  disconnect()
}

async function onPost (did: string, post: any) {
  mainState.globallineNumberOfPosts ++

  messageContainerHeight = (messageContainer.value as any).clientHeight

  // 言語解析
  if (post.record.text != null) {
    const languages = post.__custom.detectedLanguages
    if (languages != null) {
      const yourLanguage = languages.findIndex((language: any) => {
        return language.lang === mainState.currentSetting.globallineLanguage
      }) !== - 1
      if (!yourLanguage) return
    }
  }

  if (mainState.globallineProfiles[did] == null)
    mainState.globallineProfiles[did] = {}
  post.author = mainState.globallineProfiles[did]
  mainState.globallinePosts.unshift(post)
}

function toggleConnect () {
  Util.blurElement()
  if (state.subscriber?.socketState === 0) connect()
  else if (state.subscriber?.socketState === 2) disconnect()
}

// subscribeRepo - プロフィール取得

let timer: undefined | NodeJS.Timer = undefined

function createProfileTimer () {
  timer = setTimeout(async () => {
    try {
      for (const did in mainState.globallineProfiles) {
        const profile = mainState.globallineProfiles[did]
        if (profile.handle != null) continue
        const newProfile = await mainState.atp.fetchProfile(did)
        for (const key in newProfile) {
          mainState.globallineProfiles[did][key] = newProfile[key]
        }
        break
      }
    } finally {
      createProfileTimer()
    }
  }, 1000)
}

function destroyProfileTimer () {
  clearInterval(timer)
  timer = undefined
}

// ポストアクション

function updateThisPostThread (newPosts: Array<TTPost>) {
  mainState.globallinePosts.forEach((post: TTPost, index: number) => {
    const newPost = newPosts.find((newPost: TTPost) => post.cid === newPost.cid)
    if (newPost != null) Util.replacePost(mainState.globallinePosts[index], newPost)
  })
}

function removeThisPost (uri: string) {
  mainState.globallinePosts = mainState.globallinePosts
    .filter((post: TTPost) => {
      return post.uri !== uri
    })
}

// グローバルライン設定ポップアップ

function openGloballineSettingsPopup () {
  Util.blurElement()
  state.globallineSettingsPopupDisplay = true
}

function closeGloballineSettingsPopup () {
  state.globallineSettingsPopupDisplay = false
}

// スクロール制御

let mutationObserver: undefined | MutationObserver = undefined
const messageContainer = ref(null)
let messageContainerHeight = 0

function startControlToScroll () {
  mutationObserver = new MutationObserver(onMutated)
  mutationObserver.observe((messageContainer.value as unknown as HTMLElement), {
    childList: true,
    characterData: false,
    characterDataOldValue: false,
    attributes: true,
    subtree: false,
  })
}

function endControlToScroll () {
  mutationObserver?.disconnect()
}

function onMutated () {
  const scrollTop = window.pageYOffset || document.documentElement.scrollTop
  if (scrollTop == 0) return
  const heightDiff = (messageContainer.value as unknown as HTMLElement).clientHeight - messageContainerHeight
  window.scrollTo({
    left: 0,
    top: scrollTop + heightDiff
  })
}
</script>

<template>
  <div class="globalline-view">
    <div
      class="message-container"
      ref="messageContainer"
      >
      <div
        v-for="message of mainState.globallinePosts"
        :key="message.cid"
        class="message-wrapper"
        :data-is-reply="message.record.reply != null"
        :data-is-quote-repost="message.record.embed?.record != null"
        :data-is-loaded="message.author.did != null"
        :data-is-muted="message.author.viewer?.muted"
        :data-is-blocking="message.author.viewer?.blocking != null"
      >
        <!-- ポスト -->
        <Post
          :position="mainState.currentSetting.globallineLayout ?? 'post'"
          :post="message"
          :forceHideImages="true"
          @updateThisPostThread="updateThisPostThread"
          @removeThisPost="removeThisPost"
        >
          <template v-slot:body-after>
            <!-- リプライ／引用リポストアイコン -->
            <div
              v-if="message.record.reply != null"
              class="reply-icon"
            >
              <SVGIcon name="post" />
            </div>
            <div
              v-if="message.record.embed?.record != null"
              class="quote-repost-icon"
            >
              <SVGIcon name="quoteRepost" />
            </div>
          </template>
        </Post>
      </div>
    </div>

    <!-- グローバルラインヘッダー -->
    <Portal to="home-view-header-portal">
      <div class="globalline-footer">
        <!-- 情報 -->
        <div class="info">
          <dl>
            <dt>
              <SVGIcon name="post" />
            </dt>
            <dd>{{ mainState.globallinePosts.length.toLocaleString() }} / {{ mainState.globallineNumberOfPosts.toLocaleString() }}</dd>
          </dl>
        </div>

        <!-- 電源ボタン -->
        <button
          :class="state.subscriber?.socketState === 2 ? 'button--important' : 'button--bordered'"
          class="power-button"
          @click.stop="toggleConnect"
        >
          <template v-if="state.subscriber?.socketState === 0">
            <SVGIcon name="play" />
          </template>
          <template v-else-if="state.subscriber?.socketState === 1">
            <SVGIcon name="menu" />
            <Loader />
          </template>
          <template v-else-if="state.subscriber?.socketState === 2">
            <SVGIcon name="pause" />
          </template>
        </button>

        <!-- グローバルライン設定ポップアップトリガー -->
        <button
          class="button--bordered globalline-settings-popup-button"
          @click.stop="openGloballineSettingsPopup"
        >
          <SVGIcon name="setting" />
        </button>
      </div>
    </Portal>

    <!-- グローバルライン設定ポップアップ -->
    <GloballineSettingsPopup
      v-if="state.globallineSettingsPopupDisplay"
      @close="closeGloballineSettingsPopup"
    />
  </div>
</template>

<style lang="scss" scoped>
.globalline-view {
  display: flex;
  flex-direction: column;
  flex-grow: 1;
}

.message-container {
  display: flex;
  flex-direction: column;
  flex-grow: 1;
  padding: 0.5em 0 var(--sp-menu-height);
}

.message-wrapper {
  position: relative;

  &[data-is-loaded="false"] {
    opacity: 0.5;
  }
  &[data-is-muted="true"],
  &[data-is-blocking="true"] {
    display: none;
  }
}

// ポスト
.post {
  padding: 0.5rem 1rem;
}

// リプライ／引用リポストアイコン
.reply-icon,
.quote-repost-icon {
  position: absolute;
  top: -0.5rem;
  left: -0.5rem;

  & > .svg-icon {
    font-size: 1.25rem;
    transform: scaleX(- 1.0);
  }
}
.reply-icon {
  & > .svg-icon {
    fill: rgb(var(--post-color));
    stroke: rgb(var(--bg-color));
  }
}
.quote-repost-icon {
  & > .svg-icon {
    fill: rgb(var(--share-color));
    stroke: rgb(var(--bg-color));
  }
}

// グローバルラインヘッダー
.globalline-footer {
  display: flex;
  align-items: center;
  grid-gap: 0.75rem;

  // 電源ボタン
  .power-button {
    position: relative;
    min-width: 4rem;

    & > .svg-icon--menu {
      fill: transparent;
    }

    & > .loader {
      font-size: 0.5rem;
    }
  }

  // 情報
  & > .info {
    display: flex;
    flex-grow: 1;
    align-content: center;
    justify-content: flex-end;
    grid-gap: 0.5rem;

    & > dl {
      display: flex;
      align-items: center;
      grid-gap: 0.5rem;

      & > dt {
        & > .svg-icon {
          fill: rgba(var(--fg-color), 0.5);
          font-size: 0.875rem;
        }
      }

      & > dd {
        color: rgba(var(--fg-color), 0.75);
        font-family: monospace;
        line-height: 1.25;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
      }
    }
  }

  button {
    margin: -1rem 0;
  }
}
</style>
