<template>
  <div class="mx-auto p-4 relative" v-if="hasSubdomain">
    <div v-if="isLoading" class="text-center">
      <p class="mt-2 text-gray-500 dark:text-gray-300 min-h-24">
        Loading Relay Server List<span class="loading-dots"></span>
      </p>
    </div>

    <div v-else-if="eventTags.length" class="relative z-10 text-center my-2">
      <h2 class="text-xl font-semibold tracking-tight text-white dark:text-white uppercase">
        Relay URLs
      </h2>
      <p class="text-white">NIP-65</p>
      <ul class="mt-4 text-lg">
        <li v-for="(url, index) in eventTags" :key="index" class="text-purple-500 hover:underline">
          <a :href="url" target="_blank">{{ url }}</a>
        </li>
      </ul>
    </div>

    <div v-else class="text-center text-gray-500">
      No server tags found.
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
  const eventTags = ref([]);

  
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
    
    try {
        const event = await ndk.fetchEvent({ kinds: [10002], authors: [pubkey] });

        if (event) {
            console.log("Fetched Event:", event);

            // Extract and filter only 'server' tags
            eventTags.value = event.tags
                .filter(tag => tag[0] === "r")
                .map(tag => tag[1]); // Get only the URLs

            console.log("Extracted Server Tags:", eventTags.value);
        } else {
            console.warn("No event found for kind 10063");
        }
    } catch (error) {
        console.error("Error fetching NDK event:", error);
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