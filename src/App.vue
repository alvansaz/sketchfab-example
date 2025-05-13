<script setup>
import SketchfabViewer from './components/SketchfabViewer.vue'
</script>

<template>

    <main>
        <SketchfabViewer
          modelId="bdf40cfa5c4b466c9ccaf3869f64a3a8"
          :autostart="true"
          :transparent="false"
        />
    </main>
</template>

<style scoped>
main {
  height: 100dvh;
}
</style>
