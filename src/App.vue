<script setup>
import { RouterLink, RouterView } from 'vue-router'
import HelloWorld from './components/HelloWorld.vue'
import AgoraAudioCall from './components/AgoraAudioCall.vue';
</script>

<template>
  <header>
    Bismillah
    <AgoraAudioCall/>
  </header>

  <RouterView />
</template>


