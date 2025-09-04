<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref } from 'vue';
import AppPage from '@/components/AppPage.vue';
import { locationManager } from '@telegram-apps/sdk';
import L from 'leaflet';

type CancelablePromise<T> = Promise<T> & { cancel?: () => void };

const mapContainer = ref<HTMLDivElement | null>(null);
let map: L.Map | null = null;
let marker: L.Marker | null = null;
let requestPromise: CancelablePromise<{
  latitude: number;
  longitude: number;
  altitude?: number;
  course?: number;
  speed?: number;
  horizontal_accuracy?: number;
  vertical_accuracy?: number;
  course_accuracy?: number;
  speed_accuracy?: number;
}> | null = null;

const isSupported = locationManager.isSupported();
const isMounting = ref(false);
const isMounted = ref(false);
const mountError = ref<unknown>(null);
const lastError = ref<string | null>(null);

function initializeMap(lat: number, lng: number) {
  if (!map && mapContainer.value) {
    map = L.map(mapContainer.value).setView([lat, lng], 15);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      maxZoom: 19,
      attribution: '© OpenStreetMap contributors',
    }).addTo(map);
  }
  if (map) map.setView([lat, lng], 15);
  if (!marker && map) marker = L.marker([lat, lng]).addTo(map);
  else if (marker) marker.setLatLng([lat, lng]);
}

async function mountLM() {
  if (!isSupported || !locationManager.mount.isAvailable()) return;
  isMounting.value = true;
  try {
    await locationManager.mount();
    isMounted.value = locationManager.isMounted();
    mountError.value = null;
  } catch (e) {
    mountError.value = e;
    isMounted.value = false;
  } finally {
    isMounting.value = false;
  }
}

async function requestLocationOnce() {
  lastError.value = null;
  if (!isSupported || !locationManager.requestLocation.isAvailable()) {
    lastError.value = 'LocationManager is not available.';
    return;
  }
  try {
    requestPromise?.cancel?.();
    requestPromise = locationManager.requestLocation();
    const loc = await requestPromise;
    initializeMap(loc.latitude, loc.longitude);
  } catch (e: any) {
    if (e?.name === 'CanceledError') return;
    lastError.value = String(e?.message || e);
  }
}

function openSettings() {
  if (locationManager.openSettings.isAvailable()) locationManager.openSettings();
}

onMounted(async () => {
  await mountLM();
});

onBeforeUnmount(() => {
  requestPromise?.cancel?.();
  locationManager.unmount?.();
  if (map) {
    map.remove();
    map = null;
    marker = null;
  }
});
</script>

<template>
  <AppPage title="Location Map">
    <div v-if="!isSupported">Location Manager is not supported in this environment.</div>
    <div v-else>
      <div style="display:flex;gap:8px;margin-bottom:8px;">
        <button @click="requestLocationOnce">Request Location</button>
        <button @click="openSettings">Open Settings</button>
      </div>
      <div v-if="isMounting">Mounting location manager…</div>
      <div v-if="mountError" style="color: var(--tgui-text-color-destructive);">{{ String(mountError) }}</div>
      <div v-if="lastError" style="color: var(--tgui-text-color-destructive);">{{ lastError }}</div>
      <div ref="mapContainer" style="height:70vh;width:100%;border-radius:12px;overflow:hidden;background:#e9ecef;" />
    </div>
  </AppPage>
</template>

<style scoped>
button {
  padding: 8px 12px;
  border-radius: 10px;
  border: none;
  background: var(--tgui-button-color, #2481cc);
  color: white;
  font-weight: 600;
}

button + button { margin-left: 8px; }
</style>


