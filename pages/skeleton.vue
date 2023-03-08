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
  MeshStandardMaterial,
  CylinderGeometry,
  SphereGeometry,
  PlaneGeometry,
  Skeleton,
  Bone,
  ArrowHelper,
  MeshBasicMaterial,
  SkeletonHelper,
  Group,
  Vector2,
} from 'three'
import { gsap } from 'gsap'
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'

import { deg2rad } from '~/helpers/deg2rad'

const config = {
  iterations: 8,
  splits: 1,
  end_splits: 3,

  branch_length: 100,
  branch_thickness: 25,

  branch_theta: 45,
  branch_phi: 45,

  theta_randomness: 0,
  phi_randomness: 25,

  length_exponent: 0.5,
}
const up = new Vector3(0, 1, 0)

/**
 * Scene initialization
 */
const container = ref(null)
const scene = new Scene()
const camera = new PerspectiveCamera(75, 1, 0.1, 2000)
camera.position.set(0, 100, -650)
const renderer = new WebGLRenderer({
  antialias: true,
})
const controls = new OrbitControls(camera, renderer.domElement)
controls.target = new Vector3(0, 300, 0)
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
 * Entities
 */
const tree = new Group()
const branches = []
const bones = []
let skeleton

function addNode({ origin, dir, level, parent }) {
  const length =
    config.branch_length * Math.pow(1 - level / config.iterations, config.length_exponent)

  const end = origin.clone().add(dir.clone().multiplyScalar(length))
  const cross = dir.clone().cross(up).normalize()
  const angle = dir.clone().angleTo(up)

  const bonePos = dir.clone().multiplyScalar(length)
  const bone = new Bone()
  bone.position.x = bonePos.x
  bone.position.y = bonePos.y
  bone.position.z = bonePos.z
  bone.setRotationFromAxisAngle(cross, angle * -1)

  const thickness = config.branch_thickness * Math.pow(1 - level / config.iterations, 1.5)
  const endThickness = config.branch_thickness * Math.pow(1 - (level + 1) / config.iterations, 1.5)

  const geometry = new CylinderGeometry(thickness, endThickness, length)
  const mesh = new Mesh(geometry, new MeshStandardMaterial({ color: 0x00ff00 }))
  const meshPos = new Vector3(0, -0.5, 0).multiplyScalar(length)
  mesh.rotateX(Math.PI)
  mesh.position.x = meshPos.x
  mesh.position.y = meshPos.y
  mesh.position.z = meshPos.z

  tree.add(mesh)
  bone.add(mesh)
  scene.add(bone)
  bones.push(bone)
  if (parent) {
    parent.bone.add(bone)
  }

  const branch = {
    origin,
    dir,
    end,
    level,
    bone,
    mesh,
  }

  branches.push(branch)
  return branch
}

function addLevel(n) {
  const openBranches = branches.filter((branch) => branch.level === n - 1)

  openBranches.forEach((branch) => {
    const splits =
      config.splits -
      Math.round(branch.level / (config.iterations - 1)) * (config.splits - config.end_splits)

    // trunk continuation
    addNode({
      origin: branch.end,
      dir: new Vector3(0, 1, 0),
      level: n,
      parent: branch,
    })

    // branches
    for (let i = 0; i < splits; i++) {
      const iProgress = i / splits
      const theta = deg2rad(config.branch_theta + config.theta_randomness * (Math.random() - 1))
      const initialPhi = 137.5 * (branch.level + 1)
      const phi = deg2rad(
        initialPhi + config.branch_phi * i + config.phi_randomness * (Math.random() - 1)
      )
      const dir = new Vector3().setFromSpherical(new Spherical(1, theta, phi))
      // const cross = branch.dir.clone().cross(up).normalize()
      // const angle = branch.dir.clone().angleTo(up)
      // dir.applyAxisAngle(cross, angle * -1)

      // const dir = new Vector3(0, 1, 0)
      //   const length =
      //     config.branch_length *
      //     Math.pow(1 - branch.level / config.iterations, 1.5) *
      //     (1 + (Math.random() * 2 - 1) * config.length_randomness)
      addNode({
        origin: branch.end,
        dir,
        level: n,
        parent: branch,
        // length,
      })
    }
  })
}

let root
function buildTree() {
  // initial branch
  const origin = new Vector3(0, 0, 0)
  const dir = new Vector3(0, 1, 0)

  root = addNode({ origin, dir, level: 0 })
  for (let i = 1; i < config.iterations; i++) {
    addLevel(i)
  }

  skeleton = new Skeleton(bones)
}

/**
 * Lights
 */
function addLights() {
  scene.add(new AmbientLight(0x404040))
  const sun = new DirectionalLight(0xffffff, 0.5)
  sun.position.set(-250, 150, -250)
  scene.add(sun)
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
  const tl = gsap.timeline({ repeat: -1 })

  for (let i = 0; i < config.iterations; i++) {
    const bonesInLevel = branches
      .filter((branch) => branch.level === i)
      .map((branch) => branch.bone)
    const boneScales = bonesInLevel.map((bone) => bone.scale)
    const bonePositions = bonesInLevel.map((bone) => bone.position)
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
    // gsap.from(bonePositions, {
    //   y: 0.5,
    //   duration: 1,
    // })
  }
}

onMounted(() => {
  init()

  addLights()
  buildTree()

  resize()
  animate()
  playGrowAnimation()
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
