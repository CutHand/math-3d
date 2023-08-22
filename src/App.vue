<script setup>
import { onMounted, ref, reactive } from "vue";
import * as THREE from "three"
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';  // 控制器
import { GUI } from "three/examples/jsm/libs/lil-gui.module.min.js"  // GUI
import * as TWEEN from "three/examples/jsm/libs/tween.module"  // TWEEN
// import { Tween } from "gsap/gsap-core";

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
// 创建透视相机
const perspectiveCamera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 1, 10000)

camera = perspectiveCamera
camera.position.set(0, 400, 400);
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
// constrols.autoRotate = true

// 添加世界坐标系
// const axesHelper = new THREE.AxesHelper(100)
// scene.add(axesHelper)

// 设置全屏按钮
let eventObj = {
  View: "透视视图",
  EnterFullScreen() {
    document.body.requestFullscreen()
    console.log("enter fullscreen")
  },
  ExitFullscreen() {
    document.exitFullscreen()
    console.log("exit fullscreen")
  },
  ClearAllVoxel() {
    console.log(objects)
    objects.forEach((mesh, index) => {
      if (mesh.name !== "plane") {
        scene.remove(mesh)
      }
    })
    objects.splice(1, objects.length - 1);
    render()
  },
  FrontCamera() {
    const camera_tween = new TWEEN.Tween(camera.position)
    camera_tween.to({ x: 0, y: 0, z: 400 }, 1000)
    camera_tween.start()
  },
  BackCamera() {
    const camera_tween = new TWEEN.Tween(camera.position)
    camera_tween.to({ x: 0, y: 0, z: -400 }, 1000)
    camera_tween.start()
  },
  RightCamera() {
    const camera_tween = new TWEEN.Tween(camera.position)
    camera_tween.to({ x: 400, y: 0, z: 0 }, 1000)
    camera_tween.start()
  },
  LeftCamera() {
    const camera_tween = new TWEEN.Tween(camera.position)
    camera_tween.to({ x: -400, y: 0, z: 0 }, 1000)
    camera_tween.start()
  },
  UpCamera() {
    const camera_tween = new TWEEN.Tween(camera.position)
    camera_tween.to({ x: 0, y: 400, z: 0 }, 1000)
    camera_tween.start()
  },
  DownCamera() {
    const camera_tween = new TWEEN.Tween(camera.position)
    camera_tween.to({ x: 0, y: -400, z: 0 }, 1000)
    camera_tween.start()
  }

}
const gui = new GUI()
gui.add(eventObj, "EnterFullScreen").name("进入全屏")
gui.add(eventObj, "ExitFullscreen").name("退出全屏")
gui.add(eventObj, "ClearAllVoxel").name("清空所有")
gui.add(constrols, "autoRotate").name("自动旋转")
gui.add(eventObj, "UpCamera").name("看向上面")
gui.add(eventObj, "DownCamera").name("看向下面")
gui.add(eventObj, "LeftCamera").name("看向左面")
gui.add(eventObj, "RightCamera").name("看向右面")
gui.add(eventObj, "FrontCamera").name("看向前面")
gui.add(eventObj, "BackCamera").name("看向后面")
// gui.add(eventObj, "View", ['透视视图', '正交视图']).name("视图").onChange(view => {
//   console.log(view)
//   if (view == '正交视图') {
//     console.log("正交切换")
//     //1.计算透视相机到场景 scene 的深度距离 depth
//     let target = scene.position.clone();;
//     let camPos = camera.position.clone();
//     let depth = camPos.sub(target).length();
//     //2.得到透视相机的宽高比和垂直可视角度
//     let aspect = camera.aspect;
//     let fov = camera.fov;
//     //3.根据上述变量计算正交投影相机的视口矩形
//     let top_ortho = depth * Math.atan((Math.PI / 180) * (fov) / 2);
//     let right_ortho = top_ortho * aspect;
//     let bottom_ortho = - top_ortho;
//     let left_ortho = - right_ortho;
//     //4.最后创建正交投影相机
//     let near = camera.near;
//     let far = camera.far;
//     const orthographicCamera = new THREE.OrthographicCamera(
//       left_ortho, right_ortho,
//       top_ortho, bottom_ortho,
//       near, far
//     );
//     scene.remove(camera);
//     camera = orthographicCamera;
//   } else {
//     scene.remove(camera);
//     camera = perspectiveCamera;
//   }
//   scene.add(camera);
//   render()
// })

// roll-over helpers
const rollOverGeo = new THREE.BoxGeometry(50, 50, 50);
rollOverMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000, opacity: 0.5, transparent: true });
rollOverMesh = new THREE.Mesh(rollOverGeo, rollOverMaterial);
scene.add(rollOverMesh);
// cubes

const map = new THREE.TextureLoader().load('textures/square-outline-textured.png');
map.colorSpace = THREE.SRGBColorSpace;
cubeGeo = new THREE.BoxGeometry(50, 50, 50);
cubeMaterial = new THREE.MeshLambertMaterial({ color: 0x5fd5f0, map: map });

raycaster = new THREE.Raycaster();
pointer = new THREE.Vector2();
const geometry = new THREE.PlaneGeometry(1000, 1000);
geometry.rotateX(- Math.PI / 2);

plane = new THREE.Mesh(geometry, new THREE.MeshBasicMaterial({ visible: false }));
plane.name = "plane"
scene.add(plane);
objects.push(plane);

// lights

const ambientLight = new THREE.AmbientLight(0x606060, 3);
scene.add(ambientLight);

const directionalLight = new THREE.DirectionalLight(0xffffff, 3);
directionalLight.position.set(1, 0.75, 0.5).normalize();
scene.add(directionalLight);

renderer.domElement.addEventListener('pointermove', onPointerMove);
let mouseDownTime = 0;
let mouseDownPositionXandY = 0
renderer.domElement.addEventListener('pointerdown', (event) => {
  mouseDownPositionXandY = event.clientX + event.clientY
  console.log(`按下时的x坐标${event.clientX}`)
  console.log(`按下时的y坐标${event.clientY}`)
  mouseDownTime = Date.now();  // 获取当前时间戳
});
renderer.domElement.addEventListener('pointerup', (event) => {
  const mouseUpTime = Date.now();  // 获取当前时间戳
  const timeDifference = mouseUpTime - mouseDownTime;  // 计算时间差（毫秒）
  console.log(`松开时的x坐标${event.clientX}`)
  console.log(`松开时的y坐标${event.clientY}`)
  console.log(`按下和松开之间的时间差为 ${timeDifference} 毫秒`);
  if (timeDifference < 200 && Math.abs(mouseDownPositionXandY - event.clientX - event.clientY) < 20) {
    onPointerDown(event)
  }
});
document.addEventListener('keydown', onDocumentKeyDown);
document.addEventListener('keyup', onDocumentKeyUp);
// 渲染函数
const render = () => {
  renderer.setAnimationLoop(render)  // 可用来代替requestAnimationFrame的内置函数. 对于WebXR项目，必须使用此函数。
  // requestAnimationFrame(render)  // window的方法，诉浏览器——你希望执行一个动画，并且要求浏览器在下次重绘之前调用指定的回调函数更新动画。
  constrols && constrols.update()  // 更新控制器
  TWEEN.update()  // TWEEN动画更新
  renderer.render(scene, camera)  // 渲染
}
function onPointerMove(event) {
  pointer.set((event.clientX / window.innerWidth) * 2 - 1, - (event.clientY / window.innerHeight) * 2 + 1);  // // 将鼠标位置归一化为设备坐标。x 和 y 方向的取值范围是 (-1 to +1)
  raycaster.setFromCamera(pointer, camera);  // 通过摄像机和鼠标位置更新射线
  const intersects = raycaster.intersectObjects(objects, false);  // 计算物体和射线的焦点
  if (intersects.length > 0) {
    const intersect = intersects[0];
    // console.log(intersect.point)
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
      voxel.name = "voxel"
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