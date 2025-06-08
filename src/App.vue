<script setup lang="ts">
import { ref,watch, onMounted } from 'vue'
import Altimeter from './devices/Altimeter.vue'
</script>

<template>
  <h1>Neon</h1>
    <p>Welcome to the Neon app!</p>
    <p>Explore the features and enjoy your stay.</p>
  <Altimeter />
</template>
