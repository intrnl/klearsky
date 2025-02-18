<script lang="ts" setup>
import { inject } from "vue"
import Loader from "@/components/Loader.vue"
import PageHeader from "@/components/PageHeader.vue"
import Post from "@/components/Post.vue"
import Util from "@/composables/util"

const mainState = inject("state") as MainState

function updateThisPostThread (newPosts: Array<TTPost>) {
  const posts = mainState.currentPosts
  if (posts == null) return

  // MEMO: ポストスレッドの全同一ポストに最新のデータを反映する
  posts.forEach((post: TTPost, index: number) => {
    const newPost = newPosts.find((newPost: TTPost) => post?.cid === newPost.cid)
    if (newPost != null) Util.replacePost(posts[index], newPost)
  })
}

function removeThisPost (uri: string) {
  mainState.currentPosts = mainState.currentPosts.filter((post: TTPost) => post.uri !== uri)
}
</script>

<template>
  <div class="post-view">
    <PageHeader
      :hasBackButton="true"
      :title="$t('post')"
      :subTitle="mainState.currentPosts[0] != null ? mainState.currentPosts[0].author.displayName : ''"
    />
    <Post
      v-for="post, postIndex of mainState.currentPosts"
      :key="post.cid"
      position="post"
      :post="post"
      :data-has-child="post.cid === mainState.currentPosts[postIndex + 1]?.record.reply?.parent?.cid"
      @updateThisPostThread="updateThisPostThread"
      @removeThisPost="removeThisPost"
    />
    <Loader v-if="mainState.listProcessing" />
  </div>
</template>

<style lang="scss" scoped>
.post-view {
  display: flex;
  flex-direction: column;
  flex-grow: 1;
  padding-bottom: var(--sp-menu-height);
  position: relative;
}

.post[data-focus="true"] {
  scroll-margin: 3.25rem;
}
</style>
