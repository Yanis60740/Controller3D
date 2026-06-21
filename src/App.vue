<template>
  <main>
    <View3D />
  </main>
</template>

<script setup>
import View3D from './components/View3D.vue'
</script>

<style>
/* Reset de base pour ne pas avoir de marges blanches autour de la 3D */
body {
  margin: 0;
  padding: 0;
  background-color: #1a1a1a;
}
</style>