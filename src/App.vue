<script setup>
import { ref } from "vue";
import { invoke } from "@tauri-apps/api/core";

const greetMsg = ref("");
const name = ref("");

async function greet() {
 
  greetMsg.value = await invoke("greet", { name: name.value });
}
</script>

<template>
  <div class="app">
    <header class="header">
      <h1>Alloy</h1>
    </header>

    <main class="content">
      <p>Now playing: Example Track</p>
    </main>

    <div class="player-bar" role="group" aria-label="Media controls">
      <button class="ctrl ctrl--secondary" aria-label="Loop">⟲</button>
      <button class="ctrl" aria-label="Previous">⏮</button>
      <button class="ctrl ctrl--primary" aria-label="Pause">⏸</button>
      <button class="ctrl" aria-label="Next">⏭</button>
      <button class="ctrl ctrl--secondary" aria-label="Shuffle">🔀</button>
    </div>
  </div>
</template>

<style src="./style/style.css">

</style>
