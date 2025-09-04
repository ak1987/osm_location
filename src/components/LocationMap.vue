<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref } from 'vue';
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
    return requestBrowserGeolocation();
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

function requestBrowserGeolocation() {
  if (!('geolocation' in navigator)) {
    lastError.value = 'Browser geolocation is not available.';
    return;
  }
  navigator.geolocation.getCurrentPosition(
    (pos) => {
      const { latitude, longitude } = pos.coords;
      initializeMap(latitude, longitude);
    },
    (err) => {
      lastError.value = `Geolocation error: ${err.message}`;
    },
    { enableHighAccuracy: true, timeout: 10000, maximumAge: 0 },
  );
}

onMounted(async () => {
  await mountLM();
  if (isMounted.value) void requestLocationOnce();
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
  <div>
    <div style="display:flex;gap:8px;margin-bottom:8px;">
      <button @click="requestLocationOnce">Request Location</button>
      <button @click="openSettings">Open Settings</button>
      <button @click="requestBrowserGeolocation">Try Browser Geolocation</button>
    </div>
    <div style="margin-bottom:6px; font-size: 12px; opacity: .8;">
      <div>supported: {{ String(isSupported) }}</div>
      <div>mounting: {{ String(isMounting) }}</div>
      <div>mounted: {{ String(isMounted) }}</div>
    </div>
    <div v-if="isMounting">Mounting location manager…</div>
    <div v-if="mountError" style="color: var(--tgui-text-color-destructive);">{{ String(mountError) }}</div>
    <div v-if="lastError" style="color: var(--tgui-text-color-destructive);">{{ lastError }}</div>
    <div ref="mapContainer" style="height:65vh;width:100%;border-radius:12px;overflow:hidden;background:#e9ecef;" />
    <div style="margin-top:8px; font-size:12px; opacity:.75;">
      Tip: Permission prompt appears only after user interaction and only inside Telegram apps.
    </div>
  </div>
  
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


