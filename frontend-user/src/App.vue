<template>
  <div class="app">
    <PdfViewer />
  </div>
</template>

<script setup lang="ts">
import PdfViewer from '@/components/PdfViewer.vue'
</script>

<style lang="scss">
.app {
  width: 100%;
  min-height: 100vh;
  background-color: var(--bg-color);
}
</style>
