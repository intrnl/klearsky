<script lang="ts" setup>
const yyyy = (new Date).getFullYear()
</script>

<template>
  <div class="copyright">
    <div class="kebab-header">Klearsky</div>
    <small>&copy; {{ yyyy }} mimonelu</small>
    <RouterLink
      class="textlink"
      :to="{ path: '/profile/post', query: { account: 'mimonelu.net' } }"
    >Bluesky@mimonelu.net</RouterLink>
    <a
      class="textlink"
      href="https://twitter.com/mimonelu"
      rel="noreferrer"
      target="_blank"
    >Twitter@mimonelu</a>
    <div />
    <div class="kebab-header">Bluesky</div>
    <small>&copy; {{ yyyy }} Bluesky PBLLC</small>
    <a
      class="textlink"
      href="https://blueskyweb.xyz/"
      rel="noreferrer"
      target="_blank"
    >Bluesky official web site</a>
    <a
      class="textlink"
      href="https://twitter.com/bluesky"
      rel="noreferrer"
      target="_blank"
    >Twitter@bluesky</a>
  </div>
</template>

<style lang="scss" scoped>
.copyright {
  display: flex;
  align-items: center;
  flex-direction: column;
  grid-gap: 1em;
}

.kebab-header {
  display: flex;
  align-items: center;
  justify-content: center;
  grid-gap: 0.5em;
  width: 100%;

  &::before,
  &::after {
    background-color: rgba(var(--fg-color), 0.125);
    content: "";
    display: block;
    flex-grow: 1;
    height: 1px;
  }
}
</style>
