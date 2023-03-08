<script setup>
import {
  Scene,
  PerspectiveCamera,
  WebGLRenderer,
  Vector3,
  Spherical,
  Mesh,
  AmbientLight,
  DirectionalLight,
  SpotLight,
  MeshStandardMaterial,
  CylinderGeometry,
  SphereGeometry,
  PlaneGeometry,
  CircleGeometry,
  Skeleton,
  Bone,
  ArrowHelper,
  MeshBasicMaterial,
  SkeletonHelper,
  Group,
  Euler,
  DoubleSide,
  CameraHelper,
  PCFSoftShadowMap,
} from 'three'
import { gsap } from 'gsap'
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'
import { ImprovedNoise } from 'three/examples/jsm/math/ImprovedNoise.js'
import { GUI } from 'dat.gui'
import { deg2rad } from '~/helpers/deg2rad'

const config = {
  iterations: 6,
  splits: 2,
  end_splits: 3,
  leaf_splits: 3,
  leaf_size: 0.15,

  branch_length: 2,
  branch_thickness: 0.25,

  branch_theta: 137.5,
  branch_phi: 65,

  theta_randomness: 35,
  phi_randomness: 15,

  length_exponent: 1.25,
  trunk_length_exponent: 1.25,
}

/**
 * Scene initialization
 */
const container = ref(null)
const scene = new Scene()
const camera = new PerspectiveCamera(75, 1, 0.1, 2000)
camera.position.set(0, 2, 10)
const renderer = new WebGLRenderer({
  antialias: true,
})
const controls = new OrbitControls(camera, renderer.domElement)
controls.target = new Vector3(0, 4, 0)
renderer.shadowMap.enabled = true
renderer.shadowMap.type = PCFSoftShadowMap
renderer.setPixelRatio(window.devicePixelRatio)

function init() {
  container.value.appendChild(renderer.domElement)
}

function resize() {
  camera.aspect = window.innerWidth / window.innerHeight
  camera.updateProjectionMatrix()
  renderer.setSize(window.innerWidth, window.innerHeight)
}

/**
 * Tree
 */
const branches = []
const bones = []
// let skeleton

function addNode({ level, parent, theta, phi, x = 0, y = 0, z = 0 }) {
  const isLast = level === config.iterations - 1
  const length =
    config.branch_length * Math.pow(1 - level / config.iterations, config.length_exponent)

  const bone = new Bone()
  bone.setRotationFromEuler(new Euler(0, theta, phi, 'XYZ'))
  bone.position.x = x
  bone.position.y = y
  bone.position.z = z

  if (parent) {
    bone.position.y += parent.length
  }

  const branch = {
    level,
    length,
    theta,
    phi,
    bone,
  }

  if (isLast) {
    const geometry = new CircleGeometry(config.leaf_size)
    const material = new MeshStandardMaterial({ color: 0x00ff00 })
    material.side = DoubleSide
    const mesh = new Mesh(geometry, material)
    mesh.position.y = config.leaf_size
    mesh.castShadow = true
    bone.add(mesh)
    branch.mesh = mesh
  } else {
    const thickness = config.branch_thickness * Math.pow(1 - level / config.iterations, 1.5)
    const endThickness =
      config.branch_thickness * Math.pow(1 - (level + 1) / config.iterations, 1.5)
    const geometry = new CylinderGeometry(thickness, endThickness, length)
    const mesh = new Mesh(geometry, new MeshStandardMaterial({ color: 0x00ff00 }))
    mesh.position.y = length * 0.5
    mesh.rotateX(Math.PI)

    mesh.castShadow = true
    bone.add(mesh)
    branch.mesh = mesh
  }

  bone.initialRotation = bone.rotation
  scene.add(bone)
  bones.push(bone)
  if (parent) {
    parent.bone.add(bone)
  }

  branches.push(branch)
  return branch
}

function addLevel(n) {
  const openBranches = branches.filter((branch) => branch.level === n - 1)

  openBranches.forEach((branch) => {
    const isLast = n === config.iterations - 1
    const splits = isLast
      ? config.leaf_splits
      : config.splits -
        Math.round(branch.level / (config.iterations - 1)) * (config.splits - config.end_splits)

    // trunk continuation
    addNode({
      theta: 0,
      phi: 0,
      level: n,
      parent: branch,
    })

    // branches
    for (let i = 0; i < splits; i++) {
      const theta = deg2rad(
        137.5 * branch.level +
          config.branch_theta * i +
          config.theta_randomness * (Math.random() - 1)
      )
      const phi = deg2rad(config.branch_phi + config.phi_randomness * (Math.random() - 1))
      addNode({
        theta,
        phi,
        level: n,
        parent: branch,
      })
    }
  })
}

function buildTree() {
  addNode({
    theta: 0,
    phi: 0,
    level: 0,
    x: 0,
    z: -5,
  })
  addNode({
    theta: 0,
    phi: 0,
    level: 0,
    x: 5,
    z: 2.5,
  })

  addNode({
    theta: 0,
    phi: 0,
    level: 0,
    x: -5,
    z: 2.5,
  })
  for (let i = 1; i < config.iterations; i++) {
    addLevel(i)
  }
}

/**
 * Other Entities
 */
function addLights() {
  scene.add(new AmbientLight(0x404040))
  const sun = new SpotLight(0xffffff, 0.25)
  sun.castShadow = true
  sun.position.set(10, 25, 15)
  sun.shadow.mapSize.width = 4096
  sun.shadow.mapSize.height = 4096
  scene.add(sun)
  // scene.add(new CameraHelper(sun.shadow.camera))
}

function addBase() {
  const base = new Mesh(new PlaneGeometry(50, 50), new MeshStandardMaterial({ color: 0xcccccc }))
  base.receiveShadow = true
  base.rotateX(deg2rad(-90))
  scene.add(base)
}

function addHelper() {
  const origin = new Vector3(0, 0, 0)
  const xDir = new Vector3(1, 0, 0)
  const yDir = new Vector3(0, 1, 0)
  const zDir = new Vector3(0, 0, 1)
  const xHelper = new ArrowHelper(xDir, origin, 1, 0xff0000)
  const yHelper = new ArrowHelper(yDir, origin, 1, 0x00ff00)
  const zHelper = new ArrowHelper(zDir, origin, 1, 0x0000ff)
  scene.add(xHelper)
  scene.add(yHelper)
  scene.add(zHelper)
}

/**
 * Runtime
 */

function animate(t) {
  requestAnimationFrame(animate)
  controls.update()
  renderer.render(scene, camera)
}

function playGrowAnimation() {
  return new Promise((resolve) => {
    const tl = gsap.timeline({ repeat: 0, delay: 1.5 })

    for (let i = 0; i < config.iterations; i++) {
      const bonesInLevel = branches
        .filter((branch) => branch.level === i)
        .map((branch) => branch.bone)
      const boneScales = bonesInLevel.map((bone) => bone.scale)
      tl.from(
        boneScales,
        {
          x: 0,
          y: 0,
          z: 0,
          duration: 1,
        },
        '-=0.75'
      )
    }

    tl.call(() => {
      resolve()
    })
  })
}

function playMotion() {
  requestAnimationFrame(playMotion)

  const time = performance.now()
  const timeFraction = 0.01
  const noiseFraction = 0.1
  bones.forEach((bone, index) => {
    const worldPosition = new Vector3()
    bone.getWorldPosition(worldPosition)
    const noiseValue = new ImprovedNoise().noise(
      (worldPosition.x + time * timeFraction) * noiseFraction,
      (worldPosition.y + time * timeFraction) * noiseFraction,
      (worldPosition.z + time * timeFraction) * noiseFraction
    )
    bone.rotation.z = bone.initialRotation.z + deg2rad(noiseValue) * 0.1
    bone.rotation.y = bone.initialRotation.y + Math.sin(time * 0.01) * 0.001
  })
}

onMounted(async () => {
  init()

  addLights()
  addBase()
  // addHelper()
  buildTree()

  resize()
  animate()
  await playGrowAnimation()
  playMotion()
})
</script>

<template>
  <div class="page" ref="container"></div>
</template>

<style lang="scss">
.page {
  canvas {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
  }
}
</style>
