<script>
  import { onMount, createEventDispatcher } from 'svelte';
  import ROSLIB from 'roslib';

  export let ros;
  export let connectionStatus;

  const dispatch = createEventDispatcher();

  onMount(() => {
    ros = new ROSLIB.Ros({
      url: 'ws://192.168.0.106:9090'  // Replace with your robot's IP
    });

    ros.on('connection', () => {
      console.log('Connected to websocket server.');
      connectionStatus = 'Connected';
      dispatch('interaction', { component: 'ROSConnection', action: 'Connected' });
    });

    ros.on('error', (/** @type {any} */ error) => {
      console.log('Error connecting to websocket server: ', error);
      connectionStatus = 'Error';
      dispatch('interaction', { component: 'ROSConnection', action: 'Error' });
    });

    ros.on('close', () => {
      console.log('Connection to websocket server closed.');
      connectionStatus = 'Closed';
      dispatch('interaction', { component: 'ROSConnection', action: 'Closed' });
    });
  });
</script>
