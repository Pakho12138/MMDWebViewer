<template>
  <div>
    <canvas ref="threeRef"></canvas>
    <div ref="statsRef"></div>
  </div>
</template>

<script setup lang="ts">
import * as THREE from 'three';
import { MMDLoader } from 'three/examples/jsm/loaders/MMDLoader.js';
import { MMDAnimationHelper } from 'three/examples/jsm/animation/MMDAnimationHelper';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { GUI } from 'three/examples/jsm/libs/lil-gui.module.min.js';
import Stats from 'three/examples/jsm/libs/stats.module.js';
import { Water } from 'three/examples/jsm/objects/Water';
import { Reflector } from 'three/examples/jsm/objects/Reflector';
import { onMounted, ref, nextTick, watch, onBeforeUnmount } from 'vue';

const threeRef = ref(null);
const statsRef: any = ref(null);
let scene: any;
let camera: any;
let renderer: any;
let orbitControls: any; // 相机控件
let clock: any;
let lightBox: any;
let lightRotate = false; // 灯光旋转
let stats: any; // 资源监视器
let loader: any; // MMD加载器
let MMDCamare: any; // MMD照相机轨
let MMDHelper: any; // MMD照相机轨
let audio: any;
let MMDCanPlay: boolean = false;
let playControl = ref(false);
let objBox: any = {
  stage: '',
  box: '',
};
let managerBox: any; // 加载器
let helperBox: any; // 加载器
let animationFrame: any; // 动画

let water: any; // 水面
let stageMesh: any; // 舞台Mesh
let cubeCamera: any; //

function init() {
  initRender();
  initScene();
  initCamera();
  initClock();
  initManager();
  initLoader();
  // addWater();
  initObj();
  initLight();
  initHelper();
  initStats();
  initOrbitControls();
  initGui();
  updateRenderer();
}

function addWater() {
  const waterGeometry = new THREE.PlaneGeometry(10000, 10000);

  water = new Water(waterGeometry, {
    textureWidth: 512,
    textureHeight: 512,
    waterNormals: new THREE.TextureLoader().load('img/waternormals.jpg', function (texture: any) {
      texture.wrapS = texture.wrapT = THREE.RepeatWrapping;
    }),
    sunDirection: new THREE.Vector3(),
    sunColor: 0xffffff,
    // waterColor: 0x001e0f,
    waterColor: 0x000000,
    distortionScale: 3.7,
    fog: scene.fog !== undefined,
    transparent: true,
    opacity: 0.5,
  });

  water.rotation.x = -Math.PI / 2;
  water.position.y = 0.1;
  // water.material.transparent = true;
  // water.material.opacity = 0; // 设置透明度，值在0到1之间，0为完全透明，1为完全不透明
  scene.add(water);
}

// 初始渲染器
function initRender() {
  renderer = new THREE.WebGLRenderer({
    canvas: threeRef.value,
    antialias: true,
    alpha: true,
  });
  renderer.setSize(window.innerWidth, window.innerHeight);
  // renderer.setClearAlpha(0.0); // 设置背景透明度
  renderer.setClearColor(0x000000, 0);
  renderer.setPixelRatio(window.devicePixelRatio);

  renderer.physicallyCorrectLights = true; //正确的物理灯光照射
  renderer.outputEncoding = THREE.sRGBEncoding; //采用sRGBEncoding
  renderer.toneMapping = THREE.ACESFilmicToneMapping; //aces标准
  renderer.toneMappingExposure = 1.25; //色调映射曝光度
  renderer.shadowMap.enabled = true; //渲染器阴影渲染
  renderer.shadowMap.type = THREE.PCFSoftShadowMap; //阴影类型（处理运用Shadow Map产生的阴影锯齿）

  // 定义gammaOutput和gammaFactor
  renderer.gammaOutput = true;
  renderer.gammaFactor = 2.2; //电脑显示屏的gammaFactor为2.2
}

// 渲染场景
function initScene() {
  scene = new THREE.Scene(); // 场景
  // scene.background = new THREE.Color('#000');
  // scene.fog = new THREE.Fog('#000', 20, 100);
}

// 渲染时钟
function initClock() {
  clock = new THREE.Clock(); // 世界时钟
}

// 初始化光源
function initLight() {
  lightBox = {
    ambientLight: new THREE.AmbientLight('rgb(50, 50, 50)', 1.7), // 环境光
    spotLight: new THREE.SpotLight('rgb(255, 255, 255)', 1, 200, (Math.PI / 180) * 30, 0.3, 0), // 聚光灯
  };
  lightBox.spotLight.position.set(0, 70, 80);
  lightBox.spotLight.castShadow = true;
  lightBox.spotLight.shadow.mapSize = new THREE.Vector2(1024, 1024);
  scene.add(lightBox.spotLight);
  scene.add(lightBox.ambientLight);

  // // 创建环境光照
  // if (scene.getObjectByName('ambientLight')) scene.remove(scene.getObjectByName('ambientLight'));
  // const ambientLight = new THREE.AmbientLight(0xa2a2a2, 1);
  // ambientLight.name = 'ambientLight';
  // scene.add(ambientLight);

  // const dirLight = new THREE.DirectionalLight(0xffffff, 1);
  // //光源等位置
  // dirLight.position.set(2, 5, 2);
  // //可以产生阴影
  // dirLight.castShadow = true;
  // dirLight.shadow.mapSize = new THREE.Vector2(1024, 1024);
  // scene.add(dirLight);

  // // 添加半球光光源
  // const hemLight = new THREE.HemisphereLight(0x7e7ab0, 0xffc0cb, 1);
  // hemLight.position.set(0, 5, 0);
  // scene.add(hemLight);
}

// 初始化相机
function initCamera() {
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 1, 1000); // 相机
  camera.position.x = 0;
  camera.position.y = 15;
  camera.position.z = 25;
  camera.up.x = 0;
  camera.up.y = 1;
  camera.up.z = 0;
  // this.camera.lookAt(new THREE.Vector3(0, 0, 0))

  // camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  // camera.position.set(0, 15, 25);
}

// 初始加载管理器
function initManager() {
  // 加载器配置
  let long = 40;
  let y = 0;
  // 加载器公共事件
  let managerEvent = (name: string, manager: any) => {
    manager.onStart = (url: string, itemsLoaded: boolean, itemsTotal: number) => {
      y += 10;
      let geometry = new THREE.BoxGeometry(long, 2, 2, 4, 4);
      let material = new THREE.MeshStandardMaterial({
        color: `rgb(125, 255, 255)`,
        roughness: 0,
        metalness: 0,
      });
      objBox[name] = new THREE.Mesh(geometry, material);
      objBox[name].castShadow = true;
      objBox[name].receiveShadow = true;
      objBox[name].position.set(-long / 2, y, 0);
      objBox[name].scale.x = 0;
      scene.add(objBox[name]);

      objBox[name + 'BoxLine'] = new THREE.LineSegments(new THREE.EdgesGeometry(geometry), new THREE.LineBasicMaterial({ color: 0xffffff }));
      objBox[name + 'BoxLine'].position.set(0, y, 0);
      scene.add(objBox[name + 'BoxLine']);
    };
    manager.onLoad = () => {
      scene.remove(objBox[name]);
      clearObjCache(objBox[name]);
      scene.remove(objBox[name + 'BoxLine']);
      clearObjCache(objBox[name + 'BoxLine']);
    };
    manager.onProgress = (url: string, itemsLoaded: number, itemsTotal: number) => {
      console.log(Math.round((itemsLoaded / itemsTotal) * 100) + '%');
      const progress = Math.round((itemsLoaded / itemsTotal) * 100) / 100;
      objBox[name].scale.x = progress;
      objBox[name].position.x = -long / 2 + (progress / 2) * long; // 调整位置，使对象回到原来的位置
    };
    manager.onError = (url: string) => {
      objBox[name + 'BoxLine'].material.color.setHex('rgb(255, 0, 0)');
    };
  };
  // 初始加载器
  managerBox = {
    MMDManager: new THREE.LoadingManager(),
  };
  managerEvent('MMDManager', managerBox.MMDManager);
}

let personModelMesh: any;
// 初始化加载器
function initLoader() {
  loader = new MMDLoader(managerBox.MMDManager);

  const stageUrl = '/model/夜月蓝/场景.pmx';
  // const stageUrl = '/model/room/Stage0514.pmx';
  loader.load(stageUrl, (mmd: any) => {
    // mmd使用了特殊的材质，无法通过traverse访问到
    mmd.material.forEach((item: any) => {
      item.emissive = new THREE.Color(0x000000); // 定义材质发射的颜色
      item.emissiveIntensity = 1; // 设置发光强度

      // 处理树叶材质的显示
      if (item.name == '材質2') {
        item.transparent = true; // 开启透明度
        item.alphaTest = 0.5; // 只渲染透明度大于设定值的像素
        // item.depthTest = false; // 禁用深度测试
        item.depthWrite = false; // 禁止写入深度缓冲区
      }

      // 处理玻璃材质的显示
      if (item.name == '材質9') {
        console.log(item.name);

        item.transparent = true; // 开启透明度
        item.opacity = 0;
        item.reflectivity = 0.5; // 反射度
      }

      // if (item.userData.MMD.mapFileName == 'Tex/Tex_0009.png') {
      //   console.log(item.name);

      //   item.transparent = true; // 开启透明度
      //   item.opacity = 0.5;
      // }
    });

    mmd.traverse((obj: any) => {
      //加这句，让模型等每个部分都能产生阴影
      if (obj.isMesh) {
        // obj.castShadow = true;
        obj.receiveShadow = true;
      }
    });

    stageMesh = mmd;
    // scene.add(mmd);
  });

  // const modelUrl = '/model/huahuo/花火_无面具.pmx'
  // const modelUrl = '/model/huangquan/星穹铁道—黄泉（轴修复）.pmx'
  // const modelUrl = '/model/hutao/胡桃.pmx'
  // const modelUrl = '/model/funingna/【芙宁娜】.pmx'
  // const modelUrl = '/model/sanyueqi/三月七 1.0.pmx';
  // const modelUrl = '/model/阮·梅/阮·梅1.0.pmx';
  // const modelUrl = '/model/流萤/流萤3.0.pmx';
  // const modelUrl = '/model/藿藿/藿藿4.0.pmx';
  const modelUrl = '/model/花火/花火3.0.pmx';

  loader.load(
    modelUrl,
    (mesh: any) => {
      // mmd.scale.setScalar(0.2);

      // mmd使用了特殊的材质，无法通过traverse访问到
      mesh.material.forEach((item: any) => {
        item.emissive = new THREE.Color(0x000000); // 定义材质发射的颜色
        item.emissiveIntensity = 1; // 设置发光强度

        // item.castShadow = true;
        // item.receiveShadow = true;
      });

      mesh.traverse((obj: any) => {
        if (obj.isMesh) {
          obj.castShadow = true;
          // obj.receiveShadow = true;
        }
      });

      personModelMesh = mesh;

      MMDHelper = new MMDAnimationHelper();
      loadAnimationDance();
      // loader.loadAnimation('/animate/站立.vmd', mesh, (modelAnimation: any) => {
      //   modelAnimation.name = 'stand';
      //   MMDHelper.add(mesh, {
      //     animation: modelAnimation,
      //     physics: true,
      //   });
      //   MMDCanPlay = true;
      //   scene.add(mesh);
      // });
    },
    (xhr: any) => {
      // objBox.MMDManager.scale.x = xhr.loaded / xhr.total;
      console.log(xhr.loaded / xhr.total);
    },
    (err: any) => {
      console.error(err);
    }
  );
}

// 初始物体
function initObj() {
  let geometry = new THREE.BoxGeometry(200, 10, 200, 4, 4);
  let material = new THREE.MeshLambertMaterial({ color: 'rgb(45, 0, 50)' });
  objBox.stage = new THREE.Mesh(geometry, material);
  objBox.stage.castShadow = true;
  objBox.stage.receiveShadow = true;
  objBox.stage.position.set(0, -5, 0);
  scene.add(objBox.stage);

  // 镜面反射
  // const reflector = new Reflector(geometry, {
  //   clipBias: 0.003,
  //   textureWidth: window.innerWidth * window.devicePixelRatio,
  //   textureHeight: window.innerHeight * window.devicePixelRatio,
  //   color: 0x889999,
  // });
  // reflector.position.set(0, -5.01, 0);
  // scene.add(reflector);
}

// 初始辅助
function initHelper() {
  helperBox = {
    axesHelper: { helper: new THREE.AxesHelper(10000) }, // 坐标轴
    gridHelper: { helper: new THREE.GridHelper(200, 30, 'white', 'rgb(150, 150, 150)') }, // 网格
    spotLightHelper: { helper: new THREE.SpotLightHelper(lightBox.spotLight) }, // 聚光灯
  };
  // scene.add(helperBox.spotLightHelper.helper);
  // scene.add(helperBox.axesHelper.helper)
  scene.add(helperBox.gridHelper.helper);
}

// 初始监视器
function initStats() {
  stats = new Stats(); // 资源监视器
  stats.setMode(0);
  stats.domElement.id = 'three-stats';
  stats.domElement.style.position = 'absolute';
  stats.domElement.style.left = 'unset';
  stats.domElement.style.right = '300px';
  stats.domElement.style.top = '0px';
  statsRef.value.appendChild(stats.domElement);
}

function initOrbitControls() {
  orbitControls = new OrbitControls(camera, renderer.domElement);

  // 使用了 OrbitControls，camera 对象的 lookAt 方法失效
  // 这里通过调整  controls.target 控制初始摄像机的位置
  // controls.target = new THREE.Vector3(0, 10, 0);

  // // 获取当前摄像机的位置
  const cameraPosition = camera.position;
  orbitControls.target = new THREE.Vector3(cameraPosition.x, cameraPosition.y, 0);
}

let gui: any;
let guiParam: any;
// 初始控制台
function initGui() {
  gui = new GUI(); // 控制台
  guiParam = {
    // MMD设置
    action: 'stand',
    play: playControl.value,
    cameraPlay: true,
    cameraRotate: orbitControls.autoRotate,
    loadStageModel: false,
    // 辅助设置
    spotLightHelper: false,
    gridHelper: true,
    axesHelper: false,
    // 聚光灯设置
    castShadow: lightBox.spotLight.castShadow,
    lightRotate: lightRotate,
    color: lightBox.spotLight.color.getHex(),
    intensity: lightBox.spotLight.intensity,
    distance: lightBox.spotLight.distance,
    angle: (lightBox.spotLight.angle * 180) / Math.PI,
    penumbra: lightBox.spotLight.penumbra,
    decay: lightBox.spotLight.decay,
    positionX: lightBox.spotLight.position.x,
    positionY: lightBox.spotLight.position.y,
    positionZ: lightBox.spotLight.position.z,
    // 环境光设置
    ambientLightColor: lightBox.ambientLight.color.getHex(),
    ambientLightIntensity: lightBox.ambientLight.intensity,
  };

  guiMMDSetting();
  guiHelperSetting();
  guiAmbientLightSetting();
  guiLightSetting();
}

// MMD设置
function guiMMDSetting() {
  let MMDSetting = gui.addFolder('MMD设置');
  const actionOptions = {
    站立: 'stand',
    '跳舞（极乐净土）': 'dance',
  };
  MMDSetting.add(guiParam, 'action', actionOptions)
    .name('动作')
    .onChange((value: string) => {
      console.log(value);
      switch (value) {
        case 'stand':
          MMDHelper.doAnimation('stand');
          break;
        case 'dance':
          loadAnimationDance();
          break;
      }
    });
  MMDSetting.add(guiParam, 'play')
    .name('播放（Play）')
    .onChange((data: any) => {
      playControl.value = data;
    });
  MMDSetting.add(guiParam, 'cameraPlay')
    .name('镜头播放（CameraPlay）')
    .onChange((data: any) => {
      if (!data) {
        camera.position.x = 0;
        camera.position.y = 15;
        camera.position.z = 25;
        camera.up.x = 0;
        camera.up.y = 1;
        camera.up.z = 0;
      }
      MMDHelper.playCamera = data;
    });
  MMDSetting.add(guiParam, 'cameraRotate')
    .name('镜头旋转（CameraRotate）')
    .onChange((data: any) => {
      orbitControls.autoRotate = data;
    });
  MMDSetting.add(guiParam, 'loadStageModel').name('加载舞台模型（LoadStageModel）');
  MMDSetting.open();
}

// 辅助设置
function guiHelperSetting() {
  let helperSetting = gui.addFolder('辅助设置（Helper Setting）');
  helperSetting
    .add(guiParam, 'gridHelper')
    .name('网格（GridHelper）')
    .onChange((data: any) => {
      if (data) {
        scene.add(helperBox.gridHelper.helper);
      } else {
        scene.remove(helperBox.gridHelper.helper);
      }
    });
  helperSetting
    .add(guiParam, 'axesHelper')
    .name('坐标轴（AxesHelper）')
    .onChange((data: any) => {
      if (data) {
        scene.add(helperBox.axesHelper.helper);
      } else {
        scene.remove(helperBox.axesHelper.helper);
      }
    });
  helperSetting
    .add(guiParam, 'spotLightHelper')
    .name('聚光灯辅助线（SpotLightHelper）')
    .onChange((data: any) => {
      if (data) {
        scene.add(helperBox.spotLightHelper.helper);
      } else {
        scene.remove(helperBox.spotLightHelper.helper);
      }
    });
}

// 聚光灯设置
function guiLightSetting() {
  let lightSetting = gui.addFolder('聚光灯设置（Spot Light Setting）');
  lightSetting
    .add(guiParam, 'castShadow')
    .name('投影（CastShadow）')
    .onChange((data: any) => {
      lightBox.spotLight.castShadow = data;
    });
  lightSetting
    .add(guiParam, 'lightRotate')
    .name('灯光旋转（LightRotate）')
    .onChange((data: any) => {
      lightRotate = data;
    });
  lightSetting
    .addColor(guiParam, 'color', -500, 500)
    .name('颜色（Color）')
    .onChange((data: any) => {
      lightBox.spotLight.color.setHex(data);
      helperBox.spotLightHelper.helper.update();
    });
  lightSetting
    .add(guiParam, 'intensity', 0, 1.5)
    .name('强度（Intensity）')
    .onChange((data: any) => {
      lightBox.spotLight.intensity = data;
      helperBox.spotLightHelper.helper.update();
    });
  lightSetting
    .add(guiParam, 'distance', 0, 1000, 1)
    .name('距离（Distance）')
    .onChange((data: any) => {
      lightBox.spotLight.distance = data;
      helperBox.spotLightHelper.helper.update();
    });
  lightSetting
    .add(guiParam, 'angle', 0, 60, 1)
    .name('角度（Angle）')
    .onChange((data: any) => {
      lightBox.spotLight.angle = (Math.PI / 180) * data;
      helperBox.spotLightHelper.helper.update();
    });
  lightSetting
    .add(guiParam, 'penumbra', 0, 1, 0.1)
    .name('半影（Penumbra）')
    .onChange((data: any) => {
      lightBox.spotLight.penumbra = data;
      helperBox.spotLightHelper.helper.update();
    });
  lightSetting
    .add(guiParam, 'decay', 0, 5, 0.1)
    .name('衰减（Decay）')
    .onChange((data: any) => {
      lightBox.spotLight.decay = data;
      helperBox.spotLightHelper.helper.update();
    });
  lightSetting
    .add(guiParam, 'positionX', -500, 500, 1)
    .name('位置X轴（PositionX）')
    .onChange((data: any) => {
      lightBox.spotLight.position.x = data;
      helperBox.spotLightHelper.helper.update();
    });
  lightSetting
    .add(guiParam, 'positionY', -500, 500, 1)
    .name('位置Y轴（PositionY）')
    .onChange((data: any) => {
      lightBox.spotLight.position.y = data;
      helperBox.spotLightHelper.helper.update();
    });
  lightSetting
    .add(guiParam, 'positionZ', -500, 500, 1)
    .name('位置Z轴（PositionZ）')
    .onChange((data: any) => {
      lightBox.spotLight.position.z = data;
      helperBox.spotLightHelper.helper.update();
    });
  lightSetting.open();
}

// 环境光设置
function guiAmbientLightSetting() {
  let ambientLightSetting = gui.addFolder('环境光设置（Ambient Light Setting）');
  ambientLightSetting
    .addColor(guiParam, 'ambientLightColor', -500, 500)
    .name('颜色（Color）')
    .onChange((data: any) => {
      lightBox.ambientLight.color.setHex(data);
    });
  ambientLightSetting
    .add(guiParam, 'ambientLightIntensity', 0, 3)
    .name('强度（Intensity）')
    .onChange((data: any) => {
      lightBox.ambientLight.intensity = data;
    });
  ambientLightSetting.open();
}

// 加载跳舞动作
function loadAnimationDance() {
  const actionUrl = '/animate/极乐净土.vmd';
  // const cameraUrl = '/camera/极乐净土.vmd';
  const cameraUrl = '/camera/極楽净土 镜头 by 永远赤红的幼月.vmd';
  const musicUrl = '/music/极乐净土.mp3';

  if (actionUrl) {
    // 载入模型与动画
    loader.loadAnimation(actionUrl, personModelMesh, (modelAnimation: any) => {
      modelAnimation.name = 'dance';
      MMDHelper.add(personModelMesh, {
        animation: modelAnimation,
        physics: true,
      });
      // 载入相机
      if (cameraUrl) {
        loader.loadAnimation(cameraUrl, camera, (cameraAnimation: any) => {
          MMDCamare = cameraAnimation;
          MMDHelper.add(camera, { animation: cameraAnimation });
        });
      }
      // 载入音乐
      if (musicUrl) {
        const listener = new THREE.AudioListener();
        const audioLoader = new THREE.AudioLoader();
        audioLoader.load(
          musicUrl,
          (buffer: any) => {
            audio = new THREE.Audio(listener).setBuffer(buffer);
            MMDHelper.add(audio, { duration: buffer.duration });
            scene.add(personModelMesh);
            MMDCanPlay = true;
          },
          null,
          (err: any) => {
            console.log(err);
          }
        );
      }
    });
  }
}

// 更新场景
function updateRenderer() {
  animationFrame = requestAnimationFrame(updateRenderer);

  const delta = clock.getDelta();
  orbitControls.update(delta);

  if (lightRotate) {
    animation();
  }

  // 是否开始动作
  if (MMDCanPlay && playControl.value) {
    MMDHelper.update(delta);
  }

  // 控制是否加载舞台
  if (guiParam.loadStageModel) {
    if (stageMesh && !stageMesh.parent) {
      scene.add(stageMesh);
    }
  } else {
    scene.remove(stageMesh);
  }

  renderer.render(scene, camera);
  stats.update();

  // water.material.uniforms['time'].value += 1.0 / 60.0;
}

function animation() {
  let r = 80;
  let deg = Date.now() * 0.001;
  lightBox.spotLight.position.x = -Math.cos(deg) * r;
  lightBox.spotLight.position.z = Math.sin(deg) * r;
  helperBox.spotLightHelper.helper.update();
}

// 清空物体缓存
function clearObjCache(obj: any) {
  obj.geometry.dispose();
  obj.material.dispose();
}

// 清空缓存
function clearCache() {
  // 渲染器缓存
  renderer.dispose();
  renderer.forceContextLoss();
  renderer.clear(true, true, true);
  renderer.domElement = null;
  // 场景缓存
  scene.dispose();
  // 几何体缓存
  clearObjCache(objBox.stage);
  if (objBox.knife) {
    objBox.knife.children.forEach((elem: any) => {
      clearObjCache(elem);
    });
  }
  // gui
  gui.destroy();
}

watch(playControl, (newValue: any, oldValue: any) => {
  if (newValue) {
    audio && audio.play();
  } else {
    audio && audio.pause();
  }
});

onMounted(() => {
  init();
});

onBeforeUnmount(() => {
  const statsEl = document.getElementById('three-stats');
  statsEl && document.body.removeChild(statsEl);
  clearCache();
  audio && audio.pause();
  MMDHelper.remove(audio);
  renderer = null;
  scene = null;
  camera = null;
  stats = null;
  helperBox = null;
  managerBox = null;
  objBox = null;
  clock = null;
  orbitControls = null;
  cancelAnimationFrame(animationFrame);
});
</script>

<style lang="scss" scoped>
canvas {
  display: block;
  width: 100%;
  height: 100%;
}
</style>
