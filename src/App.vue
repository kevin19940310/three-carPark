<script setup>
import * as Three from "three";
import * as YUKA from "yuka";

import { Pathfinding } from "three-pathfinding";

// 导入控制器
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
import { DRACOLoader } from "three/examples/jsm/loaders/DRACOLoader.js";

Three.ColorManagement.enabled = true;

const Color = {
  GROUND: 0x606060,
  NAVMESH: 0xffffff,
};

const ZONE = "level1";

const mouse = new Three.Vector2();
const mouseDown = new Three.Vector2();
const pathfinder = new Pathfinding();
const raycaster = new Three.Raycaster();

let level, navmesh;

let groupID = 0;
let path;

const playerPosition = new Three.Vector3(0, 0, 0);
const targetPosition = new Three.Vector3();

// 创建场景
const scene = new Three.Scene();
// 创建相机
const camera = new Three.PerspectiveCamera(
  75,
  window.innerWidth / window.innerHeight,
  0.1,
  1000
);
camera.position.set(0, 10, 30);
camera.lookAt(0, 0, 0);

// 创建渲染器
const Renderer = new Three.WebGLRenderer({
  antialias: true,
});
Renderer.shadowMap.enabled = true;
Renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(Renderer.domElement);

const ambient = new Three.AmbientLight(0x101030);
scene.add(ambient);

// 创建灯光
const directionalLight = new Three.DirectionalLight(0xffeedd);
directionalLight.position.set(0, 0.5, 0.5);
scene.add(directionalLight);

const controls = new OrbitControls(camera, Renderer.domElement);
controls.enableDamping = true;

// 加载模型
const loader = new GLTFLoader();
const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath("./draco/");
loader.setDRACOLoader(dracoLoader);

loader.load("/src/mode/modelMap2.gltf", function (gltf) {
  gltf.scene.traverse((child) => {
    if (child.isMesh && child.name === "Navmesh") {
      navmesh = child;
    }
  });

  const zone = Pathfinding.createZone(navmesh.geometry);
  pathfinder.setZoneData(ZONE, zone);
  navmesh.parent.remove(navmesh);

  const nav1 = new Three.Mesh(
    navmesh.geometry,
    new Three.MeshBasicMaterial({
      color: 0x808080,
      wireframe: false,
      opacity: 0,
    })
  );

  const nav2 = new Three.Mesh(
    navmesh.geometry,
    new Three.MeshBasicMaterial({
      color: Color.NAVMESH,
      opacity: 0,
      transparent: true,
    })
  );
  scene.add(nav1);
  scene.add(nav2);
  scene.add(gltf.scene);
});

let plane;
loader.load("/src/mode/carRoom.gltf", function (gltf) {
  // console.log(gltf.scene);
  plane = gltf.scene;
  plane.traverse((child) => {
    if (child.isMesh) {
      child.receiveShadow = true;
      child.castShadow = true;
      // child.position.z = child.position.z - 8;
      child.position.y = child.position.y + 0.2;
    }
  });
  scene.add(plane);
});

// 创建开始椎体
const startGeometry = new Three.ConeGeometry(0.2, 1, 32);
startGeometry.rotateX(Math.PI);
const startMaterial = new Three.MeshStandardMaterial({ color: 0xff0000 });
const startMesh = new Three.Mesh(startGeometry, startMaterial);
// startMesh.scale.set(0.1, 0.1, 0.1);
startMesh.position.set(-5, 1, 0);
startMesh.receiveShadow = true;
startMesh.castShadow = true;
scene.add(startMesh);

// 创建结束椎体
const endGeometry = new Three.ConeGeometry(0.2, 1, 32);
endGeometry.rotateX(Math.PI);
Pathfinding.createZone(endGeometry);
const endMaterial = new Three.MeshStandardMaterial({ color: 0x049ef4 });
const endMesh = new Three.Mesh(endGeometry, endMaterial);
// endMesh.scale.set(0.1, 0.1, 0.1);
endMesh.position.set(-5, 1, 0);
endMesh.receiveShadow = true;
endMesh.castShadow = true;
scene.add(endMesh);

window.addEventListener("pointerdown", (event) => {
  mouseDown.x = (event.clientX / window.innerWidth) * 2 - 1;
  mouseDown.y = -(event.clientY / window.innerHeight) * 2 + 1;
});

window.addEventListener("pointerup", (event) => {
  mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
  mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

  if (
    Math.abs(mouseDown.x - mouse.x) > 0 ||
    Math.abs(mouseDown.y - mouse.y) > 0
  )
    return;

  camera.updateMatrixWorld();

  raycaster.setFromCamera(mouse, camera);
  const intersects = raycaster.intersectObject(navmesh);
  const point = intersects[0].point;
  if (event.metaKey || event.ctrlKey || event.button === 2) {
    playerPosition.copy(intersects[0].point);
    startMesh.position.copy({
      x: point.x,
      y: 1,
      z: point.z,
    });
  } else {
    targetPosition.copy(intersects[0].point);
    endMesh.position.copy({
      x: point.x,
      y: 1,
      z: point.z,
    });
  }
  // console.log(playerPosition, targetPosition, ZONE, groupID, "22222");
  // groupID = pathfinder.getGroup(ZONE, playerPosition);
  if (!playerPosition.x || !targetPosition.x) return;
  path = pathfinder.findPath(playerPosition, targetPosition, ZONE, groupID);
  path.unshift(playerPosition);
  showPathLine(path);
});

let line;
function showPathLine(path) {
  if (line) {
    scene.remove(line);
    line.geometry.dispose();
    line.material.dispose();
  }
  const positions = [];
  for (let i = 0; i < path.length; i++) {
    positions.push(path[i].x, 0.2, path[i].z);
  }
  const geometry = new Three.BufferGeometry();
  geometry.setAttribute(
    "position",
    new Three.Float32BufferAttribute(positions, 3)
  );
  const material = new Three.LineBasicMaterial({
    color: 0xffff00,
    linewidth: 3,
  });
  line = new Three.Line(geometry, material);
  scene.add(line);
}

requestAnimationFrame(function animate() {
  controls.update();
  Renderer.render(scene, camera);
  requestAnimationFrame(animate);
});
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
