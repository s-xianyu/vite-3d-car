<template>
  <div ref="canvas">
    <div class="color-picker">
      <div class="light-color-picker color-picker-wrap">
        <div>
          <input v-model="backLightColor" type="color" />
        </div>
        <div>背景光</div>
      </div>
      <div class="car-color-picker color-picker-wrap">
        <div>
          <input v-model="carCoatColor" type="color" />
        </div>
        <div>车身色</div>
      </div>
    </div>
  </div>
</template>
<script setup>
// This starter template is using Vue 3 <script setup> SFCs
// Check out https://vuejs.org/api/sfc-script-setup.html#script-setup
import { ref, onMounted, watch } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from '@/utils/GLTFLoader';
import { DRACOLoader } from '@/utils/DRACOLoader';
import { OrbitControls } from "@/utils/OrbitControls";
import {
  getMaterials,
  changModel,
  createCustomMaterial,
  generateVirtualLight,
  floatMesh,
  setMovingSpot,
  createContactShadow
} from "@/utils/utils";
const canvas = ref();
// 场景对象
let scene;
// 3D 模型对象
let model;
const modelZ = 0;
// 车身颜色
let carCoatColor = ref("#2f426f");
// 背景光源颜色
let backLightColor = ref("#2de8cd");
// 虚拟背景
let virtualBackgroundMesh;
// 相机运动锁
let cameraMoveClock = false;
// 相机控制对象
let controls;
// 虚拟场景对象
let virtualScene = new THREE.Scene();
// 透视摄像机
let camera = new THREE.PerspectiveCamera(30, window.innerWidth / window.innerHeight)
// 聚光灯对象
const spotLight = new THREE.SpotLight();
// 环境光
const ambientLight = new THREE.AmbientLight(0x404040);
// 模型的接触阴影对象
const shadowGroup = new THREE.Group();

// 渲染器，用WebGL渲染出你精心制作的场景
const renderer = new THREE.WebGLRenderer({
  powerPreference: "high-performance",
  antialias: true,
  alpha: true,
});
// 设置渲染器
const setRender = () => {
  renderer.shadowMap.enabled = true;
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio)
  // GLTFLoader 将自动配置从 .gltf 或 .glb 文件引用的纹理。
  renderer.outputEncoding = THREE.sRGBEncoding;
  //是否使用物理上正确的光照模式。默认为false。
  // renderer.physicallyCorrectLights = true;
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  // BakeShadows
  renderer.shadowMap.autoUpdate = false;
  renderer.shadowMap.needsUpdate = true;
}
// 设置环境光
const setAmbientLight = () => {
  //intensity - (参数可选)光照的强度。缺省值为 1。
  ambientLight.intensity = 0.2;
  scene.add(ambientLight);
}
// 设置相机
const setCamera = () => {
  camera.position.set(0, 0.8, 8);
  camera.castShadow = true;
  scene.add(camera);
}
// 加载3d模型
const load3DModel = async () => {
  // return new Promise((resolve => {
  //   new GLTFLoader().load('/001/red_bricks_04_4k.gltf', gltf => {
  //     model = gltf
  //     if (model.scene) {
  //       const { materials, nodes } = getMaterials(model.scene);
  //
  //       Object.values(nodes).forEach((node) => {
  //         node.receiveShadow = false;
  //         node.castShadow = false;
  //       });
  //       Object.assign(model, { materials, nodes });
  //     }
  //     model.scene.scale.set(1.6, 1.6, 1.6);
  //     // model.scene.position.set(-0.5, -0.18, modelZ);
  //     model.scene.position.set(0, 0, modelZ);
  //     model.scene.rotation.set(0, Math.PI, 0);
  //     scene.add(model.scene);
  //   })
  //   resolve()
  // }))

  // glb 是压缩的gltf，需要使用 dracoLoader 解压缩
  const dracoLoader = new DRACOLoader().setDecoderPath("/libs/draco/");
  // gltf 加载器
  const gltfLoader = new GLTFLoader().setDRACOLoader(dracoLoader);
  // 加载模型获取，模型对象信息
  const model = await gltfLoader.loadAsync("/911.glb");
  console.log(model)
  // 从模型中获取材料和节点对象，并添加至模型对象上
  if (model.scene) {
    const { materials, nodes } = getMaterials(model.scene);

    Object.values(nodes).forEach((node) => {
      node.receiveShadow = false;
      node.castShadow = false;
    });
    Object.assign(model, { materials, nodes });
  }
  model.scene.scale.set(1.6, 1.6, 1.6);
  // model.scene.position.set(-0.5, -0.18, modelZ);
  model.scene.position.set(0, 0, modelZ);
  model.scene.rotation.set(0, Math.PI, 0);
  scene.add(model.scene);
}
// 自动以3d模型
const customModel = async () => {
  changModel(model, "rubber", {
    color: "#222",
    roughness: 0.6,
    roughnessMap: null,
    normalScale: [4, 4],
  });
  changModel(model, "window", {
    color: "black",
    roughness: 0,
    clearcoat: 0.1,
  });
  changModel(model, "coat", {
    color: carCoatColor.value,
    envMapIntensity: 4,
  });

  changModel(model, "paint", {
    roughness: 0.5,
    metalness: 0.8,
    envMapIntensity: 2,
  });
}
// 设置聚光灯
const setSpotLight = async () => {
  spotLight.position.set(0, 15, 0);
  //intensity - (可选参数) 光照强度。 缺省值 1。
  spotLight.intensity = 2;
  //聚光锥的半影衰减百分比。在0和1之间的值。默认为0。
  spotLight.penumbra = 1;
  //从聚光灯的位置以弧度表示聚光灯的最大范围。应该不超过 Math.PI/2。默认值为 Math.PI/3。
  spotLight.angle = 0.3;
  spotLight.shadow.bias = -0.0001;
  spotLight.castShadow = true;
  spotLight.target.position.set(0, 0, 6);
  scene.add(spotLight.target);

  scene.add(spotLight);
}
// 虚拟HDR环境，设置场景的环境
const setEnvironment = (
    scene,
    resolution = 256,
    frames = 1,
    near = 1,
    far = 1000,
    background = false
) => {
  const fbo = new THREE.WebGLCubeRenderTarget(resolution);
  fbo.texture.type = THREE.HalfFloatType;
  const cubeCamera = new THREE.CubeCamera(near, far, fbo);

  virtualScene.add(cubeCamera);

  // 天花板灯
  const topLight = generateVirtualLight({
    intensity: 0.75,
    scale: [10, 10, 1],
    position: [0, 5, -9],
    rotation: [Math.PI / 2, 0, 0],
  });

  // 四周灯光
  const leftTopLight = generateVirtualLight({
    intensity: 4,
    scale: [20, 0.1, 1],
    position: [-5, 1, -1],
    rotation: [0, Math.PI / 2, 0],
  });
  const leftBottomLight = generateVirtualLight({
    intensity: 1,
    scale: [20, 0.5, 1],
    position: [-5, -1, -1],
    rotation: [0, Math.PI / 2, 0],
  });
  const rightTopLight = generateVirtualLight({
    intensity: 1,
    scale: [20, 1, 1],
    position: [10, 1, 0],
    rotation: [0, -Math.PI / 2, 0],
  });

  const floatLight = generateVirtualLight({
    form: "ring",
    color: "red",
    intensity: "1",
    scale: 10,
    position: [-15, 4, -18],
    target: [0, 0, 0],
  });

  virtualScene.add(topLight);
  virtualScene.add(leftTopLight);
  virtualScene.add(leftBottomLight);
  virtualScene.add(rightTopLight);
  virtualScene.add(floatLight);

  if (background !== "only") {
    scene.environment = fbo.texture;
  }
  if (background) {
    scene.background = fbo.texture;
  }

  // 模拟 MeshDepthMaterial 设置背景颜色，颜色不受光照影响

  const geometry = new THREE.SphereGeometry(1, 64, 64);
  const material = createCustomMaterial(backLightColor.value);

  virtualBackgroundMesh = new THREE.Mesh(geometry, material);
  virtualBackgroundMesh.scale.set(100, 100, 100);
  virtualScene.add(virtualBackgroundMesh);

  // 让环形网格运动起来
  floatMesh({
    group: floatLight,
    speed: 5,
    rotationIntensity: 2,
    floatIntensity: 2,
  });

  // 更新相机内容
  let count = 1;
  const virtualRender = () => {
    if (frames === Infinity || count < frames) {
      cubeCamera.update(renderer, virtualScene);
      count++;
    }
    requestAnimationFrame(virtualRender);
  };
  virtualRender();
}
// 为车设置聚光灯
const setBigSpotLight = () => {
  // model.scene.receiveShadow = false;
  scene.background = new THREE.Color("#d4cfa3");
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;

  const material = new THREE.MeshPhongMaterial({
    side: THREE.DoubleSide,
    color: "#00ff1a",
    emissive: "#ac8f3e",
  });
  const FloorGeometry = new THREE.PlaneGeometry(200, 200);

  const floorMesh = new THREE.Mesh(FloorGeometry, material);

  floorMesh.rotation.x = Math.PI / 2;
  floorMesh.receiveShadow = true;

  floorMesh.position.set(0, -1.02, 0);

  scene.add(floorMesh);

  const bigSpotLight = new THREE.SpotLight("#ffffff", 2);

  bigSpotLight.angle = Math.PI / 8;
  bigSpotLight.penumbra = 0.2;
  bigSpotLight.decay = 2;
  bigSpotLight.distance = 30;

  bigSpotLight.position.set(0, 10, 0);
  bigSpotLight.target.position.set(0, 0, modelZ);

  scene.add(bigSpotLight.target);
  scene.add(bigSpotLight);
  // guiMeshPhongMaterial(floorMesh, material, FloorGeometry);
};
// 设置接触阴影,显得更真实
const setContactShadow = () => {
  shadowGroup.position.set(0, -1.01, modelZ);
  shadowGroup.rotation.set(0, Math.PI / 2, 0);
  scene.add(shadowGroup);
  createContactShadow(scene, renderer, shadowGroup);
};

let stopID;
const clock = new THREE.Clock();
let cameraX, cameraZ;
const setCameraAnimate = () => {
  const vector = new THREE.Vector3();

  if (cameraMoveClock) {
    cancelAnimationFrame(stopID);
  } else {
    const t = clock.getElapsedTime();

    const theta = t / 6;

    const newx = 14 * Math.sin(theta) - 6;
    const newy = 14 * Math.cos(theta) - 6;
    cameraX = newx < -10 ? cameraX : newx;
    cameraZ = newy < -10 ? cameraZ : newy;

    camera.position.lerp(vector.set(cameraX, 0.5, cameraZ), 0.05);
    camera.lookAt(model.scene.position);
  }

  stopID = requestAnimationFrame(setCameraAnimate);
};
// 添加控制效果
const addControls = () => {
  controls = new OrbitControls(camera, renderer.domElement);

  controls.addEventListener("start", () => (cameraMoveClock = true));
  controls.addEventListener("end", () => {
    cameraMoveClock = false;
  });
};
// 开启实时渲染，无需手动渲染
const autoRender = () => {
  // controls.update();
  renderer.render(scene, camera);
  requestAnimationFrame(autoRender);
};
// 监听颜色变化
const watchColorChange = () => {
  //车身样色改变
  watch(carCoatColor, (val, old) => {
    requestAnimationFrame(() => {
      if (val) {
        model.materials.coat.color.set(val);
        model.materials.coat.needsUpdate = true;
      }
    });
  });

  // 灯光颜色改变
  watch(backLightColor, (val, old) => {
    requestAnimationFrame(() => {
      if (val) {
        virtualBackgroundMesh.material = createCustomMaterial(val);
        virtualBackgroundMesh.material.needsUpdate = true;
      }
    });
  });
};
// 监听页面变化重新渲染画布
const listenPageSizeChange = () => {
  window.addEventListener("resize", changeRenderSize);
};
const changeRenderSize = () => {
  requestAnimationFrame(() => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();

    renderer.setSize(window.innerWidth, window.innerHeight);
  });
};
const init = async () => {
  // 创建场景
  scene = new THREE.Scene();
  // 设置渲染器
  setRender();
  // 设置环境光
  setAmbientLight();
  // 设置相机
  setCamera();
  // 加载3d模型
  await load3DModel();
  // 自动以3d模型
  customModel();
  // 设置聚光灯
  setSpotLight();
  const t = 600;
  setTimeout(() => {
    // 设置 HDR环境（模拟HDR贴图）
    setEnvironment(scene, 256, Infinity);
    // 设置运动光源，获得打光效果
    setMovingSpot(virtualScene);
  }, t * 2);
  setTimeout(() => {
    spotLight.visible = false;
    // 设置舞台聚光灯
    setBigSpotLight();
    // 设置接触阴影
    setContactShadow();
  }, 3 * t);
  setTimeout(() => {
    // 设置相机运动动画
    setCameraAnimate();
  }, t * 5);
  // 添加控制效果
  addControls();
  // 将渲染结果写入html dom中
  canvas.value.appendChild(renderer.domElement);
  // 开启实时渲染，无需手动渲染
  autoRender();
  // 监听颜色变化
  watchColorChange();
  // 监听页面变化重新渲染画布
  listenPageSizeChange();
}
onMounted(() => {
  init()
})
</script>

<style>
/* semantic color variables for this project */
:root {
  --color-background: #ffffff;
  --color-text: #333333;
}

* {
  box-sizing: border-box;
}

body, div, dl, dt, dd, ul, ol, li, h1, h2, h3, h4, h5, h6, pre, code, form, fieldset, legend, input, button, textarea, p, blockquote, th, td {
  margin: 0;
  padding: 0;
}

body {
  min-height: 100vh;
  color: var(--color-text);
  background: var(--color-background);
  transition: color 0.5s, background-color 0.5s;
  line-height: 1.6;
  font-family: Inter, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu,
  Cantarell, 'Fira Sans', 'Droid Sans', 'Helvetica Neue', sans-serif;
  font-size: 15px;
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#app {
  position: absolute;
  top: 0;
  right: 0;
  left: 0;
  bottom: 0;
  overflow: hidden;
  /*background: #4e5969;*/
  /*background: #d5cca3;*/
  background: #0f0001;
}

.scene {
  width: 100%;
  height: 100%;
}

.color-picker {
  /*display: none;*/
  position: absolute;
  top: 5%;
  right: 5%;
  color: darkgray;
  font-size: 12px;
  text-align: center;
  background: white;
  border-radius: 4px;
  box-shadow: 0 4px 8px rgb(0 0 0 / 40%);
  /*box-shadow: 0 3px 5px -1px #00000033, 0 5px 8px 0 #00000024, 0 1px 14px 0 #0000001f;*/
}

.color-picker .color-picker-wrap {
  display: inline-block;
  width: 60px;
  height: 60px;
  text-align: center;
  padding-top: 6px;
}

.color-picker .color-picker-wrap:hover {
  background: rgba(0, 0, 0, 0.1);
}


input[type=color] {
  width: 28px;
  height: 28px;
  border: none;
  cursor: pointer;
  padding: 0;
  box-sizing: border-box;
  background: none;
}


::-webkit-color-swatch {
  display: block;
  width: 24px;
  height: 24px;
  padding: 0;
  box-sizing: border-box;
  border-radius: 100%;
  border: 0.5px solid #ffffff;
  box-shadow: 0 3px 5px -1px #00000033, 0 5px 8px 0 #00000024, 0 1px 14px 0 #0000001f;
}


</style>
