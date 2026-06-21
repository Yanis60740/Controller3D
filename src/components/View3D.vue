<template>
  <div ref="canvasContainer" class="canvas-container"></div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js'
import { EXRLoader } from 'three/addons/loaders/EXRLoader.js'
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'
import GUI from 'lil-gui'

// ==========================================
// CONFIGURATION PAR NOM DE PIÈCE (Distance + Axe)
// ==========================================
const EXPLOSION_CONFIG = {
  el_2:  { dist: 40,  axis: 'z' }, 
  el_3:  { dist: 60,  axis: 'z' }, 
  el_4:  { dist: 60,  axis: 'z' }, 
  el_5:  { dist: 100, axis: 'z' }, 
  el_6:  { dist: 150, axis: 'z' }, 
  el_7:  { dist: 170, axis: 'z' }, 
  el_8:  { dist: 150, axis: 'z' }, 
  el_9:  { dist: 150, axis: 'z' }, 
  el_10: { dist: 190, axis: 'z' }, 
  el_11: { dist: 200, axis: 'z' }, 
  el_12: { dist: 220, axis: 'z' }, 
  el_13: { dist: 230, axis: 'z' }, 
  el_14: { dist: 270, axis: 'z' }  
}
const EXPLOSION_DURATION = 1.2 
// ==========================================

const gui = new GUI()
const canvasContainer = ref(null)

let scene, camera, renderer, animationFrameId, controls, dracoLoader
let explodedElements = [] 
let clock

const raycaster = new THREE.Raycaster()
const mouse = new THREE.Vector2()

let isExploded = false          
let isTravellingFinished = false 
let explosionProgress = 0 
let lastCameraOffset = 0 // 🎯 Stocke le recul appliqué au frame précédent

const easeInOutCubic = (t) => t < 0.5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2

const createFloorAlphaMap = () => {
  const size = 512
  const canvas = document.createElement('canvas')
  canvas.width = size
  canvas.height = size
  const ctx = canvas.getContext('2d')
  
  const gradient = ctx.createRadialGradient(size / 2, size / 2, 0, size / 2, size / 2, size / 2)
  gradient.addColorStop(0, '#ffffff')   
  gradient.addColorStop(0.4, '#ffffff')  
  gradient.addColorStop(1, '#000000')   
  ctx.fillStyle = gradient
  ctx.fillRect(0, 0, size, size)
  return new THREE.CanvasTexture(canvas)
}

let pointerDownTime = 0
const onPointerDown = () => { pointerDownTime = performance.now() }

const onPointerUp = (event) => {
  if (!isTravellingFinished || !canvasContainer.value) return
  if (performance.now() - pointerDownTime > 200) return

  const rect = canvasContainer.value.getBoundingClientRect()
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1

  raycaster.setFromCamera(mouse, camera)
  const intersects = raycaster.intersectObjects(scene.children, true)

  if (intersects.length > 0) {
    const hitModel = intersects.some(hit => hit.object !== scene.children.find(c => c.isMesh && c.geometry.type === 'PlaneGeometry'))
    if (hitModel) isExploded = !isExploded
  }
}

onMounted(() => {
  scene = new THREE.Scene()
  scene.background = new THREE.Color('#1a1a1a')
  
  clock = new THREE.Clock(false) 

  setTimeout(() => {
    clock.start()
  }, 3000)

  camera = new THREE.PerspectiveCamera(75, canvasContainer.value.clientWidth / canvasContainer.value.clientHeight, 0.1, 1000)
  camera.position.set(0, 45, 35)

  const cameraFolder = gui.addFolder('Camera')
  cameraFolder.add(camera.position, 'y', -50, 50, 0.5).name('Hauteur Caméra')
  cameraFolder.add(camera.position, 'x', -50, 50, 0.5).name('Position X Caméra')
  cameraFolder.add(camera.position, 'z', -50, 50, 0.5).name('Position Z Caméra')

  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(canvasContainer.value.clientWidth, canvasContainer.value.clientHeight)
  renderer.shadowMap.enabled = true 
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  renderer.outputColorSpace = THREE.SRGBColorSpace
  
  renderer.toneMapping = THREE.ACESFilmicToneMapping 
  renderer.toneMappingExposure = 1.0 
  gui.add(renderer, 'toneMappingExposure', 0, 3, 0.1).name('Exposition HDRI')
  
  canvasContainer.value.appendChild(renderer.domElement)

  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true 
  controls.dampingFactor = 0.05
  controls.target.set(0, 0, 0)
  controls.enableZoom = true
  controls.minDistance = 15   
  controls.maxDistance = 150  
  controls.enabled = false    

  new EXRLoader()
    .load('/forest.exr', (texture) => { 
      texture.mapping = THREE.EquirectangularReflectionMapping
      scene.environment = texture
    }, undefined, (error) => {
      console.error('Erreur lors du chargement du fichier EXR :', error)
    })

  const floorMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x151515, roughness: 0.8, alphaMap: createFloorAlphaMap(), transparent: true                
  })
  const floor = new THREE.Mesh(new THREE.PlaneGeometry(100, 100), floorMaterial)
  floor.rotation.x = -Math.PI / 2 
  floor.receiveShadow = true 
  scene.add(floor)

  const floorFolder = gui.addFolder('Réglages du Sol')
  floorFolder.add(floor.position, 'y', -20, 5, 0.1).name('Hauteur Sol')
  floorFolder.add(floorMaterial, 'roughness', 0, 1, 0.05).name('Rugosité')
  
  const modelContainer = new THREE.Group() 
  scene.add(modelContainer)

  dracoLoader = new DRACOLoader()
  dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.6/')
  
  const loader = new GLTFLoader()
  loader.setDRACOLoader(dracoLoader)
  
  loader.load(
    '/controller.glb', 
    (gltf) => {
      const model = gltf.scene
      explodedElements = []

      const box = new THREE.Box3().setFromObject(model)
      const center = new THREE.Vector3()
      box.getCenter(center)
      model.position.sub(center) 
      modelContainer.add(model)
      model.updateMatrixWorld(true)

      for (let i = 2; i <= 14; i++) {
        const name = `el_${i}`
        const targetMesh = model.getObjectByName(name)
        if (targetMesh) {
          const itemConfig = EXPLOSION_CONFIG[name] || { dist: 0, axis: 'z' }
          
          targetMesh.userData.axis = itemConfig.axis || 'z'
          targetMesh.userData.originalPos = targetMesh.position[targetMesh.userData.axis]
          targetMesh.userData.explodeOffset = itemConfig.dist || 0
          
          explodedElements.push(targetMesh)
        }
      }

      model.traverse((child) => {
        if (child.isMesh) {
          child.castShadow = true
          child.receiveShadow = true
        }
      })
      
      const totalSize = new THREE.Vector3()
      box.getSize(totalSize)
      floor.position.y = -totalSize.y / 2
      floorFolder.controllers.forEach(c => c.updateDisplay())

      const modelFolder = gui.addFolder('Contrôleur')
      modelFolder.add(modelContainer.position, 'x', -30, 30, 0.2).name('Position X')
      modelFolder.add(modelContainer.position, 'y', -30, 10, 0.2).name('Position Y')
      modelFolder.add(modelContainer.position, 'z', -50, 10, 0.2).name('Position Z')
      modelFolder.add(modelContainer.rotation, 'x', 0, Math.PI * 2, 0.02).name('Rotation X')
      modelFolder.add(modelContainer.rotation, 'y', 0, Math.PI * 2, 0.02).name('Rotation Y')
      modelFolder.add(modelContainer.rotation, 'z', 0, Math.PI * 2, 0.02).name('Rotation Z')
      
      modelContainer.rotation.set(0, 3.7, 0)
    },
    undefined,
    (error) => console.error('Erreur de chargement du GLB :', error)
  )
    
  // 🎯 Fonction helper pour réinitialiser et synchroniser OrbitControls sans saut
  const initOrbitControls = () => {
    if (controls) controls.dispose() // On nettoie l'ancienne instance
    controls = new OrbitControls(camera, renderer.domElement)
    controls.enableDamping = true 
    controls.dampingFactor = 0.05
    controls.target.set(0, 0, 0)
    controls.enableZoom = true
    controls.minDistance = 15   
    controls.maxDistance = 150  
    controls.enabled = true
  }
    
  const animate = () => {
    animationFrameId = requestAnimationFrame(animate)
    const deltaTime = clock.getDelta()
    
    // GESTION PROGRESSION EXPLOSION
    if (isExploded) {
      explosionProgress = Math.min(1, explosionProgress + deltaTime / EXPLOSION_DURATION)
    } else {
      explosionProgress = Math.max(0, explosionProgress - deltaTime / EXPLOSION_DURATION)
    }
    const easedFactor = easeInOutCubic(explosionProgress)
    const targetCameraRecoil = 30 * easedFactor

    // 1. TRAVELLING CAMÉRA PARFAITEMENT FLUIDE
    if (!isTravellingFinished) {
      const duration = Math.PI * 2 // Durée totale calée sur l'angle de rotation
      const elapsed = clock.getElapsedTime() * 2
      const travellingProgress = Math.min(1, elapsed / duration)
      const easedTravelling = easeInOutCubic(travellingProgress)
      
      // Déclenchement de l'explosion à 75% du travelling (équivalent à l'ancien Math.PI * 1.5)
      if (travellingProgress >= 0.75 && !isExploded) {
        isExploded = true
      }

      if (travellingProgress >= 1) {
        // --- FRAME DE TRANSITION EXACT ---
        isTravellingFinished = true
        
        // Position de base finale (angle = 2PI -> sin=0, cos=1)
        camera.position.set(0, 20, 35)
        camera.lookAt(0, 0, 0)
        
        // On initialise OrbitControls sur cette position exacte !
        initOrbitControls()
        
        // On applique le recul de l'explosion sur ce premier frame interactif
        if (targetCameraRecoil > 0) {
          const dir = camera.position.clone().normalize()
          camera.position.addScaledVector(dir, targetCameraRecoil)
        }
        lastCameraOffset = targetCameraRecoil
      } else {
        // --- PENDANT LE TRAVELLING ---
        const angle = easedTravelling * Math.PI * 2
        
        // Position théorique sur la courbe (sans recul)
        const targetX = Math.sin(angle) * 35
        const targetY = 45 + (20 - 45) * easedTravelling
        const targetZ = Math.cos(angle) * 35
        camera.position.set(targetX, targetY, targetZ)

        // Application du recul dans l'axe de la caméra (aligné avec la phase OrbitControls)
        if (targetCameraRecoil > 0) {
          const dir = camera.position.clone().normalize()
          camera.position.addScaledVector(dir, targetCameraRecoil)
        }
        
        cameraFolder.controllers.forEach(c => c.updateDisplay())
        camera.lookAt(0, 0, 0)
      }
    } else {
      // --- MODE INTERACTIF (ORBIT CONTROLS) ---
      // On retire le recul du frame précédent avant l'update de la souris
      if (lastCameraOffset > 0) {
        const dir = new THREE.Vector3().copy(camera.position).sub(controls.target).normalize()
        camera.position.addScaledVector(dir, -lastCameraOffset)
      }

      controls.target.set(0, 0, 0) 
      controls.update()

      // On réapplique le recul actuel après l'update pour le rendu visuel
      if (targetCameraRecoil > 0) {
        const dir = new THREE.Vector3().copy(camera.position).sub(controls.target).normalize()
        camera.position.addScaledVector(dir, targetCameraRecoil)
      }
      lastCameraOffset = targetCameraRecoil 
    }

    // 2. LE MODÈLE RESTE FIXE AU CENTRE
    modelContainer.position.z = 0 

    // 3. MOUVEMENT INDIVIDUEL DES PIÈCES
    explodedElements.forEach((el) => {
      const axis = el.userData.axis 
      const targetPos = el.userData.originalPos + (el.userData.explodeOffset * easedFactor)
      el.position[axis] = THREE.MathUtils.lerp(el.position[axis], targetPos, 0.1)
    })

    renderer.render(scene, camera)
  }
  animate()

  window.addEventListener('resize', onWindowResize)
  window.addEventListener('pointerdown', onPointerDown)
  window.addEventListener('pointerup', onPointerUp)
})

const onWindowResize = () => {
  if (!canvasContainer.value) return
  camera.aspect = canvasContainer.value.clientWidth / canvasContainer.value.clientHeight
  camera.updateProjectionMatrix()
  renderer.setSize(canvasContainer.value.clientWidth, canvasContainer.value.clientHeight)
}

onBeforeUnmount(() => {
  cancelAnimationFrame(animationFrameId)
  window.removeEventListener('resize', onWindowResize)
  window.removeEventListener('pointerdown', onPointerDown)
  window.removeEventListener('pointerup', onPointerUp)
  
  if (dracoLoader) dracoLoader.dispose() 
  if (controls) controls.dispose()
  if (renderer) renderer.dispose()
  if (gui) gui.destroy()
})
</script>

<style scoped>
.canvas-container {
  width: 100%;
  height: 100vh; 
  overflow: hidden;
  background-color: #1a1a1a;
  cursor: grab;
}
.canvas-container:active {
  cursor: grabbing;
}
</style>