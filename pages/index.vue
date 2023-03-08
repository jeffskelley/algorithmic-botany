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
  PlaneGeometry,
} from 'three'
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'

import { deg2rad } from '~/helpers/deg2rad'

const config = {
  iterations: 6,
  splits: 5,
  branch_length: 100,
  branch_thickness: 10,
  branch_angle: 25,

  end_splits: 3,

  theta_randomness: 0.05,
  phi_randomness: 0,
  length_randomness: 0.2,
}
const up = new Vector3(0, 1, 0)

/**
 * Scene initialization
 */
const container = ref(null)
const scene = new Scene()
const camera = new PerspectiveCamera(75, 1, 0.1, 2000)
camera.position.set(0, 100, -550)
const renderer = new WebGLRenderer({
  antialias: true,
})
const controls = new OrbitControls(camera, renderer.domElement)
controls.target = new Vector3(0, 150, 0)
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
const branches = []
function addBranch({
  origin,
  dir,
  level,
  thickness = config.branch_thickness,
  endThickness = config.branch_thickness,
  length = config.branch_length,
}) {
  const end = origin.clone().add(dir.clone().multiplyScalar(length))
  const geometry = new CylinderGeometry(endThickness, thickness, length)
  const mesh = new Mesh(geometry, new MeshStandardMaterial({ color: 0x00ff00 }))
  const center = origin.clone().lerp(end, 0.5)

  mesh.position.x = center.x
  mesh.position.y = center.y
  mesh.position.z = center.z

  const cross = dir.clone().cross(up).normalize()
  const angle = dir.clone().angleTo(up)
  mesh.setRotationFromAxisAngle(cross, angle * -1)
  scene.add(mesh)

  branches.push({
    origin,
    dir,
    end,
    level,
    // thickness,
    // endThickness,
    // length,
    // mesh,
  })
}

function addLevel(n) {
  const openBranches = branches.filter((branch) => branch.level === n - 1)

  openBranches.forEach((branch) => {
    const splits =
      config.splits -
      Math.round(branch.level / (config.iterations - 1)) * (config.splits - config.end_splits)

    for (let i = 0; i < splits; i++) {
      const iProgress = i / splits

      const theta = deg2rad(
        config.branch_angle + 360 * config.theta_randomness * (Math.random() * 2 - 1)
      )
      const phi = deg2rad(360 * iProgress + 360 * config.phi_randomness * (Math.random() * 2 - 1))
      const dir = new Vector3().setFromSpherical(new Spherical(1, theta, phi))

      const cross = branch.dir.clone().cross(up).normalize()
      const angle = branch.dir.clone().angleTo(up)
      dir.applyAxisAngle(cross, angle * -1)

      const thickness = config.branch_thickness * Math.pow(1 - branch.level / config.iterations, 3)
      const endThickness =
        config.branch_thickness * Math.pow(1 - (branch.level + 1) / config.iterations, 3)
      const length =
        config.branch_length *
        Math.pow(1 - branch.level / config.iterations, 1.5) *
        (1 + (Math.random() * 2 - 1) * config.length_randomness)

      addBranch({
        origin: branch.end,
        dir,
        level: n,
        thickness,
        endThickness,
        length,
      })
    }
  })
}

function buildTree() {
  // initial branch
  const origin = new Vector3(0, 0, 0)
  const dir = new Vector3(0, 1, 0)
  addBranch({ origin, dir, level: 0 })
  for (let i = 1; i < config.iterations; i++) {
    addLevel(i)
  }
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

function animate() {
  requestAnimationFrame(animate)

  controls.update()
  renderer.render(scene, camera)
}

onMounted(() => {
  init()

  addLights()
  buildTree()

  resize()
  animate()
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
