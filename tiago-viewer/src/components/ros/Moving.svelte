<script>
// @ts-nocheck

  import { createEventDispatcher } from 'svelte';
  import ROSLIB from 'roslib';

  export let ros;
  let linearX = 0;
  let angularZ = 0;
  let intervalId;

  const dispatch = createEventDispatcher();

  function sendCmdVel() {
    const cmdVelTopic = new ROSLIB.Topic({
      ros: ros,
      name: '/input_joy/cmd_vel',
      messageType: 'geometry_msgs/Twist'
    });

    const twist = new ROSLIB.Message({
      linear: {
        x: linearX,
        y: 0,
        z: 0
      },
      angular: {
        x: 0,
        y: 0,
        z: angularZ
      }
    });

    cmdVelTopic.publish(twist);
  }

  function handlePressStart(direction) {
    switch (direction) {
      case 'up':
        linearX = 1;
        break;
      case 'down':
        linearX = -1;
        break;
      case 'left':
        angularZ = 1;
        break;
      case 'right':
        angularZ = -1;
        break;
    }
    sendCmdVel();
    intervalId = setInterval(sendCmdVel, 10);
  }

  function handlePressEnd() {
    dispatch('interaction', { component: 'Moving', action: 'Press End' });
    clearInterval(intervalId);
    linearX = 0;
    angularZ = 0;
    sendCmdVel();
  }
</script>

<div class="flex flex-col items-center space-y-2">
  <div class="flex justify-center">
    <div
      class="w-16 h-16 bg-gray-200 border border-gray-400 flex items-center justify-center rounded cursor-pointer active:bg-gray-300"
      role="button"
      aria-label="Move up"
      tabindex="0"
      on:mousedown={() => handlePressStart('up')}
      on:mouseup={handlePressEnd}
      on:touchstart={() => handlePressStart('up')}
      on:touchend={handlePressEnd}>
      ▲
    </div>
  </div>
  <div class="flex space-x-2">
    <div
      class="w-16 h-16 bg-gray-200 border border-gray-400 flex items-center justify-center rounded cursor-pointer active:bg-gray-300"
      role="button"
      aria-label="Move left"
      tabindex="0"
      on:mousedown={() => handlePressStart('left')}
      on:mouseup={handlePressEnd}
      on:touchstart={() => handlePressStart('left')}
      on:touchend={handlePressEnd}>
      ◄
    </div>
    <div
      class="w-16 h-16 bg-gray-200 border border-gray-400 flex items-center justify-center rounded cursor-pointer active:bg-gray-300"
      role="button"
      aria-label="Move down"
      tabindex="0"
      on:mousedown={() => handlePressStart('down')}
      on:mouseup={handlePressEnd}
      on:touchstart={() => handlePressStart('down')}
      on:touchend={handlePressEnd}>
      ▼
    </div>
    <div
      class="w-16 h-16 bg-gray-200 border border-gray-400 flex items-center justify-center rounded cursor-pointer active:bg-gray-300"
      role="button"
      aria-label="Move right"
      tabindex="0"
      on:mousedown={() => handlePressStart('right')}
      on:mouseup={handlePressEnd}
      on:touchstart={() => handlePressStart('right')}
      on:touchend={handlePressEnd}>
      ►
    </div>
  </div>
</div>

