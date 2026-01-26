<script setup>
import { ref } from "vue";
import { invoke } from "@tauri-apps/api/core";

const greetMsg = ref("");
const name = ref("");

async function greet() {
  // Learn more about Tauri commands at https://tauri.app/develop/calling-rust/
  greetMsg.value = await invoke("greet", { name: name.value });
}
</script>

<template>
<div class="header">
  <h1>Lorem</h1>
  
</div>
</template>

<style src="./style/style.css" scoped>

</style>
<style>


</style>
