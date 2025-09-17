<script setup lang="ts">
  import Search from './components/Search.vue'
</script>

<template>
  <div>
    <img src="/wowv-logo.jpg" class="logo" alt="WOW Vegas" />
  </div>
  <Search msg="Hourly Leaderboard Result Search" />
</template>

<style scoped>
.logo {
  height: 6em;
  padding: 1.5em;
  filter: drop-shadow(0 0 2em #646cffaa);
}
</style>
