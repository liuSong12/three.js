<template>
  <div></div>
</template>


<script setup>
import * as THREE from 'three'
import gsap from "gsap"
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import { DRACOLoader } from "three/examples/jsm/loaders/DRACOLoader";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
import { RGBELoader } from "three/examples/jsm/loaders/RGBELoader";
import { Water } from "three/examples/jsm/objects/Water2"

// 初始化场景
const scene = new THREE.Scene()
// 初始化相机
const camera = new THREE.PerspectiveCamera(
  75,
  window.innerWidth / window.innerHeight,
  0.1,
  1000
)

camera.position.set(0, 5, 20)
camera.lookAt(0, 0, 0)

const renderer = new THREE.WebGLRenderer({
  // 抗锯齿
  antialias: true
})
renderer.setSize(window.innerWidth, window.innerHeight)
document.body.appendChild(renderer.domElement)
renderer.shadowMap.enabled = true
// renderer.physicallyCorrectLights = true
renderer.useLegacyLights = true

renderer.outputColorSpace = THREE.SRGBColorSpace
renderer.toneMapping = THREE.ACESFilmicToneMapping
renderer.toneMappingExposure = 1

const controls = new OrbitControls(camera, renderer.domElement)
controls.enableDamping = true

const gltfLoader = new GLTFLoader()
const dracoLoader = new DRACOLoader()
dracoLoader.setDecoderPath("/draco/")
gltfLoader.setDRACOLoader(dracoLoader)

gltfLoader.load("/model/scene.glb", (glb) => {
  const model = glb.scene
  model.traverse(child => {
    if (child.name == "Plane") {
      child.visible = false
    }
    if (child.isMesh) {
      child.receiveShadow = true
      child.castShadow = true
    }
  })
  scene.add(model)
})

// 创建水面
const waterGeometry = new THREE.CircleGeometry(300, 20)
const water = new Water(waterGeometry, {
  textureWidth: 1024,
  textureHeight: 1024,
  color: 0xeeeeff,
  flowDirection: new THREE.Vector2(1, 1),
  scale: 50 //水流快慢，越大越慢
})
water.rotation.x = -Math.PI / 2
water.position.y = -0.4
scene.add(water)


const rgbeLoader = new RGBELoader()
rgbeLoader.load("/textures/sky.hdr", (texture) => {
  texture.mapping = THREE.EquirectangularReflectionMapping
  scene.background = texture
  scene.environment = texture
})

const light = new THREE.DirectionalLight(0xffffff, 0.1)
light.position.set(20, 50, 0)
// 设置阴影模糊度
light.shadow.radius = 50
// 设置清晰度，避免模糊度太模糊
light.shadow.mapSize.set(4096, 4096)

scene.add(light)

// 添加房子里面的灯泡
const pointLight = new THREE.PointLight(0xffffff, 300)
pointLight.position.set(0.1, 2.4, 0)
pointLight.castShadow = true
scene.add(pointLight)

// 创建三个球
const pointLightGroup = new THREE.Group()
pointLightGroup.position.set(-8, 2.5, -1.5)
let pointLightArr = [];
let radius = 3;
for (let index = 0; index < 3; index++) {

  // 创建球
  const sphereGeometry = new THREE.SphereGeometry(0.2)
  const sphereMaterial = new THREE.MeshStandardMaterial({ color: 0xffffff })
  const sphere = new THREE.Mesh(sphereGeometry, sphereMaterial)
  sphere.receiveShadow = false
  pointLightArr.push(sphere)

  // 设置球的位置
  sphere.position.set(
    radius * Math.cos((index * 2 * Math.PI) / 3),
    Math.cos((index * 2 * Math.PI) / 3),
    radius * Math.sin((index * 2 * Math.PI) / 3)
  )

  let pointLightItem = new THREE.PointLight(0xffffff, 2)
  // pointLightItem.castShadow = true // 这个会错
  sphere.add(pointLightItem)
  pointLightGroup.add(sphere)
}

scene.add(pointLightGroup)



// 创建漫天星星
const starsInstance = new THREE.InstancedMesh(
  new THREE.SphereGeometry(0.1),
  new THREE.MeshStandardMaterial({
    color: 0xffffff,
    emissive: 0xffffff, //自发光属性
    emissiveIntensity: 10
  }),
  500//100颗星星
)

// 把星星随机到天上
for (let index = 0; index < 500; index++) {
  let x = Math.random() * 100 - 50
  let y = Math.floor((Math.random() * (50 - 5))) + 5
  let z = Math.random() * 100 - 50

  let matrix = new THREE.Matrix4()
  matrix.setPosition(x, y, z)
  starsInstance.setMatrixAt(index, matrix)
}
scene.add(starsInstance)


// 补间函数
let options = {
  angle: 0
}

gsap.to(options, {
  angle: Math.PI * 2,
  duration: 10,//10s一圈
  repeat: -1,
  ease: "linear",
  onUpdate: () => {
    pointLightGroup.rotation.y = options.angle;
    pointLightArr.forEach((item, index) => {
      item.position.set(
        radius * Math.cos((index * 2 * Math.PI) / 3),
        Math.cos((index * 2 * Math.PI) / 3 + options.angle * 5),
        radius * Math.sin((index * 2 * Math.PI) / 3)
      )
    })
  }
})

window.addEventListener("wheel", (e) => {
  console.log(e)
})

function animate() {
  renderer.render(scene, camera)
  controls.update()
  requestAnimationFrame(animate)
}

animate()


window.onresize = () => {
  renderer.setSize(window.innerWidth, window.innerHeight)
  camera.aspect = window.innerWidth / window.innerHeight
  camera.updateProjectionMatrix()
}



</script>