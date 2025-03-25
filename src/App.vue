<template>
  <div class="app-container">
    <ProseMirrorEditor />
  </div>
</template>

<script setup>
import ProseMirrorEditor from './components/ProseMirrorEditor.vue'
</script>

<style>
.app-container {
  padding: 20px;
  max-width: 1200px;
  margin: 0 auto;
}
</style>
