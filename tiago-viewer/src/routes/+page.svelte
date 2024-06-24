<script>
    import Viewer from './Viewer.svelte';
    import ROSConnection from './ROSConnection.svelte';
    import Moving from './Moving.svelte';
    import TTS from './TTS.svelte';
    
    /**
   * @type {any}
   */
    let ros;
    let connectionStatus = 'Connecting...';
    </script>
  
  <ROSConnection bind:ros bind:connectionStatus />
  
  <div>
      <p class="status">{connectionStatus}</p>
      <Viewer />  
      <Moving {ros} />
      <TTS {ros} />
  </div>
  
  <style>
    .status {
      font-size: 1.5em;
      margin-bottom: 20px;
    }
  </style>