<script>
// @ts-nocheck

    import { createEventDispatcher } from 'svelte';
    import ROSLIB from 'roslib';
    import { writable } from 'svelte/store';
  
    export let ros;
    export let showModal = writable(false);
  
    let ttsText = "";
    let ttsFeedback = "";
    let ttsResult = "";
  
    function sendTTSMessage() {
      const ttsClient = new ROSLIB.ActionClient({
        ros: ros,
        serverName: "/tts",
        actionName: "pal_interaction_msgs/TtsAction",
      });
  
      const goal = new ROSLIB.Goal({
        actionClient: ttsClient,
        goalMessage: {
          rawtext: {
            text: ttsText,
            lang_id: "en_US",
          },
        },
      });
  
      goal.on("feedback", (feedback) => {
        ttsFeedback = JSON.stringify(feedback);
      });
  
      goal.on("result", (result) => {
        ttsResult = JSON.stringify(result);
      });
  
      goal.send();
    }
  
    function handlePredefinedMessage(message) {
      ttsText = message;
      sendTTSMessage();
    }
  
    function closeModal() {
      showModal.set(false);
    }
  </script>
  
  <style>
    .modal {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      background: rgba(0, 0, 0, 0.5);
      display: flex;
      justify-content: center;
      align-items: center;
    }
    .modal-content {
      background: white;
      padding: 20px;
      border-radius: 8px;
      max-width: 400px;
      width: 100%;
    }
  </style>
  
  <div class="modal" on:click={closeModal}>
    <div class="modal-content" on:click|stopPropagation>
      <h2 class="text-lg font-semibold mb-4">Send TTS Message</h2>
      <input type="text" bind:value={ttsText} placeholder="Enter TTS text" class="p-2 border border-gray-300 rounded w-full mb-4" />
      <button on:click={sendTTSMessage} class="w-full p-2 bg-blue-500 text-white rounded mb-4">Send TTS</button>
      <div class="flex flex-col space-y-2">
        <button on:click={() => handlePredefinedMessage('Hello, I am Tiago')} class="p-2 bg-gray-200 rounded">Hello, I am Tiago</button>
        <button on:click={() => handlePredefinedMessage('How are you?')} class="p-2 bg-gray-200 rounded">How are you?</button>
        <button on:click={() => handlePredefinedMessage('I would like to help you.')} class="p-2 bg-gray-200 rounded">I would like to help you.</button>
        <button on:click={() => handlePredefinedMessage('Opening the fifth drawer')} class="p-2 bg-gray-200 rounded">Opening the fifth drawer</button>
      </div>
      <button on:click={closeModal} class="mt-4 w-full p-2 bg-red-700 text-white rounded">Close</button>
    </div>
  </div>
  