<script setup>
import { onMounted, ref, reactive } from "vue";
import * as THREE from "three"
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';  // 控制器
import { GUI } from "three/examples/jsm/libs/lil-gui.module.min.js"  // GUI

let camera, scene, renderer;
let plane;
let pointer, raycaster, isShiftDown = false;
let rollOverMesh, rollOverMaterial;
let cubeGeo, cubeMaterial;

const objects = []
let canvasDom = ref(null)

// 创建场景
scene = new THREE.Scene();
scene.background = new THREE.Color(0x333333)
scene.enviroment = new THREE.Color("#333")
// 创建相机
camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 1, 10000)
camera.position.set(500, 800, 1300);
camera.lookAt(0, 0, 0);
// 创建渲染器
renderer = new THREE.WebGLRenderer({
  antialias: true  // 抗锯齿
})
renderer.setPixelRatio(window.devicePixelRatio);  // 设置设备像素比。通常用于避免HiDPI设备上绘图模糊
renderer.setSize(window.innerWidth, window.innerHeight)
// renderer.setClearColor("#ccc", 0.5)    // 设置颜色及其透明度 .setClearColor ( color : Color, alpha : Float ) : undefined

// 设置画布的响应式布局
window.addEventListener("resize", () => {
  renderer.setSize(window.innerWidth, window.innerHeight)
  camera.aspect = window.innerWidth / window.innerHeight
  camera.updateProjectionMatrix()  // 更新相机投影矩阵
})

// 添加网格地面
const gridHelper = new THREE.GridHelper(1000, 20)
gridHelper.material.opacity = 0.5
gridHelper.material.transparent = true
scene.add(gridHelper)

// 添加控制器
const constrols = new OrbitControls(camera, renderer.domElement)
constrols.update()
constrols.autoRotate = true

// 添加世界坐标系
const axesHelper = new THREE.AxesHelper(100)
scene.add(axesHelper)

// 设置全屏按钮
let eventObj = {
  EnterFullScreen() {
    document.body.requestFullscreen()
    console.log("enter fullscreen")
  },
  ExitFullscreen() {
    document.exitFullscreen()
    console.log("exit fullscreen")
  }
}
const gui = new GUI()
gui.add(eventObj, "EnterFullScreen").name("进入全屏")
gui.add(eventObj, "ExitFullscreen").name("退出全屏")
gui.add(constrols, "autoRotate").name("自动旋转")

// roll-over helpers
const rollOverGeo = new THREE.BoxGeometry(50, 50, 50);
rollOverMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000, opacity: 0.5, transparent: true });
rollOverMesh = new THREE.Mesh(rollOverGeo, rollOverMaterial);
scene.add(rollOverMesh);
// cubes

const map = new THREE.TextureLoader().load('textures/square-outline-textured.png');
map.colorSpace = THREE.SRGBColorSpace;
cubeGeo = new THREE.BoxGeometry(50, 50, 50);
cubeMaterial = new THREE.MeshLambertMaterial({ color: 0xfeb74c, map: map });

raycaster = new THREE.Raycaster();
pointer = new THREE.Vector2();
const geometry = new THREE.PlaneGeometry(1000, 1000);
geometry.rotateX(- Math.PI / 2);

plane = new THREE.Mesh(geometry, new THREE.MeshBasicMaterial({ visible: false }));
scene.add(plane);
objects.push(plane);

// lights

const ambientLight = new THREE.AmbientLight(0x606060, 3);
scene.add(ambientLight);

const directionalLight = new THREE.DirectionalLight(0xffffff, 3);
directionalLight.position.set(1, 0.75, 0.5).normalize();
scene.add(directionalLight);

document.addEventListener('pointermove', onPointerMove);
document.addEventListener('pointerdown', onPointerDown);
document.addEventListener('keydown', onDocumentKeyDown);
document.addEventListener('keyup', onDocumentKeyUp);
// 渲染函数
const render = () => {
  renderer.setAnimationLoop(render)  // 可用来代替requestAnimationFrame的内置函数. 对于WebXR项目，必须使用此函数。
  // requestAnimationFrame(render)  // window的方法，诉浏览器——你希望执行一个动画，并且要求浏览器在下次重绘之前调用指定的回调函数更新动画。
  constrols && constrols.update()  // 更新控制器
  renderer.render(scene, camera)  // 渲染
}
function onPointerMove(event) {
  pointer.set((event.clientX / window.innerWidth) * 2 - 1, - (event.clientY / window.innerHeight) * 2 + 1);
  raycaster.setFromCamera(pointer, camera);
  const intersects = raycaster.intersectObjects(objects, false);
  if (intersects.length > 0) {
    const intersect = intersects[0];
    rollOverMesh.position.copy(intersect.point).add(intersect.face.normal);
    rollOverMesh.position.divideScalar(50).floor().multiplyScalar(50).addScalar(25);
    render();
  }
}
function onPointerDown(event) {
  pointer.set((event.clientX / window.innerWidth) * 2 - 1, - (event.clientY / window.innerHeight) * 2 + 1);
  raycaster.setFromCamera(pointer, camera);
  const intersects = raycaster.intersectObjects(objects, false);
  if (intersects.length > 0) {
    const intersect = intersects[0];
    // delete cube
    if (isShiftDown) {
      if (intersect.object !== plane) {
        scene.remove(intersect.object);
        objects.splice(objects.indexOf(intersect.object), 1);
      }
      // create cube
    } else {
      const voxel = new THREE.Mesh(cubeGeo, cubeMaterial);
      voxel.position.copy(intersect.point).add(intersect.face.normal);
      voxel.position.divideScalar(50).floor().multiplyScalar(50).addScalar(25);
      scene.add(voxel);
      objects.push(voxel);
    }
    render();
  }
}
function onDocumentKeyDown(event) {
  switch (event.keyCode) {
    case 16: isShiftDown = true; break;
  }
}
function onDocumentKeyUp(event) {
  switch (event.keyCode) {
    case 16: isShiftDown = false; break;
  }
}
// 生命周期，挂载时
onMounted(() => {
  canvasDom.value.appendChild(renderer.domElement)  // 渲染器插入到Dom中
  render()
})
</script>
<template>
  <div class="home">
    <div class="canvas-container" ref="canvasDom"></div>
  </div>
</template>
<style scoped></style>