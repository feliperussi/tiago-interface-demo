<script>
// @ts-nocheck

    import { onMount } from 'svelte';
    import ROSLIB from 'roslib';
    import * as THREE from 'three';
    import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
  
    let scene, camera, renderer, points, robotSphere, controls;
    let robotPosition = new THREE.Vector3(0, 0, 0); // Initial robot position
  
    let laserScanData = {
      ranges: [],
      angle_min: 0,
      angle_max: 0,
      angle_increment: 0
    };
  
    const vertexShader = `
      attribute float distance;
      varying float vDistance;
      void main() {
        vDistance = distance;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        gl_PointSize = 5.0;
      }
    `;
  
    const fragmentShader = `
      varying float vDistance;
      void main() {
        float intensity = vDistance / 10.0;
        gl_FragColor = vec4(intensity, 1.0 - intensity, 0.0, 1.0);
      }
    `;
  
    function init() {
      scene = new THREE.Scene();
      camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
      renderer = new THREE.WebGLRenderer();
      renderer.setSize(window.innerWidth, window.innerHeight);
      document.getElementById('canvas-container').appendChild(renderer.domElement);
  
      const geometry = new THREE.BufferGeometry();
      const material = new THREE.ShaderMaterial({
        vertexShader: vertexShader,
        fragmentShader: fragmentShader,
        attributes: {
          distance: { type: 'f', value: null }
        }
      });
  
      points = new THREE.Points(geometry, material);
      scene.add(points);
  
      // Add robot sphere with different color
      const robotGeometry = new THREE.SphereGeometry(0.17, 32, 32);
      const robotMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000 });
      robotSphere = new THREE.Mesh(robotGeometry, robotMaterial);
      scene.add(robotSphere);
  
      camera.position.set(0, -5, 5); // Position the camera to look north
      camera.lookAt(0, 0, 0);
  
      // Initialize OrbitControls
      controls = new OrbitControls(camera, renderer.domElement);
      controls.enableDamping = true; // Enable damping (inertia)
      controls.dampingFactor = 0.25; // Damping factor
      controls.screenSpacePanning = false; // Do not allow panning
  
      controls.minAzimuthAngle = 0; // Restrict horizontal rotation
      controls.maxAzimuthAngle = 0;
      controls.minPolarAngle = 50 * Math.PI / 180; // Limit vertical rotation angle
      controls.maxPolarAngle = 170 * Math.PI / 180;
  
      animate();
    }
  
    function animate() {
      requestAnimationFrame(animate);
      controls.update(); // Update the controls
      renderer.render(scene, camera);
    }
  
    function updatePoints() {
      if (laserScanData.ranges.length > 0) {
        const positions = [];
        const distances = [];
        const angleMin = laserScanData.angle_min;
        const angleIncrement = laserScanData.angle_increment;
  
        laserScanData.ranges.forEach((range, index) => {
          const angle = angleMin + index * angleIncrement;
          const x = range * Math.sin(angle);
          const y = range * Math.cos(angle);
          positions.push(-x, y, 0);
          distances.push(range);
        });
  
        points.geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
        points.geometry.setAttribute('distance', new THREE.Float32BufferAttribute(distances, 1));
        points.geometry.attributes.position.needsUpdate = true;
        points.geometry.attributes.distance.needsUpdate = true;
  
        // Update robot sphere position
        robotSphere.position.set(robotPosition.x, robotPosition.y, robotPosition.z);
      }
    }
  
    onMount(() => {
      init();
  
      const ros = new ROSLIB.Ros({
        url: 'ws://192.168.0.106:9090',
      });
  
      ros.on('connection', () => {
        console.log('Connected to websocket server.');
      });
  
      ros.on('error', (error) => {
        console.error('Error connecting to websocket server: ', error);
      });
  
      ros.on('close', () => {
        console.log('Connection to websocket server closed.');
      });
  
      const laserScanListener = new ROSLIB.Topic({
        ros: ros,
        name: '/scan',
        messageType: 'sensor_msgs/LaserScan',
      });
  
      laserScanListener.subscribe((message) => {
        laserScanData = message;
        updatePoints();
      });
    });
  </script>
  
  <main>
    <div id="canvas-container"></div>
  </main>
  
  <style>
    main {
      padding: 2rem;
    }
  
    #canvas-container {
      width: 100%;
      height: 100%;
      position: absolute;
      top: 0;
      left: 0;
    }
  
  </style>
  