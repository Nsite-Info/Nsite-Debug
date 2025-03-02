<template>
  <div class="mx-auto p-4 relative" v-if="hasSubdomain">
    <div v-if="isLoading" class="text-center">
      <p class="mt-2 text-gray-500 dark:text-gray-300 min-h-24">
        Loading Event Data<span class="loading-dots"></span>
      </p>
    </div>

    <div v-else-if="eventFiles.length" class="relative z-10">
      <h2 class="text-xl font-semibold tracking-tight text-white uppercase">
        Extracted Files
      </h2>

      <div class="mt-4 overflow-x-auto">
        <table class="w-full border border-purple-300">
          <thead class="bg-purple-400 ">
            <tr>
              <th class="px-4 py-2 text-left border-b border-gray-300 dark:border-gray-600">Filename</th>
              <th class="px-4 py-2 text-left border-b border-gray-300 dark:border-gray-600">Hash</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(file, index) in eventFiles" :key="index" class="border-b border-gray-300 dark:border-gray-600">
              <td class="px-4 py-2 text-purple-500">{{ file.filename }}</td>
              <td class="px-4 py-2 text-white break-all">{{ file.hash }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <div v-else class="text-center text-gray-500">
      No relevant files found.
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import setup from "~/config/project";
import NDK from "@nostr-dev-kit/ndk";
import { bech32 } from "bech32";

const isLoading = ref(true);
const hasSubdomain = ref(false);
const subdomain = ref(null);
const eventFiles = ref([]);

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

      // Extract 'd' (filename) and 'x' (hash) pairs
      const tags = event.tags.reduce((acc, tag) => {
        if (tag[0] === "d") {
          acc[tag[1]] = { filename: tag[1], hash: null };
        } else if (tag[0] === "x") {
          const lastKey = Object.keys(acc).pop();
          if (lastKey) acc[lastKey].hash = tag[1];
        }
        return acc;
      }, {});

      const newFiles = Object.values(tags).filter(file => file.filename && file.hash);
      if (newFiles.length) {
        eventFiles.value.push(...newFiles);
      }

      console.log("Updated Extracted Files:", eventFiles.value);
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
