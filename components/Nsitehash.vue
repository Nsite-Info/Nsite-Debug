<template>
  <div class="mx-auto p-4 relative" v-if="hasSubdomain">
    <div v-if="isLoading" class="text-center">
      <p class="mt-2 text-gray-500 dark:text-gray-300 min-h-24">
        Loading Event Data<span class="loading-dots"></span>
      </p>
    </div>

    <div v-else-if="filteredEventTags.length" class="relative z-10 text-center my-2">
      <h2 class="text-xl font-semibold tracking-tight text-white dark:text-white uppercase">
        Nsite Index Hash
      </h2>
      <p class="text-white">Kind: 34128</p>
      <ul class="mt-4 text-lg">
        <li v-for="(tag, index) in filteredEventTags" :key="index" class="text-purple-500">
          {{ tag.value }}
        </li>
      </ul>
    </div>

    <div v-else class="text-center text-gray-500">
      No relevant tags found.
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from "vue";
import setup from "~/config/project";
import NDK from "@nostr-dev-kit/ndk";
import { bech32 } from "bech32";

const isLoading = ref(true);
const hasSubdomain = ref(false);
const subdomain = ref(null);
const eventTags = ref([]);

const filteredEventTags = computed(() => {
  return eventTags.value.filter(tag => tag.type !== "d"); // Hide "d" tag from UI
});

function extractSubdomain(hostname) {
  if (hostname.includes("localhost")) {
    const parts = hostname.split(".");
    return parts.length > 1 && parts[0] !== "localhost" ? parts[0] : null;
  }
  const parts = hostname.split(".");
  return parts.length > 2 ? parts.slice(0, -2).join(".") : null;
}

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

const subscribeToNDKEvents = async (pubkey) => {
  const ndk = new NDK({ explicitRelayUrls: setup.bootstraprelays });
  await ndk.connect();

  try {
    const filter = {
      kinds: [34128], // Listen for Kind 34128 events
      authors: [pubkey],
    };

    const subscription = ndk.subscribe(filter);

    subscription.on("event", (event) => {
      console.log("New Event Received:", event);

      // Extract relevant tags
      const dTag = event.tags.find(tag => tag[0] === "d" && tag[1] === "/index.html");
      const xTag = event.tags.find(tag => tag[0] === "x");

      // Only update if a matching 'd' tag exists
      if (dTag) {
        eventTags.value = [
          ...(xTag ? [{ type: "x", value: xTag[1] }] : []) // Add 'x' tag if present, exclude 'd'
        ];
      }

      console.log("Updated Extracted Tags:", eventTags.value);
    });

    subscription.start();
  } catch (error) {
    console.error("Error subscribing to NDK events:", error);
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
    if (pubkey) await subscribeToNDKEvents(pubkey);
  }
  isLoading.value = false;
});
</script>
