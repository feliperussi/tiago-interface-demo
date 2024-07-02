<script>
// @ts-nocheck

  import { onMount } from 'svelte';
  import { writable } from 'svelte/store';
  import ROSConnection from '../components/ros/ROSConnection.svelte';
  import Moving from '../components/ros/Moving.svelte';
  import TTS from '../components/ros/TTS.svelte';
  import VideoGrid from '../components/VideoGrid.svelte';

  let ros;
  let connectionStatus = 'Connecting...';
  let showModal = writable(false);

  function logInteraction(event) {
    console.log(`Interaction with ${event.detail.component} component: ${event.detail.action}`);
  }

  function openChat() {
    showModal.set(true);
  }

  onMount(() => {
    showModal.set(false);
  });
</script>

<ROSConnection bind:ros bind:connectionStatus on:interaction={logInteraction} />

<div class="flex flex-col items-center space-y-4">
  <p class="status">{connectionStatus}</p>
  <div class="flex justify-center">
    <VideoGrid />
  </div>
  <div class="flex justify-center space-x-4">
    <button on:click={openChat} class="bg-blue-500 text-white p-2 rounded">Open Chat</button>
    <Moving {ros} on:interaction={logInteraction} />
    <button class="bg-gray-500 text-white p-2 rounded">Another Button</button>
  </div>
</div>

{#if $showModal}
  <TTS {ros} bind:showModal />
{/if}

<style>
</style>
