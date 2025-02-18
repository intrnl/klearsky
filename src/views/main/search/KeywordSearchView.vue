<script lang="ts" setup>
import { inject, onMounted, reactive, watch } from "vue"
import { useRouter } from "vue-router"
import Post from "@/components/Post.vue"
import Util from "@/composables/util"

const mainState = inject("state") as MainState

const state = reactive<{
  text: string
}>({
  text: "",
})

const router = useRouter()

watch(() => router.currentRoute.value.query.text, (value: any) => {
  updateSearchKeywordTerm(value)
}, { immediate: true })

onMounted(() => {
  const formItem = document.getElementById("keyword-term-textbox")
  if (formItem != null) formItem.focus()
})

function updateSearchKeywordTerm (text: string) {
  state.text = text
  if (!text || mainState.currentSearchKeywordTerm === text) return
  mainState.currentSearchKeywordTerm = text
  fetchNewResults()
}

function submitForm () {
  router.push({ name: "keyword-search", query: { text: state.text } })
}

async function fetchNewResults () {
  if (mainState.processing) return
  if (state.text === "") return
  mainState.currentSearchKeywordResults.splice(0)
  mainState.processing = true
  try {
    const results: undefined | false | Array<TTPost> =
      await mainState.atp.fetchKeywordSearch(state.text)
    if (results === false) return
    if (results != null) mainState.currentSearchKeywordResults = results
  } finally {
    mainState.processing = false
  }
}

function updateThisPostThread (newPosts: Array<TTPost>) {
  const posts = mainState.currentSearchKeywordResults
  posts.forEach((oldPost: TTPost, index: number) => {
    const newPost = newPosts.find((newPost: TTPost) => oldPost.cid === newPost.cid)
    if (newPost != null) Util.replacePost(posts[index], newPost)
  })
}

function removeThisPost (uri: string) {
  mainState.currentSearchKeywordResults = mainState.currentSearchKeywordResults
    .filter((post: TTPost) => post.uri !== uri)
}
</script>

<template>
  <div class="keyword-search-view">
    <form @submit.prevent="submitForm">
      <input
        v-model="state.text"
        id="keyword-term-textbox"
        type="search"
        :placeholder="$t('searchWord')"
        autocapitalize="off"
        autocomplete="off"
        inputmode="search"
        spellcheck="false"
        class="textbox"
      >
    </form>
    <div class="main">
      <Post
        v-for="post of mainState.currentSearchKeywordResults"
        :key="post.cid"
        :position="post.__custom.forcePosition != null ? post.__custom.forcePosition as any : 'post'"
        :post="post"
        @updateThisPostThread="updateThisPostThread"
        @removeThisPost="removeThisPost"
        @onActivateHashTag="updateSearchKeywordTerm"
      />
    </div>
  </div>
</template>

<style lang="scss" scoped>
.keyword-search-view {
  padding-bottom: var(--sp-menu-height);

  form {
    border-bottom: 1px solid rgba(var(--fg-color), 0.25);
    display: grid;
    padding: 1rem;
  }
}

.main {
  display: flex;
  flex-direction: column;
  flex-grow: 1;
}

.post {
  border-bottom: 1px solid rgba(var(--fg-color), 0.125);
  &[data-position="preview"] {
    padding: 1em;
  }
}
</style>
