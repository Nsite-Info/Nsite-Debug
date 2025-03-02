<template>
    <div class="mx-auto p-4 relative" v-if="hasSubdomain">
      <div v-if="isLoading" class="text-center ">
        <p class="mt-2 text-gray-500 dark:text-gray-300 min-h-24">Loading Profile<span class="loading-dots"></span></p>
      </div>
      
      <div v-else-if="eventData" class="relative z-10 text-center">
        <div class="flex justify-center mb-4 pt-12">
          <img :src="eventData.picture || ''" alt="Profile Picture" class="w-32 h-32 rounded-full border-4 border-white" />
        </div>
        <h2 class="text-3xl md:text-4xl font-semibold tracking-tight text-white uppercase">
          {{ eventData.display_name || eventData.name }}
        </h2>
      </div>
  
    </div>
  </template>
  
  <script setup>
  import { ref, onMounted, watch } from "vue";
  import setup from "~/config/project";
  import NDK from "@nostr-dev-kit/ndk";
  import { bech32 } from "bech32";
  
  const isLoading = ref(true);
  const hasSubdomain = ref(false);
  const subdomain = ref(null);
  const eventData = ref(null);
  
  function extractSubdomain(hostname) {
  if (hostname.includes("localhost")) {
    const parts = hostname.split(".");
    return parts.length > 1 && parts[0] !== "localhost" ? parts[0] : null;
  }
  const parts = hostname.split(".");
  return parts.length > 2 ? parts.slice(0, -2).join(".") : null;
  };

  
  const bytesToHex = (bytes) => {
    return Array.from(bytes)
      .map((byte) => byte.toString(16).padStart(2, "0"))
      .join("");
  };
  
  const npubToHex = (npub) => {
    const decoded = bech32.decode(npub);
    const pubkeyBytes = bech32.fromWords(decoded.words);
    return bytesToHex(Uint8Array.from(pubkeyBytes));
  };
  
  const fetchNDKEvent = async (pubkey) => {
    const ndk = new NDK({ explicitRelayUrls: setup.bootstraprelays });
    await ndk.connect();
    const event = await ndk.fetchEvent({ kinds: [0], authors: [pubkey] });
    if (event?.content) {
      eventData.value = JSON.parse(event.content);
    }
  };
  
  onMounted(async () => {
    const hostname = window.location.hostname;
    subdomain.value = extractSubdomain(hostname);
    hasSubdomain.value = !!subdomain.value;
  
    if (hasSubdomain.value) {
      const pubkey = subdomain.value.startsWith("npub1")
        ? npubToHex(subdomain.value)
        : null;
      if (pubkey) await fetchNDKEvent(pubkey);
    }
    isLoading.value = false;
  });
  
  watch(eventData, (newVal) => {
    if (!newVal) console.error("Error parsing event data");
  });
  </script>
  
  <style scoped>
  @keyframes dot-loading {
    0% { content: ''; }
    33% { content: '.'; }
    66% { content: '..'; }
    100% { content: '...'; }
  }
  
  .loading-dots::after {
    animation: dot-loading 2.5s steps(3, end) infinite;
    content: '';
  }
  
  </style>