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
@keyframes t_1{
	0% {
		transform:scale(1) ;
		opacity: 1;

	}
	100% {
		transform:scale(3.5);
		opacity: 0;
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
.css-render {
  height: 100px;
  width: 300px;
  background-color: rgba($color: #000000, $alpha: 0.6);
  color: #fff;
} 
/*小光点*/
.ind_spot{
	position: absolute;
	width: 15px;
	height: 15px;
	background: #FFD301;
	border-radius: 50%;
      /*小光点动画*/
  .xiab_spot_1{
    width: 15px;
    height: 15px;
    animation:t_1 3s infinite linear; 
    border-radius: 50%;
    box-shadow: 0 0 10px #FFD301 inset;
    
  }
  .xiab_spot_2{
    position: absolute;
    top: 0;
    width: 15px;
    height: 15px;
    animation:t_1 3s infinite 1s linear; 
    border-radius: 50%;
    box-shadow: 0 0 10px #FFD301 inset;
    
  }
}
.light-line {
  width: 11px;
  height: 50px;
  background-size: cover;
  border-radius: 50px;
  background-repeat: no-repeat;
  background-image: url('/assets/images/111.png');
  position: absolute;
  top: 0;
}

</style>
<template>
  <div class="main">
    <div class="light-line"></div>
    <div class="ind_spot">
			<div class="xiab_spot_1"></div>
			<div class="xiab_spot_2"></div>
      
		</div>
    <div class="css-render">
    </div>
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
// 效果组合器对象
import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer";
// 在指定的相机和场景上渲染新的场景，但是不会输出场景
import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass.js";
// 使用该通道你可以传入一个自定义的着色器，用来生成高级的、自定义的后期处理通道
import { ShaderPass } from "three/examples/jsm/postprocessing/ShaderPass.js";
// 形成高亮虚幻的效果
import { UnrealBloomPass } from "three/examples/jsm/postprocessing/UnrealBloomPass.js";
import { CSS3DRenderer, CSS3DObject } from 'three/examples/jsm/renderers/CSS3DRenderer.js';
// 拷贝shader
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
import { DRACOLoader } from "three/examples/jsm/loaders/DRACOLoader";

// 基础元素
let scene, camera, renderer, controler, css3Render, objectCSS;

let [meshArr] = [[]];

let texture = [];
let animationMixer, clock = new THREE.Clock(), moduleScene;
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
  bloomStrength: 3.2, // 光线强度
  bloomThreshold: 0.1,
  bloomRadius: 1,
  exposure: 1, // 渲染器曝光级别
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
        45,
        window.innerWidth / window.innerHeight,
        1,
        18000
      );
      camera.position.set(1, 5, 8);

      // 设置环境光
      let AmbientLight = new THREE.AmbientLight("#fff", 1);
      scene.add(AmbientLight);

      // 设置平行光
      direcLight = new THREE.DirectionalLight(0xeeeeee);
      direcLight.caseShadow = true;
      direcLight.position.set(0, 150, 0);
      scene.add(direcLight);

      let s = new THREE.SphereGeometry(10, 10, 10);
      let sM = new THREE.MeshPhongMaterial({
        map: new THREE.TextureLoader().load("assets/images/Earth.png")
      })
      let t = new THREE.Mesh(s, sM)
      t.scale.set(new THREE.Vector3(10, 10, 10));

      let spc = new THREE.SpotLight();
      spc.position.y += 100;
      scene.add(spc);

      // mesh2.layers.enable(BLOOM_SCENE);
      // 设置渲染器
      renderer = new THREE.WebGLRenderer({ antialias: true });
      scene.clearColor =  "#ff5722"

      renderer.setSize(window.innerWidth, window.innerHeight);
      renderer.toneMapping = THREE.ReinhardToneMapping;
      // 渲染器开启阴影
      renderer.shadowMap.enabled = true;
      renderer.setPixelRatio(window.devicePixelRatio);
      // 渲染后的DOM放入页面
      document.querySelector(".main").appendChild(renderer.domElement);
      renderer.domElement.style.position = "absolute"
      renderer.domElement.style.top = "0"

      objectCSS = new CSS3DObject( document.querySelector(".css-render") );
      // scene.add(objectCSS);
      css3Render = new CSS3DRenderer();
      css3Render.setSize(window.innerWidth, window.innerHeight);
      document.querySelector(".main").appendChild(css3Render.domElement);
       // 相机控制
      controler = new OrbitControls(camera, css3Render.domElement);

      css3Render.domElement.onpointermove = e => {
        let tempMesh = this.getTargetMesh(e);

        if (tempMesh.length) {
          moduleScene.animations.forEach(item => {

            if (item.name.indexOf(tempMesh[0].object.name) !== -1) {
              const action = animationMixer.clipAction(item);
              action.repetitions = 1;
              action.clampWhenFinished = true;
              action.play();
            } else {
              animationMixer.clipAction(item).stop();
            }
            
          })
        } else {
          moduleScene.animations.forEach(item => {

            animationMixer.clipAction(item).stop();
            
          })
        }

      }

      controler.maxPolarAngle = Math.PI / 2 - 0.1;

      controler.addEventListener("change", ()=> {
        let  p = camera.position.clone();
        p.y = objectCSS.position.y;
        objectCSS.lookAt(p);
      })

      // 控制相机的最大和最小视距离
      // controler.minDistance = 20;

    },

    animation() {
      renderer.render(scene, camera);
      css3Render.render(scene, camera);

      // 设置先渲染光
      camera.layers.set(BLOOM_SCENE);
      if (bloomComposer) {
        bloomComposer.render();
      }
      
      // 设置渲染本来的场景
      camera.layers.set(ENTIRE_SCENE);
      if (finalComposer) {
        finalComposer.render();
      }

      if (texture.length) texture.forEach(item =>{
        item.offset.x += 0.01;
      });

      if (animationMixer) {
        animationMixer.update(clock.getDelta());
      }

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
        css3Render.setSize(window.innerWidth, window.innerHeight);
        bloomComposer.setSize(window.innerWidth, window.innerHeight);
        finalComposer.setSize(window.innerWidth, window.innerHeight);
      };
    },

    /*
      @prams type{float} size:光圈最大范围
      @prams type{number} num:光圈数量
      @prams type{Object} position:光圈位置，需要一个三维向量对象
    */ 
    createCir (size , num, position) {
      
      let [tSize, tNum, tPosition] = [size || 0.2, num || 3, position || new THREE.Vector3(0,0,0)]
      
      if (!this.cirArr) {
        this.cirArr = [];
        this.scaleX = 0;
        this.scaleZ = 0;
        this.tempOpacity = 1;
      }

      const texture = new THREE.TextureLoader().load('assets/images/发光圆.png');
      const material = new THREE.MeshBasicMaterial( { map: texture ,side:2, transparent:true, opacity: this.tempOpacity} );
      
      const tgeometry = new THREE.CircleGeometry( tSize / num ,32 );
      const tmaterial = new THREE.MeshBasicMaterial( { color:"#ff5722", side: 2} );
      const tcircle = new THREE.Mesh( tgeometry, tmaterial );
      tcircle.position.copy(tPosition);
      tcircle.position.y += 0.5;
      tcircle.scale.set(0.5, 0.5, 0.5);
      tcircle.rotateX(Math.PI / 2);
      tcircle.layers.enable(1);

      scene.add(tcircle);

      for (let i = 0; i < tNum; i++) {
        const geometry = new THREE.CircleGeometry( tSize ,32 );
        const circle = new THREE.Mesh( geometry, material );
        circle.position.copy(tPosition);
        circle.position.y += 0.5;

        circle.scale.set( i / tNum, i / tNum, i / tNum);
        circle.rotateX(Math.PI / 2);
        scene.add(circle);
        this.cirArr.push(circle);

      }

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
      loader.load("assets/modules/贵阳市地图.glb", (t) => {
        console.log(t);
        let bx2 = t.scene.getObjectByName("贵州省边线2");

        animationMixer = new THREE.AnimationMixer(t.scene);

        bx2.material = new THREE.MeshBasicMaterial({color: "#ff5722"});
        bx2.layers.enable(1);
        scene.add(t.scene);

        moduleScene = t;
        
        texture.needsUpdate = true;
        let positionArr = ["安顺市","贵阳市","遵义市", "毕节市", "铜仁市", "黔西南布依族苗族自治州", "黔南布依族苗族自治州", "黔东南苗族侗族自治州","六盘水市"]
        positionArr.forEach(item => {

          let tempObj = t.scene.getObjectByName(item);
          let objectCSS = new CSS3DObject( document.querySelector('.ind_spot').cloneNode(true ) );
          objectCSS.scale.set(0.01,0.01,0.01);
          objectCSS.rotateX(Math.PI / 2);

          const geometry = new THREE.CylinderGeometry( 0.08, 0.05, 0.5, 32 );

          let textloader = new THREE.TextureLoader().load('/assets/images/111.png');

          textloader.wrapS = textloader.wrapT = THREE.RepeatWrapping;

          textloader.needsUpdate = true;

          textloader.repeat.set(1, 1);

          const material = new THREE.MeshBasicMaterial( { transparent:true,  side: 2,  map: textloader} );
          const material2 = new THREE.MeshBasicMaterial( { transparent:true, opacity:0, side: 2,} );
          const material3 = new THREE.MeshBasicMaterial( { transparent:true, opacity:0,side: 2} );
          
          const cylinder = new THREE.Mesh( geometry, [material, material2, material3] );
          cylinder.layers.enable(1);
          scene.add( cylinder );

          cylinder.position.copy(tempObj.position);
          cylinder.position.y += 0.75;

          // scene.add(objectCSS2);
          scene.add(objectCSS);

          objectCSS.position.copy(tempObj.position);
          objectCSS.position.y += 0.5;

          meshArr.push(tempObj);

        })
        
        this.composer();
      })

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

      let theMesh = ray.intersectObjects(meshArr);

      return theMesh;
    },
  },
};
</script>

