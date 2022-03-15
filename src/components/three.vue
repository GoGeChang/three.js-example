<template>
  <div class="main">
    <div class="control">
      <ul>
        <li
          class="btn"
          v-for="(item, index) in controlOperateList"
          :key="index"
          :class="{ 'btn-active': controlOperateState[item.type] }"
          @click="operate(item.type)"
        >
          {{ item.label }}
        </li>
      </ul>
      <button
        class="btn"
        v-show="controlOperateState.isOperate"
        @click="overOperate"
      >
        结束操作
      </button>
    </div>
    <div class="label-text" id="labelText">谁也不知道你以后能变得多强！</div>
    <footer class="foot">
      <label>当前时间：{{ nowTime }}</label>
      <div style="width: 60%; color: #ff5722">{{ infor }}</div>
    </footer>
  </div>
</template>
<script>
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import {
  CSS2DRenderer,
  CSS2DObject,
} from "three/examples/jsm/renderers/CSS2DRenderer";
import GUI from "three/examples/jsm/libs/lil-gui.module.min.js";
import { Line2 } from "three/examples/jsm/lines/Line2"; // line2 的mesh
import { LineGeometry } from "three/examples/jsm/lines/LineGeometry"; // line的geometry
import { LineMaterial } from "three/examples/jsm/lines/LineMaterial"; // line2的材质
import { TransformControls } from "three/examples/jsm/controls/TransformControls"; // 控制插件
// import * as CANNON from "cannon" // 物理引擎

// 基础元素
let scene,
  camera,
  renderer,
  mouseMoveOldEle,
  cssRender,
  drawLineOldEle,
  controler,
  marklineBuffer = [],
  transControls,
  curve;

// 物理引擎

//光源
let { spotLight, direcLight } = {
  spotLight, // 聚光灯
  direcLight, // 环境光
};

// 射线
const ray = new THREE.Raycaster();

//物体组
let meshArr = new THREE.Group();

// 落雪
let snows = new THREE.Group();

// 太阳
let sun;

// 鼠标二维向量用于归一化坐标
let mouse = new THREE.Vector2();

export default {
  data() {
    return {
      // 操作面板的配置数据
      controlOperateList: [
        {
          type: "drawLine",
          label: "测距线",
        },
        {
          type: "tagg",
          label: "连接线",
        },
      ],
      //
      controlOperateState: {
        isOperate: false,
        controlOperatedrawLine: false,
        tagg: false,
      },
      controlObj: {
        drawLine: null,
        lineGeometry: null,
        drawLineGroup: new THREE.Group(),
        taggGrop: new THREE.Group(),
      },
      // gui参数
      guiParams: {
        // 运行动画基本速度
        // animationSpeed: 0.0008,
        // 聚光灯颜色
        direcLightColor: "#fff",
        // 落雪速度
        snowSpeed: 0.4,
        // 环境光颜色
        ambLightColor: "#fff",
        // 太阳运行速度
        sunSpeed: 0.0008,
        // 开启背景图
        skyBox: true,
      },
      //当前时间
      nowTime: "",
      // 提示信息
      infor: "",
    };
  },
  mounted() {
    // 初始化场景
    this.init();
    // 动画循环
    this.animation();
    // 窗口自适应
    this.windowResize();
    // 鼠标移动事件
    this.onMouseMove();
    // 鼠标点击事件
    this.onClick();
    // 天空盒
    this.initSkyBox().show();
    // 下雪
    this.initSnow();
    // 初始化GUI
    this.initGUI();
  },
  methods: {
    init() {
      scene = new THREE.Scene();

      // 雾化
      
      scene.fog = new THREE.Fog("#fff", 0.85, 100,3000);

      camera = new THREE.PerspectiveCamera(
        80,
        window.innerWidth / window.innerHeight,
        1,
        18000
      );
      camera.position.set(1, 8, 20);


      // 设置环境光
      let AmbientLight = new THREE.AmbientLight("#b2b9b1");
      scene.add(AmbientLight);

      // 设置平行光
      direcLight = new THREE.DirectionalLight(0xeeeeee);
      direcLight.caseShadow = true;
      direcLight.position.set(0, 150, 0);
      // scene.add(direcLight);

      // 设置聚光灯
      spotLight = new THREE.SpotLight(this.guiParams.direcLightColor);
      spotLight.position.set(30, 18, 52);

      spotLight.castShadow = true;
      spotLight.shadow.mapSize.width = 2048;
      spotLight.shadow.mapSize.height = 2048;

      // 增加辅助线
      scene.add(spotLight);

      // 设置渲染器
      renderer = new THREE.WebGLRenderer();

      renderer.setSize(window.innerWidth, window.innerHeight);
      // 渲染器开启阴影
      renderer.shadowMap.enabled = true;
      renderer.shadowMap.type = THREE.PCFSoftShadowMap;
      // 渲染后的DOM放入页面
      document.querySelectorAll(".main")[0].appendChild(renderer.domElement);

      // 相机控制
      controler = new OrbitControls(camera, renderer.domElement);

      // 控制相机的最大和最小视距离
      controler.maxDistance = 80;
      controler.minDistance = 20;

      //显示时间
      setInterval(() => {
        this.nowTime = new Date().toLocaleString();
      });

      // 添加三维控制器
      transControls = new TransformControls(camera, renderer.domElement);
      scene.add(transControls);

      // 初始化模型
      this.initMesh(scene);

      this.initLine();
      

    },
    initLine () {
      let lineGroup = new THREE.Group();
      let lineArrPosition = [
        {x:14,y:10, z:20},
        { x:10, y:12, z:16},
        {x:20, y:10, z:30},
        {x:17, y:20, z:10},
      ];
      lineArrPosition.map(item => {
        let material = new THREE.MeshBasicMaterial({color:"red"});
        let geometry = new THREE.BoxBufferGeometry(0.1, 0.1, 0.1);
        let mesh = new THREE.Mesh(geometry, material);
        mesh.position.copy(item);
        lineGroup.add(mesh);
      });
      curve = new THREE.CatmullRomCurve3(
        lineGroup.children.map(item => {
          return item.position
        })
      )
      let points = curve.getPoints(100);
      const lineLoop = new THREE.LineLoop(
        new THREE.BufferGeometry().setFromPoints(points),
        new THREE.LineBasicMaterial({color:"green"})
      );
      scene.add(lineLoop);
      scene.add(lineGroup);
    },
    // 设置GUI测试
    initGUI() {
      let gui = new GUI();
      // 总动画速度
      gui.add(this.guiParams, "sunSpeed", 0.0008, 0.008).onChange((e) => {
        this.guiParams.sunSpeed = e;
      });
      // 聚光灯颜色
      gui.addColor(this.guiParams, "direcLightColor").onChange((e) => {
        //点击颜色面板，e为返回的10进制颜色
        spotLight.color.set(e);
      });

      // 落雪速度
      gui.add(this.guiParams, "snowSpeed", 0.4, 5).onChange((e) => {
        this.guiParams.snowSpeed = e;
      });
      // 显示天空盒
      gui.add(this.guiParams, "skyBox").onChange((e) => {
        e ? this.initSkyBox().show() : this.initSkyBox().hide();
      });
    },

    // 初始化模型
    initMesh() {
      // 模型一
      let geometry1 = new THREE.BoxBufferGeometry(8, 8, 8);
      let mesh1ImgLoader = new THREE.TextureLoader().load(
        "../assets/images/1.jpg"
      );
      let geometryMatrial = new THREE.MeshPhongMaterial({
        map: mesh1ImgLoader,
      });
      let mesh1 = new THREE.Mesh(geometry1, geometryMatrial);

      mesh1.position.y += 4;
      mesh1.position.x -= 8;
      mesh1.isMyMesh = true; // 添加自定义属性， 方便后期操作

      // 模型二
      let geometry2 = new THREE.BoxBufferGeometry(8, 8, 8);
      let geometryMatrial2 = new THREE.MeshPhongMaterial({ color: "#fff" });
      let mesh2 = new THREE.Mesh(geometry2, geometryMatrial2);

      mesh2.position.y += 4;
      mesh2.position.x += 8;
      mesh2.isMyMesh = true; // 添加自定义属性， 方便后期操作

      // 模型三
      let geometry3 = new THREE.BoxBufferGeometry(300, 10, 300);
      let geometryMatrial3 = new THREE.MeshPhongMaterial({
        color: "#ff5722",
        transparent: true,
        opacity: 0.6,
      });
      let mesh3 = new THREE.Mesh(geometry3, geometryMatrial3);
      mesh3.position.y -= 10;
      meshArr.add(mesh1);
      meshArr.add(mesh2);
      meshArr.add(mesh3);

      // 设置阴影
      meshArr.children.forEach((item) => {
        item.castShadow = true;
        item.receiveShadow = true;
        item.isActive = true;
        
      });

      mesh3.isActive = false;

      scene.add(meshArr);

      // 添加太阳
      let sunGeo = new THREE.SphereBufferGeometry(1);
      let sunImg = new THREE.TextureLoader().load("../assets/images/sun.jpg");
      let sunMatirl = new THREE.MeshPhongMaterial({ map: sunImg });
      sun = new THREE.Mesh(sunGeo, sunMatirl);
      sun.position.set(30, 18, 52);
      scene.add(sun);

      // 添加Dom到场景
      this.initCssDom(mesh1);
    },
    // 天空盒
    initSkyBox() {
      let skyBox = {
        path: "../assets/sky_box/christmas/",
        imageList: [
          "夜景_LF.jpg",
          "夜景_RT.jpg",
          "夜景_UP.jpg",
          "夜景_DN.jpg",
          "夜景_FR.jpg",
          "夜景_BK.jpg",
        ],
      };

      let cubTextureLoader = new THREE.CubeTextureLoader()
        .setPath(skyBox.path)
        .load(skyBox.imageList);
        cubTextureLoader.encoding = THREE.sRGBEncoding;
      return {
        show() {

          scene.background = cubTextureLoader;

          scene.background.dispose();

        },
        hide() {

          scene.background.dispose();
          scene.background = null;

        },
      };
    },
    // 操作
    operate(type) {
      this.controlOperateState.isOperate = true;
      if (this.controlOperateState.isOperate) {
        this.controlOperateState[type] = true;

        Object.keys(this.controlOperateState).map((key) => {
          if (key !== "isOperate" && type !== key) {
            this.controlOperateState[key] = false;
          }
        });
      }
    },
    // 结束操作
    overOperate() {
      // 关闭操作
      Object.keys(this.controlOperateState).map((key) => {
        this.controlOperateState[key] = false;
      });
      // 清空底部文字提示
      this.infor = "";
    },
    // 精灵系统创建下雪效果
    initSnow() {
      let texture = new THREE.TextureLoader().load("./assets/images/snow.jpg");
      let spriteMaterial = new THREE.SpriteMaterial({
        map: texture, //设置精灵纹理贴图
      });
      // 创建精灵模型对象

      // 批量创建精灵模型
      for (let i = 0; i < 1000; i++) {
        let sprite = new THREE.Sprite(spriteMaterial);
        snows.add(sprite); // 添加到场景或者是组
        // 控制精灵大小,
        sprite.scale.set(0.8, 1.5, 1); // 只需要设置x、y两个分量就可以
        let k1 = Math.random() - 0.5;
        let k2 = Math.random() - 0.5;
        let k3 = Math.random() - 0.5;
        // 设置雪的随机位置
        sprite.position.set(500 * k1, 500 * k3, 500 * k2);
      }

      scene.add(snows);
    },
    // 将dom渲染到三维场景
    initCssDom(mesh) {
      // 加载DOM元素
      let cssObject = new CSS2DObject(document.querySelector("#labelText"));
      cssObject.position.set(0, 0, 0);

      mesh.add(cssObject);

      // 创建渲染器
      cssRender = new CSS2DRenderer();

      cssRender.setSize(window.innerWidth, window.innerHeight);
      cssRender.domElement.style = "position:absolute;top:0;";

      document.querySelectorAll(".main")[0].appendChild(cssRender.domElement);
    },
    animation() {
      renderer.render(scene, camera);
      cssRender.render(scene, camera);

      // 根据时间来测算动画速度
      // let time = new Date() * this.guiParams.animationSpeed;

      // 旋转第一个物体
      meshArr.children[0].rotateY(0.002);

      // 太阳公转
      if (sun) {
        let time = new Date() * this.guiParams.sunSpeed;
        let positionX = Math.sin(time) * 30;
        let positionZ = Math.cos(time) * 30;
        sun.position.x = positionX;
        sun.position.z = positionZ;
        spotLight.position.x = positionX;
        spotLight.position.z = positionZ;
      }

      // 落雪效果
      snows.children.forEach((sprite) => {
        sprite.position.y -= this.guiParams.snowSpeed;

        if (sprite.position.y < -200) {
          sprite.position.y = 200;
        }
      });

      requestAnimationFrame(this.animation);
    },
    // 自适应浏览器宽高
    windowResize() {
      window.onresize = () => {
        camera.aspect = window.innerWidth / window.innerHeight; // 更新长宽比
        camera.updateProjectionMatrix(); // 更新世界坐标

        // 更新视图宽高
        renderer.setSize(window.innerWidth, window.innerHeight);

        // 更新cssDom位置，防止错位
        cssRender.setSize(window.innerWidth, window.innerHeight);
        cssRender.domElement.style = "position:absolute;top:0;";
      };
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
    onClick() {
      renderer.domElement.onclick = (e) => {
        let meshArr = this.getTargetMesh(e);
        if (meshArr.length) {
          let theMesh = meshArr[0];
          if (theMesh.object === transControls.object) {
            controler.enabled = false;
          } else {
            controler.enabled = true;
          }
          // 画笔标注
          if (this.controlOperateState.tagg) {
            // 实例化line的geometry
            let lineGeometry = new LineGeometry();
            this.controlObj.lineGeometry = lineGeometry;

            // 添加顶点
            marklineBuffer.push(
              meshArr[0].point.x,
              meshArr[0].point.y,
              meshArr[0].point.z
            );

            // 给geometry设置顶点
            lineGeometry.setPositions(marklineBuffer);

            let theMaterial = new LineMaterial({
              color: "green",
              linewidth: 0.005, // 线段的宽
            });

            let markLine = new Line2(lineGeometry, theMaterial);

            markLine.onBeforRender = function () {
              this.material.resolution.set(window.innerWidth, innerHeight);
            };

            // 添加线段即可
            this.controlObj.taggGrop.add(markLine);
            return;
          }
          // 画线
          if (this.controlOperateState.drawLine) {
            if (!this.controlObj.drawLine) {
              // 设置gemoetry
              let lineGeometry = new THREE.BufferGeometry();
              // 设置材质
              let material = new THREE.LineBasicMaterial({
                color: "red",
                depthTest: true,
              });
              // 设置线段初始点顶点数据
              this.drawLineBufferAttribute = [
                theMesh.point.x,
                theMesh.point.y,
                theMesh.point.z,
              ];

              // 设置顶点
              lineGeometry.setAttribute(
                "position",
                new THREE.Float32BufferAttribute(
                  this.drawLineBufferAttribute,
                  3
                )
              );
              // 重新计算包围球
              lineGeometry.computeBoundingSphere();

              let theLine = new THREE.Line(lineGeometry, material);

              // 添加标识
              theLine.isDrawLine = true;

              this.controlObj.drawLine = theLine;
              this.controlObj.drawLineGroup.add(theLine);
              drawLineOldEle = theMesh;
            } else {
              // 将最后点击的位置作为线段的终点
              let thePoint = [
                ...this.drawLineBufferAttribute,
                theMesh.point.x,
                theMesh.point.y,
                theMesh.point.z,
              ];
              // 重新设置顶点数据
              this.controlObj.drawLine.geometry.setAttribute(
                "position",
                new THREE.Float32BufferAttribute(thePoint, 3)
              );

              // 完成画线
              this.controlObj.drawLine = null;
              console.log(drawLineOldEle);
              console.log(theMesh);
              // this.infor = `两点之间距离为${Math.abs(
              //   drawLineOldEle.distance - theMesh.distance
              // ).toFixed(2)}`;
              console.log(drawLineOldEle);
              this.infor = `两点之间距离为${drawLineOldEle.point.distanceTo(
                theMesh.point
              )}`;

              drawLineOldEle = null;
            }
          }
        }
      };
    },
    // 鼠标移动查找物体
    onMouseMove() {
      renderer.domElement.onmousemove = (e) => {
        let meshArr = this.getTargetMesh(e);

        // 判断射线是否碰撞到物体
        if (meshArr.length) {
          let theMesh = meshArr[0];
          // 鼠标移动获取物体改变样式
          if (mouseMoveOldEle) {
            mouseMoveOldEle.object.material.color.set("#fff");
          }
          
         
          if (theMesh.object.isActive) {
            transControls.attach(theMesh.object);
            controler.enabled = false;
            // 添加三维控制
           
            // 鼠标获取到的物体变色
            theMesh.object.material.color.set("red");

            // 鼠标移动的上一个元素
            mouseMoveOldEle = theMesh;
          } else {
            transControls.detach();
            controler.enabled = true;
          }

          // 画线
          if (this.controlObj.drawLine) {
            let thePoint = [
              ...this.drawLineBufferAttribute,
              theMesh.point.x,
              theMesh.point.y,
              theMesh.point.z,
            ];
            this.controlObj.drawLine.geometry.setAttribute(
              "position",
              new THREE.Float32BufferAttribute(thePoint, 3)
            );
          }
        } else if (mouseMoveOldEle) {
          mouseMoveOldEle.object.material.color.set("#fff");
        }
      };
    },
  },
  watch: {
    controlOperateState: {
      handler(newVal) {
        // 画线功能
        if (newVal.drawLine) {
          console.log("画线了");
          scene.add(this.controlObj.drawLineGroup);
        } else {
          if (this.controlObj.drawLineGroup.children.length) {
            console.log("结束画线，清除所有线条");
            // 从场景移除
            scene.remove(this.controlObj.drawLineGroup);
            // 清空添加的线段
            this.controlObj.drawLineGroup.children = [];
          }
        }

        // 标注功能
        if (newVal.tagg) {
          console.log("开始标注");
          scene.add(this.controlObj.taggGrop);
        } else {
          console.log("结束标注");
          scene.remove(this.controlObj.taggGrop);
          this.controlObj.taggGrop.children = [];
          marklineBuffer = [];
        }
      },
      deep: true,
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