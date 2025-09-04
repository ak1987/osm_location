<script setup lang="ts">
import AppPage from '@/components/AppPage.vue';
import LocationMap from '@/components/LocationMap.vue';
</script>

<template>
  <AppPage title="Home" :back="false">
    <LocationMap />
  </AppPage>
  
</template>

<style scoped>
</style>