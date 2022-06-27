<template>
  <div class="main">
    <footer class="foot">
    </footer>
    <script id="vertexShader1" type="x-shader/x-vertex">
      varying vec3 vNormal;
      varying vec3 vPositionNormal;
      void main()
      {
        vNormal = normalize( normalMatrix * normal );
        vPositionNormal = normalize(( modelViewMatrix * vec4(position, 1.0) ).xyz);
        gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
      }
    </script>
    <!-- fragment shader a.k.a. pixel shader -->
    <script id="fragmentShader1" type="x-shader/x-vertex">
      uniform vec3 glowColor;
      uniform float b;
      uniform float p;
      uniform float s;
      varying vec3 vNormal;
      varying vec3 vPositionNormal;
      void main()
      {
        float a = pow( b + s * abs(dot(vNormal, vPositionNormal)), p );
        gl_FragColor = vec4( glowColor, a );
      }
    </script>
    <!--shader-->
    <script type="x-shader/x-vertex" id="vertexshader">

      varying vec2 vUv;

      void main() {

      	vUv = uv;

      	gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );

      }
    </script>

    <script type="x-shader/x-fragment" id="fragmentshader">

      uniform sampler2D baseTexture;
      uniform sampler2D bloomTexture;

      varying vec2 vUv;

      void main() {

      	gl_FragColor = ( texture2D( baseTexture, vUv ) + vec4( 1.5 ) * texture2D( bloomTexture, vUv ) );

      }
    </script>
  </div>
</template>
<script>
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import GUI from "three/examples/jsm/libs/lil-gui.module.min.js";
// 效果组合器对象
import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer";
// 在指定的相机和场景上渲染新的场景，但是不会输出场景
import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass.js";
// 使用该通道你可以传入一个自定义的着色器，用来生成高级的、自定义的后期处理通道
import { ShaderPass } from "three/examples/jsm/postprocessing/ShaderPass.js";
// import {} from "three/examples/jsm/postprocessing/F"
// 形成高亮虚幻的效果
import { UnrealBloomPass } from "three/examples/jsm/postprocessing/UnrealBloomPass.js";

import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
import { DRACOLoader } from "three/examples/jsm/loaders/DRACOLoader";

// 基础元素
let scene, camera, renderer, controler, curve;


let texture = [];

//光源
let { spotLight, direcLight } = {
  spotLight, // 聚光灯
  direcLight, // 环境光
};

// 射线
const ray = new THREE.Raycaster();

// 鼠标二维向量用于归一化坐标
let mouse = new THREE.Vector2();

// 发光的参数
const params = {
  bloomStrength: 2.5, // 光线强度
  bloomThreshold: 0.1,
  bloomRadius: 1,
  exposure: 1, // 渲染器曝光级别
  edgeStrength: 3, // 边框的亮度
  edgeGlow: 2, // 光晕
  edgeThickness: 2, // 边框宽度
  downSampleRatio : 2 // 边框弯曲度
};


// 全部图层				// 发光图层
const ENTIRE_SCENE = 0, BLOOM_SCENE = 1;

let bloomComposer, finalPass, finalComposer, bloomPass;


export default {
  data() {
    return {
    };
  },
  mounted() {
    // 初始化场景
    this.init();

    // 窗口自适应
    this.windowResize();

        // 动画循环
    this.animation();

    this.exportModlue();

    let gui = new GUI();

    gui.add(params, "bloomStrength",0.1, 50).onChange((e)=> {
      params.bloomStrength  = e;
      bloomPass.strength  = e;
    })
    gui.add(params, "bloomThreshold",0.1, 10).onChange((e)=> {
      params.bloomThreshold  = e;
      bloomPass.threshold  = e;
    })
    gui.add(params, "bloomRadius",1, 10).onChange((e)=> {
      params.bloomRadius  = e;
      bloomPass.radius  = e;
    })
    gui.add(params, "exposure", 1, 10).onChange((e) => {
      params.exposure = e;
      renderer.toneMappingExposure = e;
    })

  },
  methods: {
    composer () {
      // 创建渲染层
      let renderScene = new RenderPass(scene, camera);
      // 创建辉光效果
      bloomPass = new UnrealBloomPass(
        new THREE.Vector2(window.innerWidth, window.innerHeight),
        5.5,
        1.8,
        1.85
      );
      /*设置泛光效果*/
      bloomPass.threshold = params.bloomThreshold;
      // 泛光的强度
      bloomPass.strength = params.bloomStrength;
      // 光的半径大小
      bloomPass.radius = params.bloomRadius;

      // 开始设置效果组合器,需要传入渲染器
      bloomComposer = new EffectComposer(renderer);
      bloomComposer.renderToScreen = false; // 不输出到场景上
      // 开始加入过程链

      // 先加入渲染层
      bloomComposer.addPass(renderScene);
      // 在加入辉光层
      bloomComposer.addPass(bloomPass);

      // 写入shader通道
      finalPass = new ShaderPass(
        new THREE.ShaderMaterial({
          uniforms: {
            baseTexture: { value: null },
            bloomTexture: { value: bloomComposer.renderTarget2.texture },
          },
          vertexShader: document.getElementById("vertexshader").textContent,
          fragmentShader: document.getElementById("fragmentshader").textContent,
          defines: {},
        }),
        "baseTexture"
      );
      // 需要交换
      finalPass.needsSwap = true;

      // 在写入一个渲染器
      finalComposer = new EffectComposer(renderer);
      // 开始添加过程链
      finalComposer.addPass(renderScene);
      finalComposer.addPass(finalPass);
    },
    init() {
      scene = new THREE.Scene();
      camera = new THREE.PerspectiveCamera(
        80,
        window.innerWidth / window.innerHeight,
        1,
        18000
      );
      camera.position.set(1, 8, 20);

      // 设置环境光
      let AmbientLight = new THREE.AmbientLight("#b2b9b1", 1);
      scene.add(AmbientLight);

      // 设置平行光
      direcLight = new THREE.DirectionalLight(0xeeeeee);
      direcLight.caseShadow = true;
      direcLight.position.set(0, 150, 0);
      scene.add(direcLight);

      let spc = new THREE.SpotLight();
      spc.position.y += 100;
      scene.add(spc);

      // mesh2.layers.enable(BLOOM_SCENE);
      // 设置渲染器
      renderer = new THREE.WebGLRenderer({ antialias: true });

      renderer.setSize(window.innerWidth, window.innerHeight);
      renderer.toneMapping = THREE.ReinhardToneMapping;
      // 渲染器开启阴影
      renderer.shadowMap.enabled = true;
      renderer.setPixelRatio(window.devicePixelRatio);
      // 渲染后的DOM放入页面
      document.querySelector(".main").appendChild(renderer.domElement);

      // 相机控制
      controler = new OrbitControls(camera, renderer.domElement);

      //显示时间
      setInterval(() => {
        this.nowTime = new Date().toLocaleString();
      });
    },

    animation() {
      renderer.render(scene, camera);

      // 设置先渲染光
      camera.layers.set(BLOOM_SCENE);
      if (bloomComposer) {
        bloomComposer.render();
      };
      // 设置渲染本来的场景
      camera.layers.set(ENTIRE_SCENE);
      if (finalComposer) {
        finalComposer.render();
      }

      if (texture.length) texture.forEach(item => item.offset.x -= 0.01);

      requestAnimationFrame(this.animation);
      
    },
    // 自适应浏览器宽高
    windowResize() {
      window.onresize = () => {
        let width = window.innerWidth, height = window.innerHeight;
        camera.aspect = width/ height; // 更新长宽比
        camera.updateProjectionMatrix(); // 更新世界坐标

        // 更新视图宽高
        renderer.setSize(width, height);
      };
    },
        
    exportModlue() {
      let loader = new GLTFLoader();
      // 用来解压网格数据
      let draLoader = new DRACOLoader();
      draLoader.setDecoderPath("three/examples/js/libs/draco/");
      loader.setDRACOLoader(draLoader);
      // 用来解压网格数据
      
      /*
        这里要注意的是路径，这个文件夹是放在public下的文件夹，
        不用带public，直接写路径即可，不然会报错
        nexpected token < in JSON at position 0
        at JSON.parse (<anonymous>)
      */
      loader.load("assets/modules/路.glb", (gltf) => {
        console.log(gltf);
        let disposeMesh = function (meshArr) {
          meshArr.forEach(item => {
            let tItem = gltf.scene.getObjectByName(item);
            if (tItem && tItem.geometry && tItem.material) {
              tItem.geometry.dispose();
              tItem.material.dispose();
              tItem.parent.remove(tItem);
            }
            
          })
        };
        scene.add(gltf.scene);
        // 找到需要的模型
        let bigRoad = gltf.scene.getObjectByName("NURBS_路径");

        disposeMesh(["1ap","宿舍.001","宿舍.002","宿舍.003","宿舍.004", "宿舍","宿舍1","宿舍2",
        "宿舍3","仓库","仓库001", "仓库边002", "立方体.042","立方体.043","立方体.044", "立方体.012","立方体.013","立方体.014","立方体.015"]);

        // 引入贴图
        let NewTexture = () => {
          
          let texture = new THREE.TextureLoader().load("assets/images/流光.png");
          texture.wrapS = texture.wrapT = THREE.RepeatWrapping;
          
          texture.repeat.set(1, 1);
          // texture.rotation = Math.PI / 2;
          texture.needsUpdate = true;

          return texture;
        }

        let map1 = new  NewTexture();
        // 设置材质
        let material  = new THREE.MeshBasicMaterial({
          map:map1,
          side: 2,
          transparent: true
        })
        // 给物体设置
        bigRoad.material = material;
      
        texture.push(map1);

        // 创建顶点数组
        let tpoints = [new THREE.Vector3(0, 0, 0),
          new THREE.Vector3(10, 0, 0),
          new THREE.Vector3(10, 0, 10),
          new THREE.Vector3(0, 0, 10)
        ]

        // CatmullRomCurve3创建一条平滑的三维样条曲线
        let curve = new THREE.CatmullRomCurve3(tpoints) // 曲线路径

        // 创建管道
        let tubeGeometry = new THREE.TubeGeometry(curve, 800, 0.1);

        let map2 = NewTexture();
        texture.push(map2);
        

        let mesh = new THREE.Mesh(tubeGeometry, new THREE.MeshBasicMaterial({
          map:map2,
          side:2,
          transparent: true
        }));
        mesh.material.map.rotation = 0;
        // littleRoad.material = new THREE.MeshBasicMaterial({
        //   side:2,
        //   transparent: true
        // })
        scene.add(mesh);
        mesh.layers.enable(BLOOM_SCENE);
        bigRoad.layers.enable(BLOOM_SCENE);
        this.composer();

      });
    },

    /**
     * e: 鼠标参数
     */
    getTargetMesh(e) {
      // 坐标归一化
      mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
      mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;

      // 射线查找，3D世界交互的基础方式
      ray.setFromCamera(mouse, camera);

      let theMesh = ray.intersectObjects(meshArr.children);

      return theMesh;
    },
  },
};
</script>

<style scoped lang="scss">
.btn {
  cursor: pointer;
  padding: 5px 15px;
  transition: all 0.6s;
  box-sizing: border-box;
  margin: 15px 5px;
  background-color: burlywood;
  border-radius: 5px;
  color: #fff;
  border: none;
  &:hover {
    background-color: #ff5722;
  }
}
.btn-active {
  background-color: #ff5722;
}
.main {
  width: 100%;
  height: 100vh;
  overflow: hidden;
  z-index: 5;
  .control {
    position: absolute;
    left: 10px;
    top: 10px;
    color: #fff;
    font-size: 1.2em;
    width: 20%;
    z-index: 10;
    background-color: rgba($color: #000000, $alpha: 0.2);
    box-shadow: #fff 0px 0px 15px 5px;
    ul {
      list-style: none;
      box-sizing: border-box;
      padding: 15px 8px;
      background-color: rgba($color: #c2c2c2, $alpha: 0.8);

      display: flex;
      li {
        width: 50%;
        &:hover {
          background-color: #ff5722;
        }
      }
    }
  }
}
.label-text {
  color: #fff;
  white-space: nowrap;
}
.foot {
  height: 50px;
  background-color: rgba(0, 0, 0, 0.6);
  width: 100%;
  position: absolute;
  bottom: 0;
  line-height: 50px;
  font-size: 18px;
  color: #fff;
  display: flex;
  padding: 5px 8px;
  box-sizing: border-box;
}
</style>
