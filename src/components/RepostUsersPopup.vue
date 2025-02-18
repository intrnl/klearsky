<script lang="ts" setup>
import { inject, onBeforeMount, reactive } from "vue"
import LoadButton from "@/components/LoadButton.vue"
import Popup from "@/components/Popup.vue"
import SVGIcon from "@/components/SVGIcon.vue"
import UserBox from "@/components/UserBox.vue"
import Util from "@/composables/util"
import consts from "@/consts/consts.json"

const emit = defineEmits<{(event: string): void}>()

const mainState = inject("state") as MainState

const state = reactive<{
  processing: boolean
}>({
  processing: false
})

onBeforeMount(async () => {
  await fetchContinuousResults("new")
})

function close () {
  emit("close")
}

async function fetchContinuousResults (direction: "new" | "old") {
  Util.blurElement()
  if (state.processing) return
  state.processing = true
  try {
    const cursor: undefined | string = await mainState.atp.fetchRepostUsers(
      mainState.currentRepostUsers as Array<TTUser>,
      mainState.currentRepostUsersUri as string,
      consts.limitOfFetchRepostUsers,
      direction === "old" ? mainState.currentRepostUsersCursor : undefined
    )
    if (cursor != null) mainState.currentRepostUsersCursor = cursor
  } finally {
    state.processing = false
  }
}
</script>

<template>
  <Popup
    class="repost-users-popup"
    :hasCloseButton="true"
    @close="close"
  >
    <template v-slot:header>
      <h2>
        <SVGIcon name="repost" />
        <span>{{ $t("repostUsers") }}</span>
      </h2>
    </template>
    <template v-slot:body>
      <div class="users">
        <UserBox
          v-for="user of mainState.currentRepostUsers"
          :key="user.did"
          class="user"
          :user="user"
          :contentWarningDisabled="false"
        />
      </div>
    </template>
    <template v-slot:footer>
      <LoadButton
        direction="old"
        :processing="state.processing"
        @activate="fetchContinuousResults('old')"
      />
    </template>
  </Popup>
</template>

<style lang="scss" scoped>
.repost-users-popup:deep() {
  .popup {
    &-header {
      & > h2 {
        color: rgb(var(--share-color));

        & > .svg-icon {
          fill: rgb(var(--share-color));
        }
      }
    }

    &-body > .users {
      display: flex;
      flex-direction: column;
      grid-gap: 1rem;

      & > .user-box {
        cursor: pointer;
      }
    }
  }
}
</style>
