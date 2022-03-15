<template>
  <div class="main" ref="main"></div>
</template>

<script>
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
import { DRACOLoader } from "three/examples/jsm/loaders/DRACOLoader";
import { GUI } from "three/examples/jsm/libs/lil-gui.module.min.js";
import {FontLoader} from "three/examples/jsm/loaders/FontLoader";
import {TextGeometry} from "three/examples/jsm/geometries/TextGeometry"
let scene, camera, light, renderer, controls;

let ray = new THREE.Raycaster();
let mousePosition = new THREE.Vector2();
export default {
  data() {
    return {
      waterParams: {
        color: "#ffffff",
        scale: 2,
        flowX: 1,
        flowY: 1,
      },
    };
  },
  mounted() {
    this.init();
    this.onWindowSize();
    this.aniamtion();
  },
  methods: {

    init() {
      scene = new THREE.Scene();
      scene.fog = new THREE.Fog("#f1b67f", 0, 400);

      camera = new THREE.PerspectiveCamera(
        95,
        window.innerWidth / window.innerHeight,
        1,
        1000
      );
      camera.position.set(1, 8, 5);
      scene.add(camera);

      // 环境光
      light = new THREE.AmbientLight();
      light.position.set(5, 10, 5);
      scene.add(light);

      // 平行光
      let light = new THREE.DirectionalLight("#304758", 1);
      light.castShadow = true;
      light.position.set(5, 10, 55);
      scene.add(light);

      renderer = new THREE.WebGLRenderer({
        antialias: true
      });
      renderer.setClearColor("#f1b67f");
      renderer.shadowMap.enabled = true;
      renderer.setSize(window.innerWidth, window.innerHeight);
      this.$refs.main.appendChild(renderer.domElement);

      // 添加地板
      let floorGeo = new THREE.BoxBufferGeometry(800, 1, 800);
      let floorMesh = new THREE.MeshPhongMaterial({ color: "#eacdd1" });
      let floor = new THREE.Mesh(floorGeo, floorMesh);

      floor.receiveShadow = true;
      floor.castShadow = true;
      scene.add(floor);

      // 导出模型
      this.exportModlue();

      // 设置色彩通道
      renderer.outputEncoding = THREE.sRGBEncoding;

      // 相机控制
      controls = new OrbitControls(camera, renderer.domElement);
      controls.maxPolarAngle = Math.PI / 2 - 0.05;

    },
    aniamtion() {
      renderer.render(scene, camera);
      requestAnimationFrame(this.aniamtion);
    },
    onWindowSize() {
      window.addEventListener("resize", (e) => {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();

        renderer.setSize(window.innerWidth, window.innerHeight);
      });
    },
    exportModlue() {
      let loader = new GLTFLoader();
      let draLoader = new DRACOLoader();
      draLoader.setDecoderPath("three/examples/js/libs/draco/");
      loader.setDRACOLoader(draLoader);

      loader.load("assets/modules/通风系统-可视化.glb", (gltf) => {
        scene.add(gltf.scene);
        gltf.scene.position.set(0, 1, 0);
      });
    },
  },
};
</script>

<style>
</style>