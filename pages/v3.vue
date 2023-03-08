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
  Euler,
} from 'three'
import { gsap } from 'gsap'
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'

import { deg2rad } from '~/helpers/deg2rad'

const config = {
  iterations: 7,
  splits: 1,
  end_splits: 3,

  branch_length: 2,
  branch_thickness: 2,

  branch_theta: 60,
  branch_phi: 45,

  theta_randomness: 0,
  phi_randomness: 0,

  length_exponent: 0.25,
}
const up = new Vector3(0, 1, 0)

/**
 * Scene initialization
 */
const container = ref(null)
const scene = new Scene()
const camera = new PerspectiveCamera(75, 1, 0.1, 2000)
camera.position.set(0, 1, -10)
const renderer = new WebGLRenderer({
  antialias: true,
})
const controls = new OrbitControls(camera, renderer.domElement)
controls.target = new Vector3(0, 6, 0)
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
// let skeleton

function addNode({ level, parent, theta, phi }) {
  const length =
    config.branch_length * Math.pow(1 - level / config.iterations, config.length_exponent)

  const bone = new Bone()
  bone.setRotationFromEuler(new Euler(0, theta, phi, 'XYZ'))

  if (parent) {
    bone.position.y = parent.length
  }

  const thickness = config.branch_thickness * Math.pow(1 - level / config.iterations, 0.5)
  const endThickness = config.branch_thickness * Math.pow(1 - (level + 1) / config.iterations, 0.5)

  // const geometry = new CylinderGeometry(thickness, endThickness, length)
  // const mesh = new Mesh(geometry, new MeshStandardMaterial({ color: 0x00ff00 }))
  // mesh.position.y = length * 0.5
  // mesh.rotateX(Math.PI)

  const geometry = new SphereGeometry(thickness)
  const mesh = new Mesh(
    geometry,
    new MeshStandardMaterial({ color: 0xffffff, transparent: true, opacity: 1 })
  )
  mesh.position.y = length

  tree.add(mesh)
  bone.add(mesh)
  scene.add(bone)
  bones.push(bone)
  if (parent) {
    parent.bone.add(bone)
  }

  const branch = {
    // origin,
    // dir,
    // end,
    level,
    length,
    theta,
    phi,
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

let root
function buildTree() {
  root = addNode({ theta: 0, phi: 0, level: 0 })

  for (let i = 1; i < config.iterations; i++) {
    addLevel(i)
  }

  // const skeleton = new Skeleton(bones)
  // const skeletonHelper = new SkeletonHelper(bones[0])
  // scene.add(skeletonHelper)
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

function playAnimation() {
  const tl = gsap.timeline({ repeat: -1 })

  for (let i = 0; i < config.iterations; i++) {
    const materialsInLevel = branches
      .filter((branch) => branch.level === i)
      .map((branch) => branch.mesh.material)
    const levelStartTime = i * 0.1

    tl.from(
      materialsInLevel,
      {
        opacity: 0,
        duration: 0.25,
      },
      levelStartTime
    )
    tl.to(
      materialsInLevel,
      {
        opacity: 0,
        duration: 0.25,
      },
      levelStartTime + 0.35
    )

    // const bonesInLevel = branches
    //   .filter((branch) => branch.level === i)
    //   .map((branch) => branch.bone)
    // const boneScales = bonesInLevel.map((bone) => bone.scale)
    // tl.from(
    //   boneScales,
    //   {
    //     x: 0,
    //     y: 0,
    //     z: 0,
    //     duration: 1,
    //   },
    //   '-=0.75'
    // )
    // gsap.from(bonePositions, {
    //   y: 0.5,
    //   duration: 1,
    // })
  }
}

onMounted(() => {
  init()

  addLights()
  // addHelper()
  buildTree()

  resize()
  animate()
  // playAnimation()
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
