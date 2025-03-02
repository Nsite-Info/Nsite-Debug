<template>
  <div class="bg-black mx-12">
    <Profile />
    <div class="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-5">

      <div class="xl:col-start-1 md:col-span-1 xl:col-span-1">
        <Blossomlist />
      </div>
      <div class="xl:col-start-2 md:col-span-1 xl:col-span-1">
        <Relaylist />
      </div>
      <div class="xl:col-start-3 md:col-span-2 xl:col-span-3">
        <Nsitehash />
      </div>
    </div>
    <Nsitefiles />
  </div>
</template>
<style>
body {
  background-color: black;
  color: white;
}
</style>