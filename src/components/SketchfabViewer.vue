<template>
  <div class="sketchfab-viewer">
    <iframe
      sandbox="allow-scripts allow-same-origin allow-popups allow-forms"
      id="api-frame"
      allow="autoplay; fullscreen; xr-spatial-tracking"
      allowfullscreen="allowfullscreen"
      mozallowfullscreen="true"
      webkitallowfullscreen="true"
      :class="{ hidden: !isModelLoaded }"
      style="width: 100%; border-color: transparent; transition: height 500ms; height: 100%"
    ></iframe>

    <div
      v-if="!isModelLoaded"
      class="loading-container"
      style="width: 100%; height: 100%; display: flex; justify-content: center; align-items: center"
    >
      <div class="loading-text" style="font-size: 20px; font-weight: bold">Loading...</div>
    </div>

    <select
      v-if="isModelLoaded"
      name="floors"
      id="floors"
      class="floor-selector"
      v-model="selectedFloor"
      @change="handleFloorChange"
    >
      <option value="">select floor</option>
      <option v-for="floor in floorsArray" :key="floor.id" :value="floor.name">
        {{ floor.name }}
      </option>
    </select>
  </div>
</template>

<script setup>
import { onMounted, ref } from 'vue'
import Sketchfab from '@sketchfab/viewer-api'
import { modelData } from './modelData'
import { floorsArray } from './floorsData'

// Constants
const MODEL_UID = 'bdf40cfa5c4b466c9ccaf3869f64a3a8'
const TRANSITION_DELAY_MS = 50 // Delay in milliseconds for each floor transition
const UNIT_STATUS_COLORS = {
  Available: [0.1059, 0.8078, 0.4314],
  Sold: [0.8078, 0.2745, 0.1059],
  Unavailable: [0.4784, 0.4784, 0.4784],
  Allocated: [0.1412, 0.3765, 0.902],
  Reserved: [0.3647, 0.302, 0.7569],
}

// State
let iFrameElement = null
let allUnits = []
let allUnitsName = []
let floorList = []
let nodesList = []
let roofList = []
const currentFloor = ref(38)
let selectModels = []
let apiInstance = null
const selectedFloor = ref('')
const isModelLoaded = ref(false)
const materialCache = new Map()

// Utility Functions
const getUnitColor = (status) => UNIT_STATUS_COLORS[status] || [0, 0, 0]

const processUnitsData = (data) => {
  return data.flatMap((parent) =>
    parent.units.map((unit) => ({
      ...parent,
      ...unit,
      units: undefined,
    })),
  )
}

const filterValidUnitNames = (units) => {
  return units
    .map(({ box_name, status }) => (status !== 'Conditional' ? box_name : null))
    .filter(Boolean)
}

// Model Initialization
const initializeModel = () => {
  iFrameElement = document.getElementById('api-frame')
  const client = new Sketchfab(iFrameElement)

  client.init(MODEL_UID, {
    success: handleModelSuccess,
    error: handleModelError,
    ...getViewerConfig(),
  })
}

const handleModelSuccess = (api) => {
  apiInstance = api
  selectModels = []

  api.load()
  api.start(() => {
    api.addEventListener('viewerready', () => {
      initializeModelNodes(api)
    })
  })
}

const handleModelError = (error) => {
  console.error('Model loading error:', error)
}

const assignMaterialToUnit = async (node, status, api) => {
  const cacheKey = `${node.name}_${status}`

  if (materialCache.has(cacheKey)) {
    const material = materialCache.get(cacheKey)
    api.assignMaterial(node, material.id)
    selectModels.push(material)
    return
  }

  const materialConfig = {
    channels: {
      AlbedoPBR: {
        color: getUnitColor(status),
        enable: true,
        factor: 0.5914868455453611,
      },
      Opacity: {
        enable: true,
        factor: 0.3,
        invert: false,
        ior: 3.5,
      },
      EmitColor: {
        color: getUnitColor(status),
        enable: true,
        factor: 0.3819108768217719,
      },
    },
    name: node.name,
  }

  return new Promise((resolve, reject) => {
    api.createMaterial(materialConfig, (err, material) => {
      if (!err) {
        materialCache.set(cacheKey, material)
        api.assignMaterial(node, material.id)
        selectModels.push(material)
        resolve(material)
      } else {
        reject(err)
      }
    })
  })
}

const processNodes = async (nodes, api) => {
  const floorNodes = []
  const unitNodes = []
  const roofNodes = []
  const materialPromises = []

  Object.values(nodes).forEach((node) => {
    const nodeName = node.name || ''

    if (nodeName.startsWith('floor_') && node.type === 'Group') {
      floorNodes.push(node)
    }

    if (nodeName.startsWith('Hover_') || nodeName.startsWith('Unit_')) {
      unitNodes.push(node)

      if (nodeName.startsWith('Hover_')) {
        const unitName = nodeName.split('_')[1]
        const unitIndex = allUnitsName.indexOf(unitName)

        if (unitIndex >= 0) {
          materialPromises.push(assignMaterialToUnit(node, allUnits[unitIndex].status, api))
        }
      }
    }

    if (nodeName.startsWith('Roof_')) {
      roofNodes.push(node)
    }
  })

  // Process materials in parallel
  try {
    await Promise.all(materialPromises)
  } catch (error) {
    console.error('Error assigning materials:', error)
  }

  floorList = floorNodes.sort()
  nodesList = unitNodes.sort()
  roofList = roofNodes.sort()
  isModelLoaded.value = true
}

const initializeModelNodes = (api) => {
  api.getNodeMap((err, nodes) => {
    if (err) {
      console.error('Error getting node map:', err)
      return
    }
    processNodes(nodes, api).catch((error) => {
      console.error('Error processing nodes:', error)
    })
  })

  api.addEventListener('click', handleNodeClick)
}

// Floor Management
const handleFloorChange = () => {
  const targetFloor = parseInt(selectedFloor.value === 'reset' ? 38 : selectedFloor.value)
  const currentFloorNum = parseInt(currentFloor.value)

  if (currentFloorNum > targetFloor) {
    hideFloors(currentFloorNum, targetFloor)
  } else if (currentFloorNum < targetFloor) {
    showFloors(currentFloorNum, targetFloor)
  }

  currentFloor.value = targetFloor
}

const hideFloors = (from, to) => {
  let delayCounter = 0
  for (let i = from; i >= to; i--) {
    setTimeout(() => {
      hideFloorAndUnits(i)
    }, delayCounter * TRANSITION_DELAY_MS)
    delayCounter++
  }
}

const showFloors = (from, to) => {
  let delayCounter = 0
  for (let i = from; i <= to; i++) {
    setTimeout(() => {
      showFloorAndUnits(i)
    }, delayCounter * TRANSITION_DELAY_MS)
    delayCounter++
  }
}

const hideFloorAndUnits = (floorNumber) => {
  floorList.forEach((node) => {
    if (node.name.includes(`floor_${floorNumber}`)) {
      apiInstance.hide(node.instanceID)
    }
  })

  roofList.forEach((node) => {
    if (node.name.includes(`Roof_${floorNumber}`)) {
      apiInstance.hide(node.instanceID)
    }
  })

  const floorUnits = nodesList.filter((node) => node?.name?.includes(`W${floorNumber}`))

  floorUnits.forEach((node) => apiInstance.hide(node.instanceID))
}

const showFloorAndUnits = (floorNumber) => {
  floorList.forEach((node) => {
    if (node.name.includes(`floor_${floorNumber}`)) {
      apiInstance.show(node.instanceID)
    }
  })

  roofList.forEach((node) => {
    if (node.name.includes(`Roof_${floorNumber}`)) {
      apiInstance.show(node.instanceID)
    }
  })

  const floorUnits = nodesList.filter((node) => node?.name?.includes(`W${floorNumber}`))

  floorUnits.forEach((node) => apiInstance.show(node.instanceID))
}

// Event Handlers
const handleNodeClick = (item) => {
  console.log('Node clicked:', item)
}

// Viewer Configuration
const getViewerConfig = () => ({
  annotation: 0,
  annotations_visible: 1,
  autospin: 0,
  autostart: 1,
  cardboard: 0,
  camera: 0,
  preload: 0,
  ui_stop: 0,
  transparent: 0,
  ui_animations: 0,
  ui_annotations: 1,
  ui_controls: 0,
  ui_fullscreen: 0,
  ui_general_controls: 1,
  ui_help: 1,
  ui_hint: 1,
  ui_infos: 0,
  ui_inspector: 0,
  ui_settings: 0,
  ui_vr: 0,
  ui_watermark_link: 0,
  ui_watermark: 0,
  max_texture_size: 32,
})

// Initialization
onMounted(() => {
  allUnits = processUnitsData(modelData)
  allUnitsName = filterValidUnitNames(allUnits)
  initializeModel()
})
</script>

<style scoped>
.sketchfab-viewer {
  width: 100%;
  height: 100%;
  overflow: hidden;
}

.sketchfab-viewer iframe {
  width: 100%;
  height: 100%;
  border: none;
}

.floor-selector {
  position: absolute;
  right: 50px;
  bottom: 50px;
}

.hidden {
  display: none;
}
</style>
