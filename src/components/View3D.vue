<template>
  <div ref="canvasContainer" class="canvas-container">
    <transition name="fade">
      <div v-if="isLoading" class="loader-overlay">
        <div class="loader-content">
          <h2 class="loader-title">Vectron Initialisation</h2>
          <div class="progress-bar-container">
            <div class="progress-bar" :style="{ width: loadingProgress + '%' }"></div>
          </div>
          <span class="progress-text">{{ loadingProgress }}%</span>
        </div>
      </div>
    </transition>

    <div v-if="!isLoading" class="brand-overlay">
      <h1 class="brand-title">
        <span 
          v-for="(char, index) in 'Vectron'" 
          :key="'title-' + index"
          class="anim-letter"
          :style="{ animationDelay: `${index * 0.1}s` }"
        >{{ char }}</span>
      </h1>
      <p class="brand-description">
        <span 
          v-for="(char, index) in 'Manette de précision modulaire conceptuelle'" 
          :key="'desc-' + index"
          class="anim-letter"
          :style="{ animationDelay: `${(index * 0.02) + 0.6}s` }"
        >{{ char }}</span>
      </p>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue";
import * as THREE from "three";
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import { DRACOLoader } from "three/addons/loaders/DRACOLoader.js";
import { EXRLoader } from "three/addons/loaders/EXRLoader.js";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
import {
  CSS2DRenderer,
  CSS2DObject,
} from "three/addons/renderers/CSS2DRenderer.js";

// ==========================================
// CONFIGURATION PAR NOM DE PIÈCE
// ==========================================
const EXPLOSION_CONFIG = {
  el_2: { dist: 40, axis: "z", desc: "Composant Principal", detail: "Unité centrale logeant la carte mère principale et les bus de communication inter-modules.", posX: -270, posY: 22, anchorX: -50, anchorY: 0, anchorZ: 0 },
  el_3: { dist: 60, axis: "z", desc: "Connectique/Processeur", detail: "Processeur ARM haute fréquence pour un polling-rate à 4000Hz sans latence.", posX: 270, posY: 22, anchorX: 40, anchorY: 0, anchorZ: 10 },
  el_4: { dist: 70, axis: "z", desc: "Joystick", detail: "Mécanisme à effet Hall anti-drift avec tension de retour ajustable physiquement.", posX: -270, posY: 0, anchorX: 0, anchorY: 0, anchorZ: 30 },
  el_5: { dist: 100, axis: "z", desc: "Protection Joystick", detail: "Bague anti-friction en téflon interchangeable pour une glisse fluide du stick.", posX: 270, posY: 10, anchorX: 0, anchorY: 0, anchorZ: 30 },
  el_6: { dist: 150, axis: "z" },
  el_7: { dist: 170, axis: "z", desc: "Liaison Joystick/Manche", detail: "Axe renforcé en alliage d'aluminium aéronautique supportant les torsions lourdes.", posX: -270, posY: 6, anchorX: 0, anchorY: 0, anchorZ: 0 },
  el_8: { dist: 150, axis: "z" },
  el_9: { dist: 150, axis: "z" },
  el_10: { dist: 190, axis: "z", desc: "Manche", detail: "Grip ergonomique texturé interchangeable s'adaptant à toutes les morphologies de main.", posX: -270, posY: -2, anchorX: 0, anchorY: 0, anchorZ: 0 },
  el_11: { dist: 200, axis: "z", desc: "Capteur Boutons", detail: "Nappe de circuits imprimés flexibles transmettant instantanément l'actionnement.", posX: 270, posY: -18, anchorX: 0, anchorY: 0, anchorZ: 80 },
  el_12: { dist: 220, axis: "z", desc: "Switchs Boutons", detail: "Interrupteurs optiques certifiés pour 100 millions de clics avec retour tactile franc.", posX: -270, posY: -10, anchorX: -12, anchorY: -3, anchorZ: 62 },
  el_13: { dist: 230, axis: "z", desc: "Boutons", detail: "Touches mécaniques à course ultra-courte (0.3mm) pour une réactivité maximale.", posX: -270, posY: -18, anchorX: -12, anchorY: -3, anchorZ: 50 },
  el_14: { dist: 270, axis: "z" },
};
const EXPLOSION_DURATION = 1.2;
// ==========================================

const canvasContainer = ref(null);

const isLoading = ref(true);
const loadingProgress = ref(0);

let scene, camera, renderer, labelRenderer, animationFrameId, controls, dracoLoader;
let explodedElements = [];
let clock;

const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

let isExploded = false;
let isTravellingFinished = false;
let explosionProgress = 0;
let lastCameraOffset = 0;

let focusedMesh = null;
let preFocusCameraPos = new THREE.Vector3();
let preFocusControlsTarget = new THREE.Vector3();
let targetCameraPos = new THREE.Vector3();
let targetControlsTarget = new THREE.Vector3();
let isFocusing = false;
const FOCUS_DURATION = 0.7; 
let focusProgress = 1;
let startCameraPos = new THREE.Vector3();
let startControlsTarget = new THREE.Vector3();

const easeInOutCubic = (t) =>
  t < 0.5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2;

const createFloorAlphaMap = () => {
  const size = 512;
  const canvas = document.createElement("canvas");
  canvas.width = size; canvas.height = size;
  const ctx = canvas.getContext("2d");
  const gradient = ctx.createRadialGradient(size/2, size/2, 0, size/2, size/2, size/2);
  gradient.addColorStop(0, "#ffffff");
  gradient.addColorStop(0.4, "#ffffff");
  gradient.addColorStop(1, "#000000");
  ctx.fillStyle = gradient;
  ctx.fillRect(0, 0, size, size);
  return new THREE.CanvasTexture(canvas);
};

let pointerDownTime = 0;
const onPointerDown = () => {
  pointerDownTime = performance.now();
};

const onPointerUp = (event) => {
  if (event.target.closest(".annotation-box")) return;

  if (!isTravellingFinished || !canvasContainer.value) return;
  if (performance.now() - pointerDownTime > 200) return;

  const rect = canvasContainer.value.getBoundingClientRect();
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

  raycaster.setFromCamera(mouse, camera);
  const intersects = raycaster.intersectObjects(scene.children, true);

  if (intersects.length > 0) {
    const hitModel = intersects.some(
      (hit) => hit.object !== scene.children.find((c) => c.isMesh && c.geometry.type === "PlaneGeometry")
    );
    
    if (hitModel && focusedMesh) {
      resetFocus();
    } else if (hitModel) {
      isExploded = !isExploded;
    }
  } else if (focusedMesh) {
    resetFocus();
  }
};

const resetFocus = () => {
  if (!focusedMesh) return; 
  
  focusProgress = 0;
  startCameraPos.copy(camera.position);
  startControlsTarget.copy(controls.target);
  
  focusedMesh = null;
  isFocusing = true;
  
  explodedElements.forEach((el) => {
    if (el.userData.labelDiv) {
      el.userData.labelDiv.classList.remove("active-card", "left-side-card", "right-side-card");
      updateLabelHTML(el, false);
      
      const config = EXPLOSION_CONFIG[el.name];
      if (config) {
        const positions = el.userData.line.geometry.attributes.position.array;
        positions[3] = config.posX;
        positions[4] = config.posY;
        positions[5] = 0;
        el.userData.line.geometry.attributes.position.needsUpdate = true;
      }
    }
  });
};

const focusOnElement = (mesh) => {
  if (!isTravellingFinished) return;
  
  if (focusedMesh === mesh) {
    resetFocus();
  } else {
    if (!focusedMesh) {
      preFocusCameraPos.copy(camera.position);
      preFocusControlsTarget.copy(controls.target);
    }
    if (lastCameraOffset > 0) {
      const dir = new THREE.Vector3().copy(preFocusCameraPos).sub(preFocusControlsTarget).normalize();
      preFocusCameraPos.addScaledVector(dir, -lastCameraOffset);
    }
        
    focusProgress = 0;
    startCameraPos.copy(camera.position);
    startControlsTarget.copy(controls.target);

    focusedMesh = mesh;
    isFocusing = true;

    const wp = new THREE.Vector3();
    mesh.getWorldPosition(wp);
    targetControlsTarget.copy(wp);

    targetCameraPos.copy(wp).add(new THREE.Vector3(0, 2, 25));

    explodedElements.forEach((el) => {
      if (el.userData.labelDiv) {
        if (el === mesh) {
          updateLabelHTML(el, true);
        } else {
          el.userData.labelDiv.classList.remove("active-card", "left-side-card", "right-side-card");
        }
      }
    });
  }
};

const updateLabelHTML = (el, isCardMode) => {
  const config = EXPLOSION_CONFIG[el.name];
  if (!config || !el.userData.labelDiv) return;

  if (isCardMode) {
    const sideClass = config.posX > 0 ? "left-side-card" : "right-side-card";
    el.userData.labelDiv.className = `annotation-box active-card ${sideClass}`;
    el.userData.labelDiv.innerHTML = `
      <div class="card-header">
        <span class="card-title">${config.desc}</span>
        <button class="card-close-btn">✕</button>
      </div>
      <div class="card-body">
        ${config.detail || "Aucune description supplémentaire disponible."}
      </div>
    `;
    
    const closeBtn = el.userData.labelDiv.querySelector(".card-close-btn");
    if (closeBtn) {
      closeBtn.addEventListener("click", (e) => {
        e.stopPropagation();
        resetFocus();
      });
    }
  } else {
    el.userData.labelDiv.className = "annotation-box " + (config.posX > 0 ? "right-side" : "left-side");
    if (config.posX > 0) {
      el.userData.labelDiv.innerHTML = `${config.desc} <span class="dot"></span>`;
    } else {
      el.userData.labelDiv.innerHTML = `<span class="dot"></span> ${config.desc}`;
    }
  }
};

onMounted(() => {
  scene = new THREE.Scene();
  scene.background = new THREE.Color("#1a1a1a");
  clock = new THREE.Clock(false);

  const manager = new THREE.LoadingManager();
  
  manager.onProgress = (url, itemsLoaded, itemsTotal) => {
    loadingProgress.value = Math.round((itemsLoaded / itemsTotal) * 100);
  };
  
  manager.onLoad = () => {
    setTimeout(() => {
      isLoading.value = false;
      clock.start(); 
    }, 600);
  };

  camera = new THREE.PerspectiveCamera(75, canvasContainer.value.clientWidth / canvasContainer.value.clientHeight, 0.1, 1000);
  camera.position.set(0, 45, 45);

  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(canvasContainer.value.clientWidth, canvasContainer.value.clientHeight);
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;
  renderer.outputColorSpace = THREE.SRGBColorSpace;
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  renderer.toneMappingExposure = 1.0;
  canvasContainer.value.appendChild(renderer.domElement);

  labelRenderer = new CSS2DRenderer();
  labelRenderer.setSize(canvasContainer.value.clientWidth, canvasContainer.value.clientHeight);
  labelRenderer.domElement.style.position = "absolute";
  labelRenderer.domElement.style.top = "0px";
  labelRenderer.domElement.style.pointerEvents = "auto"; 
  canvasContainer.value.appendChild(labelRenderer.domElement);

  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  controls.target.set(0, 0, 0);
  controls.enableZoom = true;
  controls.minDistance = 15;
  controls.maxDistance = 150;
  controls.enabled = false;

  new EXRLoader().load("/forest.exr", (texture) => {
    texture.mapping = THREE.EquirectangularReflectionMapping;
    scene.environment = texture;
  });

  const floorMaterial = new THREE.MeshStandardMaterial({
    color: 0x151515, roughness: 0.8, alphaMap: createFloorAlphaMap(), transparent: true,
  });
  const floor = new THREE.Mesh(new THREE.PlaneGeometry(100, 100), floorMaterial);
  floor.rotation.x = -Math.PI / 2;
  floor.receiveShadow = true;
  scene.add(floor);

  new EXRLoader(manager).load("/forest.exr", (texture) => {
    texture.mapping = THREE.EquirectangularReflectionMapping;
    scene.environment = texture;
  });

  const modelContainer = new THREE.Group();
  scene.add(modelContainer);

  dracoLoader = new DRACOLoader();
  dracoLoader.setDecoderPath("https://www.gstatic.com/draco/versioned/decoders/1.5.6/");

  const loader = new GLTFLoader(manager);
  loader.setDRACOLoader(dracoLoader);
  modelContainer.rotation.set(0, 3.7, 0);

  loader.load(
    "/controller.glb",
    (gltf) => {
      const model = gltf.scene;
      explodedElements = [];

      const box = new THREE.Box3().setFromObject(model);
      const center = new THREE.Vector3();
      box.getCenter(center);
      model.position.sub(center);
      modelContainer.add(model);
      model.updateMatrixWorld(true);

      for (let i = 2; i <= 14; i++) {
        const name = `el_${i}`;
        const targetMesh = model.getObjectByName(name);

        if (targetMesh) {
          const itemConfig = EXPLOSION_CONFIG[name] || { dist: 0, axis: "z", desc: `Pièce ${name}`, posX: 20, posY: 0 };

          targetMesh.name = name;
          targetMesh.userData.axis = itemConfig.axis || "z";
          targetMesh.userData.originalPos = targetMesh.position[targetMesh.userData.axis];
          targetMesh.userData.explodeOffset = itemConfig.dist || 0;

          const labelX = itemConfig.posX;
          const labelY = itemConfig.posY;
          const anchorOffset = new THREE.Vector3(itemConfig.anchorX || 0, itemConfig.anchorY || 0, itemConfig.anchorZ || 0);
          targetMesh.userData.anchorOffset = anchorOffset;

          const anchorGeo = new THREE.SphereGeometry(0.5, 16, 16);
          const anchorMat = new THREE.MeshBasicMaterial({ color: 0xffffff, transparent: true, opacity: 0 });
          const anchorPoint = new THREE.Mesh(anchorGeo, anchorMat);
          anchorPoint.position.copy(anchorOffset);
          targetMesh.add(anchorPoint);

          const annotationDiv = document.createElement("div");
          
          annotationDiv.addEventListener("click", (e) => {
            e.stopPropagation();
            if (!annotationDiv.classList.contains("active-card")) {
              focusOnElement(targetMesh);
            }
          });

          const annotationLabel = new CSS2DObject(annotationDiv);
          annotationLabel.position.set(labelX, labelY, 0);

          if (itemConfig.posX > 0) {
            annotationLabel.center.set(1, 0.5); 
          } else {
            annotationLabel.center.set(0, 0.5); 
          }

          targetMesh.add(annotationLabel);

          const lineMaterial = new THREE.LineBasicMaterial({ color: 0xffffff, transparent: true, opacity: 0 });
          const points = [anchorOffset, new THREE.Vector3(labelX, labelY, 0)];
          const lineGeometry = new THREE.BufferGeometry().setFromPoints(points);
          const line = new THREE.Line(lineGeometry, lineMaterial);
          line.visible = false;
          targetMesh.add(line);

          targetMesh.userData.labelDiv = annotationDiv;
          targetMesh.userData.line = line;
          targetMesh.userData.lineMaterial = lineMaterial;
          targetMesh.userData.anchorMat = anchorMat;

          updateLabelHTML(targetMesh, false);
          explodedElements.push(targetMesh);
        }
      }

      model.traverse((child) => {
        if (child.isMesh) { child.castShadow = true; child.receiveShadow = true; }
      });

      const totalSize = new THREE.Vector3();
      box.getSize(totalSize);
      floor.position.y = -totalSize.y / 2;
    },
    undefined, 
    (error) => console.error("Erreur de chargement du GLB :", error)
  );

  const initOrbitControls = () => {
    if (controls) controls.dispose();
    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.target.set(0, 0, 0);
    controls.enableZoom = true;
    controls.minDistance = 15;
    controls.maxDistance = 150;
    controls.enabled = true;
    
    controls.addEventListener('start', () => {
      if (isFocusing) isFocusing = false;
    });
  };

  const animate = () => {
    animationFrameId = requestAnimationFrame(animate);
    const deltaTime = clock.getDelta();

    if (isExploded) {
      explosionProgress = Math.min(1, explosionProgress + deltaTime / EXPLOSION_DURATION);
    } else {
      explosionProgress = Math.max(0, explosionProgress - deltaTime / EXPLOSION_DURATION);
    }
    const easedFactor = easeInOutCubic(explosionProgress);
    const targetCameraRecoil = 35 * easedFactor;

    explodedElements.forEach((el) => {
      const axis = el.userData.axis;
      const targetPos = el.userData.originalPos + el.userData.explodeOffset * easedFactor;
      el.position[axis] = THREE.MathUtils.lerp(el.position[axis], targetPos, 0.1);

      if (el.userData.labelDiv) {
        const opacityTarget = Math.max(0, (easedFactor - 0.5) * 2);
        
        if (focusedMesh) {
          if (focusedMesh === el) {
            el.userData.labelDiv.style.opacity = 1;
            el.userData.labelDiv.style.pointerEvents = "auto";
            el.userData.lineMaterial.opacity = 1;
            el.userData.anchorMat.opacity = 1;
            if (el.userData.line) el.userData.line.visible = true;
            el.userData.lineMaterial.depthTest = false;
            el.userData.line.renderOrder = 999;        

            const config = EXPLOSION_CONFIG[el.name];
            if (config && canvasContainer.value) {
              const rect = canvasContainer.value.getBoundingClientRect();
              
              let targetPixelX = 0;
              let targetPixelY = rect.height - 260; 

              if (config.posX > 0) {
                targetPixelX = 300; 
              } else {
                targetPixelX = rect.width - 300; 
              }

              const ndcX = (targetPixelX / rect.width) * 2 - 1;
              const ndcY = -(targetPixelY / rect.height) * 2 + 1;

              const meshWorldPos = new THREE.Vector3();
              el.getWorldPosition(meshWorldPos);
              const meshCamProjected = meshWorldPos.clone().project(camera);

              const targetWorldPos = new THREE.Vector3(ndcX, ndcY, meshCamProjected.z).unproject(camera);
              const targetLocalPos = targetWorldPos.clone();
              el.worldToLocal(targetLocalPos);

              const positions = el.userData.line.geometry.attributes.position.array;
              positions[3] = targetLocalPos.x;
              positions[4] = targetLocalPos.y;
              positions[5] = targetLocalPos.z;
              el.userData.line.geometry.attributes.position.needsUpdate = true;
            }

          } else {
            el.userData.labelDiv.style.opacity = 0;
            el.userData.labelDiv.style.pointerEvents = "none";
            el.userData.lineMaterial.opacity = 0;
            el.userData.anchorMat.opacity = 0;
            if (el.userData.line) el.userData.line.visible = false;
          }
        } else {
          el.userData.labelDiv.style.opacity = opacityTarget > 0.1 ? 1 : 0;
          el.userData.labelDiv.style.pointerEvents = opacityTarget > 0.1 ? "auto" : "none";
          el.userData.lineMaterial.opacity = opacityTarget;
          el.userData.anchorMat.opacity = opacityTarget;
          if (el.userData.line) el.userData.line.visible = opacityTarget > 0;
        }
      }
    });

    if (!isTravellingFinished) {
      const duration = Math.PI * 2;
      const elapsed = clock.getElapsedTime() * 2;
      const travellingProgress = Math.min(1, elapsed / duration);
      const easedTravelling = easeInOutCubic(travellingProgress);

      if (travellingProgress >= 0.75 && !isExploded) isExploded = true;

      if (travellingProgress >= 1) {
        isTravellingFinished = true;
        camera.position.set(0, 20, 45);
        camera.lookAt(0, 0, 0);
        initOrbitControls();

        if (targetCameraRecoil > 0) {
          const dir = camera.position.clone().normalize();
          camera.position.addScaledVector(dir, targetCameraRecoil);
        }
        lastCameraOffset = targetCameraRecoil;
      } else {
        const angle = easedTravelling * Math.PI * 2;
        const targetX = Math.sin(angle) * 45;
        const targetY = 45 + (20 - 45) * easedTravelling;
        const targetZ = Math.cos(angle) * 45;
        camera.position.set(targetX, targetY, targetZ);

        if (targetCameraRecoil > 0) {
          const dir = camera.position.clone().normalize();
          camera.position.addScaledVector(dir, targetCameraRecoil);
        }
        camera.lookAt(0, 0, 0);
      }
    } else {
      if (isFocusing) {
        focusProgress = Math.min(1, focusProgress + deltaTime / FOCUS_DURATION);
        
        const ease = easeInOutCubic(focusProgress);

        if (!focusedMesh) {
          targetControlsTarget.copy(preFocusControlsTarget);
          targetCameraPos.copy(preFocusCameraPos);
          if (targetCameraRecoil > 0) {
            const dir = new THREE.Vector3().copy(targetCameraPos).sub(targetControlsTarget).normalize();
            targetCameraPos.addScaledVector(dir, targetCameraRecoil);
          }
        }

        camera.position.lerpVectors(startCameraPos, targetCameraPos, ease);
        controls.target.lerpVectors(startControlsTarget, targetControlsTarget, ease);

        if (focusProgress >= 1) {
          isFocusing = false;
          
          if (!focusedMesh) {
            lastCameraOffset = targetCameraRecoil;
          }
        }
        
        controls.update();
      } else {
        if (focusedMesh) {
          const wp = new THREE.Vector3();
          focusedMesh.getWorldPosition(wp);
          controls.target.copy(wp);
        } else {
          if (lastCameraOffset > 0) {
            const dir = new THREE.Vector3().copy(camera.position).sub(controls.target).normalize();
            camera.position.addScaledVector(dir, -lastCameraOffset);
          }

          controls.update();

          if (targetCameraRecoil > 0) {
            const dir = new THREE.Vector3().copy(camera.position).sub(controls.target).normalize();
            camera.position.addScaledVector(dir, targetCameraRecoil);
          }
          lastCameraOffset = targetCameraRecoil;
        }
        controls.update();
      }
    }

    modelContainer.position.z = 0;
    renderer.render(scene, camera);
    labelRenderer.render(scene, camera);
  };
  animate();

  window.addEventListener("resize", onWindowResize);
  window.addEventListener("pointerdown", onPointerDown);
  window.addEventListener("pointerup", onPointerUp);
});

const onWindowResize = () => {
  if (!canvasContainer.value) return;
  const width = canvasContainer.value.clientWidth;
  const height = canvasContainer.value.clientHeight;
  camera.aspect = width / height;
  camera.updateProjectionMatrix();
  renderer.setSize(width, height);
  labelRenderer.setSize(width, height);
};

onBeforeUnmount(() => {
  cancelAnimationFrame(animationFrameId);
  window.removeEventListener("resize", onWindowResize);
  window.removeEventListener("pointerdown", onPointerDown);
  window.removeEventListener("pointerup", onPointerUp);
  if (dracoLoader) dracoLoader.dispose();
  if (controls) controls.dispose();
  if (renderer) renderer.dispose();
});
</script>

<style>
.annotation-box {
  background-color: transparent;
  color: #ffffff;
  font-family: "Courier New", Courier, monospace;
  font-size: 11px;
  letter-spacing: 1px;
  text-transform: uppercase;
  white-space: nowrap;
  user-select: none;
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  border-bottom: 1px solid rgba(255, 255, 255, 0.3);
  padding-bottom: 4px;
  /* Ajout de la transition */
  transition: color 0.2s ease-in-out, border-color 0.2s ease-in-out;
}

.annotation-box:hover {
  color: #f1c40f;
}

.annotation-box:hover .dot {
  background-color: #f1c40f;
  box-shadow: 0 0 8px rgba(241, 196, 15, 1);
}

.annotation-box .dot {
  width: 6px;
  height: 6px;
  background-color: #ffffff;
  border-radius: 50%;
  display: inline-block;
  box-shadow: 0 0 6px rgba(255, 255, 255, 0.8);
  flex-shrink: 0;
  /* Ajout de la transition sur le point */
  transition: background-color 0.2s ease-in-out, box-shadow 0.2s ease-in-out;
}

.annotation-box.right-side {
  justify-content: flex-end;
}

.annotation-box.left-side {
  justify-content: flex-start;
}

.annotation-box.right-side:hover,
.annotation-box.left-side:hover {
  border-bottom: 1px solid rgba(241, 196, 15, 0.6);
}

/* ==========================================================
   STYLE DE LA CARTE COMPOSANT (MODE FIXE AUX COINS DE L'ÉCRAN)
   ========================================================== */
.annotation-box.active-card {
  position: fixed !important;
  bottom: 200px !important;
  transform: none !important;
  background: rgba(20, 20, 20, 1);
  border: 1px solid rgba(241, 196, 15, 1);
  border-radius: 6px;
  padding: 14px;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 8px;
  width: 260px;
  white-space: normal;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
  cursor: default;
  z-index: 999;
}

.annotation-box.active-card.left-side-card {
  left: 40px !important;
}

.annotation-box.active-card.right-side-card {
  right: 40px !important;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
  border-bottom: 1px solid rgba(255, 255, 255, 0.15);
  padding-bottom: 6px;
}

.card-title {
  font-weight: bold;
  color: #f1c40f;
  font-size: 16px;
  letter-spacing: 1px;
}

.card-close-btn {
  background: none;
  border: none;
  color: rgba(255, 255, 255, 0.5);
  cursor: pointer;
  font-size: 16px;
  padding: 2px 6px;
  transition: color 0.3s ease-in-out;
}

.card-close-btn:hover {
  color: #ffffff;
}

.card-body {
  color: rgba(255, 255, 255, 0.8);
  font-family: sans-serif;
  font-size: 16px;
  line-height: 1.4;
  text-transform: none;
  letter-spacing: 0;
}

/* ==========================================================
   STYLE DE L'ÉCRAN DE CHARGEMENT
   ========================================================== */
.loader-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: #1a1a1a; 
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 9999;
}

.loader-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
  width: 500px;
}

.loader-title {
  color: #f1c40f;
  font-family: "Courier New", Courier, monospace;
  font-size: 28px;
  letter-spacing: 3px;
  text-transform: uppercase;
  margin: 0;
}

.progress-bar-container {
  width: 100%;
  height: 2px;
  background-color: rgba(255, 255, 255, 0.1);
  border-radius: 2px;
  overflow: hidden;
}

.progress-bar {
  height: 100%;
  background-color: #f1c40f;
  transition: width 0.1s linear; 
}

.progress-text {
  color: #ffffff;
  font-family: "Courier New", Courier, monospace;
  font-size: 12px;
  letter-spacing: 1px;
}

.fade-leave-active {
  transition: opacity 0.8s ease-in-out;
}
.fade-leave-to {
  opacity: 0;
}
</style>

<style scoped>
.canvas-container {
  width: 100%;
  height: 100vh;
  overflow: hidden;
  background-color: #1a1a1a;
  /* Supprimé le cursor: grab */
  position: relative;
}

.brand-overlay {
  position: absolute;
  top: 40px;
  left: 40px;
  z-index: 10;
  pointer-events: none; 
  font-family: "Courier New", Courier, monospace;
}

.brand-title {
  margin: 0;
  font-size: 85px;
  font-weight: 700;
  letter-spacing: 6px;
  text-transform: uppercase;
}

.brand-title .anim-letter {
  background: linear-gradient(135deg, #f39c12 0%, #f1c40f 50%, #f39c12 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  filter: drop-shadow(0 2px 10px rgba(241, 196, 15, 0.2));
}

.brand-description {
  margin: 8px 0 0 0;
  font-size: 16px;
  color: rgba(255, 255, 255, 0.5);
  text-transform: uppercase;
  letter-spacing: 2px;
}

.anim-letter {
  display: inline-block;
  opacity: 0;
  transform: translateY(15px);
  animation: letterFadeIn 0.6s cubic-bezier(0.2, 0.8, 0.2, 1) forwards;
  white-space: pre-wrap; 
}

@keyframes letterFadeIn {
  0% {
    opacity: 0;
    transform: translateY(15px);
  }
  100% {
    opacity: 1;
    transform: translateY(0);
  }
}
</style>