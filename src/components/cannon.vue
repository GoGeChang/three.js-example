<template>
  <div class="main" ref="main"></div>
</template>

<script>
import * as CANNON from "cannon";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import GUI from "three/examples/jsm/libs/lil-gui.module.min.js";
// 动画库
import gsap from "gsap";
// three元素
let scene, camera, light, renderer, ground, control;

let spherePostion;
// 物理引擎
let sphereBody, sphereMesh;
let world,
  objectToUpdate = [];
// 时钟和GUI
let Clock = new THREE.Clock(),
  oldElapsedTime = 0,
  gui = new GUI();
// 甩出的圆形
let strikeSphere = [];
// 塑料材质
const plasticMaterial = new CANNON.Material("plastic");
// 混凝土材质
const concreteMaterial = new CANNON.Material("concrete");

export default {
  name: "cannon",
  data() {
    return {
      meshArr: [],
    };
  },
  mounted() {
    this.initThree();
    this.initCannon();
    this.windowResize();
    this.animation();
    this.initEvent();
    let guiObject = {
      createSphere: () => {
        this.createSphere(Math.random() * 3, {
          x: Math.random() * 5,
          y: 30,
          z: (Math.random() - 0.5) * 5,
        });
      },
      constantraint: 1,
      maxForce: 0.5,
      sphereBounce: () => {
        spherePostion.anmiationEnd = false;
        gsap.to(spherePostion, {
          y: 22,
          z: 0,
          x: 0,
          duration: 1.5,
          onComplete: () => {
            this.sphereBounce();
          },
        });
      },
    };

    gui.add(guiObject, "sphereBounce");
    gui.add(guiObject, "createSphere");
    gui.add(guiObject, "constantraint", 1, 10);
    gui.add(guiObject, "maxForce", 0.5, 10);

    setInterval(() => {
      // 移除最新插入的两个mesh和body
      if (objectToUpdate.length > 1 && objectToUpdate.length % 2) {
        // 添加距离约束
        // let distanceConstraint = new CANNON.DistanceConstraint(
        //   objectToUpdate[objectToUpdate.length - 1].body,
        //   objectToUpdate[objectToUpdate.length - 2].body,
        //   guiObject.constantraint,
        //   guiObject.maxForce
        // );

        // world.addConstraint(distanceConstraint);
      }
      if (objectToUpdate.length > 16 && objectToUpdate.length % 2) {
        for (let i = 0; i < 2; i++) {
          scene.remove(objectToUpdate[i].mesh);
          scene.remove(objectToUpdate[i].mesh);

          world.removeBody(objectToUpdate[i].body);
          world.removeBody(objectToUpdate[i].body);
        }

        objectToUpdate.splice(0, 2);
      }
      guiObject.createSphere();
    }, 1000);
  },
  methods: {
    initEvent () {
      renderer.domElement.addEventListener("click", e=> {
        e.preventDefault();
        let x = (e.clientX / window.innerWidth) * 2 - 1;
        let y = -(e.clientY / window.innerHeight) * 2 + 1;
        /**
         * 这里的Z值，是一个 [-1, 1]之间的距离,只要是在这个区间，好像区别不大？
         * 并且通过unproject方法，传入相机来得到鼠标点到的三维空间的点
         */
        let p = new THREE.Vector3(x, y , -1).unproject(camera);
        /*
          p是一个三维向量，有一个sub(v)方法来获取p - v得的三维向量的值，
          之后在使用normalize将该向量的方向设置为和原向量相同，但是其长度（length）为1，暂时不理解，先用起来再说
        */
        let v = p.sub(camera.position).normalize();

        this.createStrikeSphere(v, camera.position);
      })
    },
    createStrikeSphere(v, c) {
      /*
        v: 减出来的向量
        c: 相机的向量
        mesh和body都用转化过的camera的position
      */
      let speed = 80; // 飞出去的初始应力，可以理解为飞出去的速度，速度越快，应力越大
      let theMaterial = new THREE.MeshPhongMaterial({ color: "#ff5722" });
      let theGeometry = new THREE.SphereBufferGeometry(1.5, 10, 10);
      let sphereMesh = new THREE.Mesh(theGeometry, theMaterial);
      sphereMesh.position.copy(c);
      sphereMesh.castShadow = true;
      sphereMesh.receiveShadow = true;

      scene.add(sphereMesh);

      let theBody = new CANNON.Body({
        material: new CANNON.Material("concrete"),
        shape: new CANNON.Sphere(1.5),
        mass: 5,
        position: new CANNON.Vec3().copy(c)
      })
       // 碰撞反弹力
      theBody.collisionResponse = 0.1;
      // 初速度的大小，是一个三维向量
      theBody.velocity.set(v.x * speed, v.y * speed, v.z * speed);
      world.add(theBody);

      strikeSphere.push({
        mesh: sphereMesh,
        body: theBody
      })
    },
    getColor16() {
      //十六进制颜色随机
      var r = Math.floor(Math.random() * 256);
      var g = Math.floor(Math.random() * 256);
      var b = Math.floor(Math.random() * 256);

      var color = "#" + r.toString(16) + g.toString(16) + b.toString(16);
      return color;
    },
    sphereBounce() {
      let timeLine = gsap.timeline();
      // 写时间轴动画
      timeLine
        .to(spherePostion, {
          y: Math.random() * 10 < 10 ? 10 : Math.random() * 10,
          z: Math.random() * 20,
          duration: 1,
          yoyo: true,
          ease: "elastic.out(1, 0.3)",
          onStart: () => {
            console.log("动画开始了");
          },
        })
        .to(spherePostion, {
          y: Math.random() * 10 < 10 ? 10 : Math.random() * 10,
          z: 15,
          duration: 1.2,
          yoyo: true,
          ease: "elastic.out(1, 0.3)",
        })
        .to(spherePostion, {
          y: 10,
          z: Math.random() * 5,
          duration: 1.2,
          yoyo: true,
          ease: "elastic.out(1, 0.3)",
          onComplete: () => {
            console.log("动画结束了");
            spherePostion.anmiationEnd = true;
            setTimeout(()=> {
              spherePostion.anmiationEnd = false;
              gsap.to(spherePostion, {
                y: 22,
                z: 0,
                x: 0,
                duration: 1.5,
                onComplete: () => {
                  this.sphereBounce();
                },
              });
            }, 1000)
          },
        });
    },
    initThree() {
      scene = new THREE.Scene();
      camera = new THREE.PerspectiveCamera(
        95,
        window.innerWidth / window.innerHeight,
        1,
        1000
      );
      camera.position.set(20, 20, 20);
      scene.add(camera);

      scene.add(new THREE.AmbientLight());

      light = new THREE.DirectionalLight("#304758", 0.8);
      light.castShadow = true;
      light.position.set(15, 20, 55);
      scene.add(light);

      let light2 = new THREE.SpotLight("#c2c2c2", 0.3);
      light2.castShadow = true;
      light2.position.set(0, 40, 0);
      light2.shadow.mapSize.width = 2048;
      light2.shadow.mapSize.height = 2048;
      scene.add(light2);

      renderer = new THREE.WebGLRenderer();
      renderer.setSize(window.innerWidth, window.innerHeight);
      renderer.shadowMap.enabled = true;
      renderer.setClearColor("#c5c5c5");

      let groundGeom = new THREE.BoxBufferGeometry(50, 1, 50);
      let groundMate = new THREE.MeshPhongMaterial({
        color: "#b1c5b4"
      });

      ground = new THREE.Mesh(groundGeom, groundMate);
      ground.position.y = 5;
      ground.receiveShadow = true;
      ground.castShadow = true;
      scene.add(ground);
      
      // 包围球体的盒子
      let groundGeom1 = new THREE.BoxBufferGeometry(800, 0.1, 800);
      let groundMate2 = new THREE.MeshPhongMaterial({
        color: "#00b483",
        // side: THREE.BackSide
      });

      let ground1 = new THREE.Mesh(groundGeom1, groundMate2);
      ground1.receiveShadow = true;
      ground1.castShadow = true;
      scene.add(ground1);

      light.target = ground;

      // 用这个方法和透明的盒子模型结合在一起，把盒子的长宽高传进来
      function box( width, height, depth ) {

        // 长宽高
				width = width * 0.5,
				height = height * 0.5,
				depth = depth * 0.5;

        // 存放顶点数据
				const geometry = new THREE.BufferGeometry();
				const position = [];

        // 设置十二条边线，保证中空效果
				position.push(
					- width, - height, - depth,
					- width, height, - depth,

					- width, height, - depth,
					width, height, - depth,

					width, height, - depth,
					width, - height, - depth,

					width, - height, - depth,
					- width, - height, - depth,

					- width, - height, depth,
					- width, height, depth,

					- width, height, depth,
					width, height, depth,

					width, height, depth,
					width, - height, depth,

					width, - height, depth,
					- width, - height, depth,

					- width, - height, - depth,
					- width, - height, depth,

					- width, height, - depth,
					- width, height, depth,

					width, height, - depth,
					width, height, depth,

					width, - height, - depth,
					width, - height, depth
				 );

        // 设置模型顶点数据
				geometry.setAttribute( 'position', new THREE.Float32BufferAttribute( position, 3 ) );

				return geometry;

			}

      let theBox = new THREE.LineSegments(  box( 20, 20, 20 ), new THREE.LineBasicMaterial( { color: 0xffaa00 } ) );
      theBox.position.y += 20;
      scene.add( theBox );

      let sphere = new THREE.SphereBufferGeometry(2);
      let sphereMaterial = new THREE.MeshPhongMaterial({ color: "#ff5722" });
      sphereMesh = new THREE.Mesh(sphere, sphereMaterial);
      sphereMesh.castShadow = true;
      sphereMesh.receiveShadow = true;
      sphereMesh.position.y += 2;
      sphereMesh.material.transparent = true;
      sphereMesh.material.opacity = 0.9;
      scene.add(sphereMesh);

      scene.fog = new THREE.Fog("#c5c5c5", 0, 200);

      this.$refs.main.appendChild(renderer.domElement);

      control = new OrbitControls(camera, renderer.domElement);
      control.maxPolarAngle = Math.PI / 2 - 0.05;
    },
    createSphere(radius, position) {
      // 创建mesh
      let sphereGemotry = new THREE.SphereBufferGeometry(radius, 20, 20);
      let sphereMaterial = new THREE.MeshStandardMaterial({
        metalness: 0.3,
        roughness: 0.2,
        color: this.getColor16(),
      });

      let sphereMesh = new THREE.Mesh(sphereGemotry, sphereMaterial);
      sphereMesh.castShadow = true;
      sphereMesh.position.copy(position);
      scene.add(sphereMesh);
      // 创建刚体
      let body = new CANNON.Body({
        shape: new CANNON.Sphere(radius),
        position: new CANNON.Vec3(0, 8, 0),
        mass: 1,
        material: plasticMaterial,
      });

      body.position.copy(position);

      world.add(body);

      objectToUpdate.push({
        mesh: sphereMesh,
        body: body,
      });
    },
    initCannon() {
      // 创建物理体系世界
      world = new CANNON.World();
      // 设置y轴方向的重力
      world.gravity.set(0, -10, 0);

      // 创建刚体模型，这里的是圆形,这里的半径和three物体的半径要一致，保证刚体和three的物体是融为一体的
      const sphereShape = new CANNON.Sphere(2);

      spherePostion = new CANNON.Vec3(0, 8, 0);
      spherePostion.anmiationEnd = false;

      // 测试圆球的刚体
      sphereBody = new CANNON.Body({
        mass: 1, // 物体的质量，碰撞时质量越小的越容易被撞开
        position: spherePostion, // 刚体的位置
        shape: sphereShape, // 赋予形状
        material: plasticMaterial, // 材质
      });

      // 给物体施加力
      // sphereBody.applyLocalForce(new CANNON.Vec3(100, 0, 0), new CANNON.Vec3(0,0,0));

      // 创建物体世界地面
      let floorBody = new CANNON.Body({
        mass: 0, // 质量为0，表示是静态的
        shape: new CANNON.Plane(), // 地面形状
        material: concreteMaterial, // 给地板设置联系材质，产生反馈情况
      });

      // 只能通过四元数来旋转，参数为：1、旋转轴, 2、角度
      floorBody.quaternion.setFromAxisAngle(
        new CANNON.Vec3(-1, 0, 0),
        Math.PI / 2
      );

      /*
        联系材质: ContactMaterial，意思就是把两个材质碰撞的情况反馈回来
        比如本个例子中，球体是塑料材质，地面设置成了混泥土材质，所以反弹效果
        会比较明显。 
        @params: {
          material1, // 第一种材质
          material, // 第二种材质
          option  // 一些参数的设置，比如摩擦力，恢复力
        }
      */
      const concretePlasticMaterial = new CANNON.ContactMaterial(
        plasticMaterial,
        concreteMaterial,
        {
          friction: 0.3,
          restitution: 0.3,
        }
      );

      // 添加盒子刚体
      let boxShape = new CANNON.Box(new CANNON.Vec3(50 / 2, 1 / 2, 50 / 2));
      let boxBody = new CANNON.Body({
        mass: 0,
        material: concreteMaterial,
        shape: boxShape,
        position: new CANNON.Vec3().copy(ground.position),
      });

      world.add(boxBody);

      // 添加联系材质
      world.addContactMaterial(concretePlasticMaterial);
      // 添加圆形刚体
      world.add(sphereBody);
      // 添加地板刚体
      world.add(floorBody);

      /**
       * 总结：基本思路和three一样，只不过写法不同，
       * 但是都是先创建相应的刚体模型然后设置好物理属性材质来决定刚体的特性并生产一个含有物理属性的刚体添加到世界里面，
       * 然后同步刚体和three物体的位置，就产生了真实的模拟
       *
       */
    },
    animation() {
      // 计算时间差
      const elapsedTime = Clock.getElapsedTime();
      const tempTime = elapsedTime - oldElapsedTime;
      oldElapsedTime = tempTime;
      // 三个参数，帧率，时间差，速度
      world.step(1 / 60, tempTime, 2);

      // 如果动画没有结束以动画的position为主
      if (!spherePostion.anmiationEnd) {
        sphereBody.position.copy(spherePostion);
        sphereMesh.position.copy(spherePostion);
      } else {
        spherePostion.copy(sphereBody.position);
      }

      // mesh的位置保持和刚体的一直
      sphereMesh.position.copy(sphereBody.position);

      // 随机落下的圆形的位置
      objectToUpdate.forEach((item) => {
        item.mesh.position.copy(item.body.position);
      });

      strikeSphere.forEach(item => {
        item.mesh.position.copy(item.body.position);
      })

      renderer.render(scene, camera);

      requestAnimationFrame(this.animation);
    },
    windowResize() {
      window.onresize = () => {
        camera.aspect = window.innerWidth / window.innerHeight; // 更新长宽比
        camera.updateProjectionMatrix(); // 更新世界坐标

        // 更新视图宽高
        renderer.setSize(window.innerWidth, window.innerHeight);
      };
    },
  },
};
</script>
<style lang="sass" scope>
</style>