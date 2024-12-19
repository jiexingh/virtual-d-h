<template>
  <div ref="canvasContainer" class="canvas-container">
    <canvas ref="webglRef" id="webgl" @pointermove="handleCanvasMove" @click="handleCanvasClick" />

    <div class="controls flex flex-col gap-2">
      <button
        class="px-4 py-2 bg-violet-500 rounded-md text-white"
        v-for="clip in animations"
        :key="clip.name"
        @click="playAction(clip.name)"
      >
        {{ clip.name }}
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, watch } from 'vue';
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { gsap } from 'gsap';
import modelUrl from '@/assets/RobotExpressive_aa2603d917384b68bb4a086f32dabe83.glb?url';

// References
const canvasContainer = ref(null);
const webglRef = ref(null);
const mixer = ref(null);
const animations = ref([]);
let scene, camera, renderer, controls, model;
const modelLoaded = ref(false);

watch(modelLoaded, (newValue) => {
  if (newValue) {
    addSpotLightToModel();
  }
});

// Initialize the 3D scene
const initScene = () => {
  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, canvas: webglRef.value });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setClearColor(0x000000, 0); // Transparent background

  // Add lights
  addLights();

  // Initialize controls
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.25;
  controls.enableZoom = true;

  loadModel();
  animate();

  // Add resize event listener with debounce
  window.addEventListener('resize', debounce(handleResize, 200));
};

// Add ambient, directional, point and spot lights
const addLights = () => {
  const ambientLight = new THREE.AmbientLight(0x404040, 1); // Soft ambient light
  scene.add(ambientLight);

  const directionalLight = new THREE.DirectionalLight(0xffffff, 1); // Simulate sunlight
  directionalLight.position.set(0, 10, 10);
  directionalLight.castShadow = true;
  scene.add(directionalLight);

  const pointLight = new THREE.PointLight(0xffffff, 1, 100); // Localized light
  pointLight.position.set(0, 5, 0);
  pointLight.castShadow = true;
  scene.add(pointLight);
};

const addSpotLightToModel = () => {
  const spotLight = new THREE.SpotLight(0xffffff, 1, 20, Math.PI / 4, 0.5, 2); // Focused spotlight
  spotLight.position.set(5, 10, 5);
  spotLight.target = model;
  scene.add(spotLight);
};

// Load the 3D model and animations
const loadModel = () => {
  const loader = new GLTFLoader();
  loader.load(
    // 'https://mmbizwxaminiprogram-1258344707.cos.ap-guangzhou.myqcloud.com/xr-frame/demo/miku.glb',
    modelUrl,
    // 'https://mmbizwxaminiprogram-1258344707.cos.ap-guangzhou.myqcloud.com/xr-frame/demo/damage-helmet/index.glb',
    // 'https://mmbizwxaminiprogram-1258344707.cos.ap-guangzhou.myqcloud.com/xr-frame/demo/jokers_mask_persona5.glb',
    (gltf) => {
      model = gltf.scene;
      scene.add(model);
      mixer.value = new THREE.AnimationMixer(model);
      animations.value = gltf.animations;

      modelLoaded.value = true;

      fitModelToView(model);
      animateModelScale();

      // animations.value.forEach((clip) => {
      //   // mixer.value.clipAction(clip).play();
      // });
    },
    undefined,
    (error) => {
      console.error('An error occurred while loading the model:', error);
    },
  );
};

// Fit the model to the camera view
// const fitModelToView = (model) => {
//   const box = new THREE.Box3().setFromObject(model);
//   const size = new THREE.Vector3();
//   box.getSize(size);

//   const cameraOffset = Math.max(size.x, size.y, size.z);
//   camera.position.z = cameraOffset * 1.5;
//   camera.lookAt(box.getCenter(new THREE.Vector3()));
// };

// Fit the model to the camera view (slightly lower than the center)
const fitModelToView = (model) => {
  const box = new THREE.Box3().setFromObject(model);
  const size = new THREE.Vector3();
  box.getSize(size);

  // Calculate the camera offset (distance from the camera to the model)
  const cameraOffset = Math.max(size.x, size.y, size.z);

  // Set the camera position
  camera.position.z = cameraOffset * 1.5; // Adjust distance to model

  // Adjust the camera position to make the model appear slightly lower than the center
  camera.position.y = camera.position.y + 2; // Move camera slightly down

  // Set the camera lookAt position, slightly below the center of the model's bounding box
  camera.lookAt(box.getCenter(new THREE.Vector3()).setY(box.getCenter(new THREE.Vector3()).y + 1)); // Look at the model center but slightly lower
  // camera.lookAt(
  //   box.getCenter(new THREE.Vector3()).setY(box.getCenter(new THREE.Vector3()).y - 0.2),
  // ); // Look at the model center but slightly lower

  // Optionally, if you want to fine-tune the lookAt further, you can adjust the 'y' position a bit more

  // 创建 BoxHelper 用于显示模型的边界框
  // const boxHelper = new THREE.BoxHelper(model, 0x00ff00); // 0x00ff00 为边框颜色
  // scene.add(boxHelper); // 将 BoxHelper 添加到场景中

  model.position.y = -size.y / 2; // Move the model down by half of its height
};

// Animate model scaling
const animateModelScale = () => {
  gsap.fromTo(
    model.scale,
    { x: 0.1, y: 0.1, z: 0.1 },
    { x: 1, y: 1, z: 1, duration: 1, ease: 'power2.out' },
  );
};

// Handle window resize with GSAP animation
const handleResize = () => {
  const width = window.innerWidth;
  const height = window.innerHeight;

  // Smoothly animate camera position change
  gsap.to(camera.position, {
    x: camera.position.x,
    y: camera.position.y,
    z: (width / height) * (camera.position.z / camera.aspect),
    duration: 0.5,
    onUpdate: () => {
      camera.aspect = width / height;
      camera.updateProjectionMatrix();
      renderer.setSize(width, height);
    },
  });
};

// Debounce function
const debounce = (func, delay) => {
  let timeout;
  return (...args) => {
    clearTimeout(timeout);
    timeout = setTimeout(() => {
      func.apply(this, args);
    }, delay);
  };
};

// Animation loop
const animate = () => {
  requestAnimationFrame(animate);
  if (mixer.value) mixer.value.update(0.016);
  controls.update();
  renderer.render(scene, camera);
};

// Play the selected animation
const playAction = (actionName) => {
  console.log('playAction:', actionName);

  if (mixer.value && animations.value) {
    // Find the animation clip based on the actionName
    const action = animations.value.find((clip) => clip.name === actionName);

    if (action) {
      // Find or reset the action before playing
      const clipAction = mixer.value.clipAction(action);

      // Set the loop mode to repeat forever
      clipAction.loop = THREE.LoopRepeat; // Or THREE.LoopPingPong if you prefer a ping-pong effect
      clipAction.repetitions = 1; // Set the number of repetitions (Infinity means infinite loop)

      // Stop all animations before playing the selected one to avoid conflicts
      mixer.value.clipAction(action).stop();

      // Reset and play the animation
      clipAction.reset().play();
    } else {
      console.warn(`Animation with name ${actionName} not found.`);
    }
  } else {
    console.warn('Mixer or animations not initialized.');
  }
};

// Lifecycle hooks
onMounted(initScene);
onBeforeUnmount(() => {
  window.removeEventListener('resize', handleResize);
  if (renderer) {
    renderer.dispose();
  }
});
</script>

<style>
.canvas-container {
  position: relative;
  width: 100vw;
  height: 100vh;
}
.controls {
  position: absolute;
  top: 20px;
  left: 20px;
  z-index: 1;
}
</style>
