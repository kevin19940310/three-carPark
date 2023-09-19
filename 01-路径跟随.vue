<script setup>
import * as THREE from "three";
import * as YUKA from "yuka";
// 导入控制器
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";

// 创建场景
const scene = new THREE.Scene();
// 创建相机
const camera = new THREE.PerspectiveCamera(
  75,
  window.innerWidth / window.innerHeight,
  0.1,
  1000
);
camera.position.set(0, 10, 20);
camera.lookAt(0, 0, 0);

// 创建渲染器
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.shadowMap.enabled = true;
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// 创建地面
const planeGeometry = new THREE.PlaneGeometry(50, 50);
const planeMaterial = new THREE.MeshStandardMaterial({ color: 0x999999 });
const plane = new THREE.Mesh(planeGeometry, planeMaterial);
plane.receiveShadow = true;
plane.rotation.x = -Math.PI / 2;
plane.position.y = -0.5;
scene.add(plane);

// 创建灯光
const light = new THREE.SpotLight(0xffffff, 3, 100, Math.PI / 6, 0.5);
light.position.set(10, 40, 10);
light.castShadow = true;
scene.add(light);

// 创建控制器
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

const time = new YUKA.Time();
requestAnimationFrame(function animate() {
  const delta = time.update().getDelta();
  controls.update();
  entityManager.update(delta);
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
});

// 创建椎体
const coneGeometry = new THREE.ConeGeometry(0.2, 1, 32);
coneGeometry.rotateX(Math.PI / 2);
const coneMaterial = new THREE.MeshStandardMaterial({ color: 0xff0000 });
const cone = new THREE.Mesh(coneGeometry, coneMaterial);
cone.matrixAutoUpdate = false;
cone.receiveShadow = true;
cone.castShadow = true;
scene.add(cone);

// 创建yuka的车辆
const vehicle = new YUKA.Vehicle();
vehicle.maxSpeed = 5;
// 设置车辆的渲染对象
vehicle.setRenderComponent(cone, callback);
function callback(entity, renderComponent) {
  console.log(entity, renderComponent);
  // renderComponent.position.copy(entity.position);
  // renderComponent.quaternion.copy(entity.rotation);
  renderComponent.matrix.copy(entity.worldMatrix);
}

// 创建yuka的路径
const path = new YUKA.Path();
path.add(new YUKA.Vector3(0, 0, 0));
path.add(new YUKA.Vector3(0, 0, 10));
path.add(new YUKA.Vector3(10, 0, 10));
path.add(new YUKA.Vector3(10, 0, 0));
path.add(new YUKA.Vector3(0, 0, 0));
// 设置路径的循环模式
path.loop = true;

// 将路径当前的位置设置为车辆的位置
vehicle.position.copy(path.current());

// 跟随路径的行为
const followPathBehavior = new YUKA.FollowPathBehavior(path);
vehicle.steering.add(followPathBehavior);

// 创建对实体管理对象
const entityManager = new YUKA.EntityManager();
entityManager.add(vehicle);

showPathLine(path);
function showPathLine(path) {
  console.log(path);
  const positions = [];
  for (let i = 0; i < path._waypoints.length; i++) {
    positions.push(
      path._waypoints[i].x,
      path._waypoints[i].y,
      path._waypoints[i].z
    );
  }
  const geometry = new THREE.BufferGeometry();
  geometry.setAttribute(
    "position",
    new THREE.Float32BufferAttribute(positions, 3)
  );
  const material = new THREE.LineBasicMaterial({ color: 0x0000ff });
  const line = new THREE.Line(geometry, material);
  scene.add(line);
}
</script>

<template>
  <div></div>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

canvas {
  width: 100vw;
  height: 100vh;
  position: fixed;
  left: 0;
  top: 0;
}
</style>
