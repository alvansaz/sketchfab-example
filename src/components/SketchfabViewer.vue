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
import { onMounted, ref } from 'vue';
import Sketchfab from '@sketchfab/viewer-api';
import { modelData } from './modelData';
import { floorsArray } from './floorsData';

// Constants
const MODEL_UID = 'bdf40cfa5c4b466c9ccaf3869f64a3a8';
const TRANSITION_DELAY_MS = 50

const UNIT_STATUS_COLORS = {
  Available: [0.1059, 0.8078, 0.4314],
  Sold: [0.8078, 0.2745, 0.1059],
  Unavailable: [0.4784, 0.4784, 0.4784],
  Allocated: [0.1412, 0.3765, 0.902],
  Reserved: [0.3647, 0.302, 0.7569],
};

// State
const iFrameElement = ref(null);
const allUnits = ref([]);
const allUnitsName = ref([]);
const floorList = ref([]);
const nodesList = ref([]);
const roofList = ref([]);
const currentFloor = ref(38);
const selectModels = ref([]);
const apiInstance = ref(null);
const selectedFloor = ref('');
const isModelLoaded = ref(false);
const materialCache = new Map();

// Utility Functions
const getUnitColor = (status) => UNIT_STATUS_COLORS[status] || [0, 0, 0];

const processUnitsData = (data) => {
  return data.flatMap((parent) =>
    parent.units.map((unit) => ({
      ...parent,
      ...unit,
      units: undefined,
    })),
  );
};

const filterValidUnitNames = (units) => {
  return units
    .map(({ box_name, status }) => (status !== 'Conditional' ? box_name : null))
    .filter(Boolean);
};

// Model Initialization
const initializeModel = () => {
  iFrameElement.value = document.getElementById('api-frame');
  const client = new Sketchfab(iFrameElement.value);

  client.init(MODEL_UID, {
    success: handleModelSuccess,
    error: handleModelError,
    ...getViewerConfig(),
  });
};

const handleModelSuccess = (api) => {
  apiInstance.value = api;
  selectModels.value = [];

  api.load();
  api.start(() => {
    api.addEventListener('viewerready', () => {
      initializeModelNodes(api);
    });
  });
};

const handleModelError = (error) => {
  console.error('Model loading error:', error);
};

const assignMaterialToUnit = async (node, status, api) => {
  const cacheKey = `${node.name}_${status}`;

  if (materialCache.has(cacheKey)) {
    const material = materialCache.get(cacheKey);
    api.assignMaterial(node, material.id);
    selectModels.value.push(material);
    return;
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
  };

  return new Promise((resolve, reject) => {
    api.createMaterial(materialConfig, (err, material) => {
      if (!err) {
        materialCache.set(cacheKey, material);
        api.assignMaterial(node, material.id);
        selectModels.value.push(material);
        resolve(material);
      } else {
        reject(err);
      }
    });
  });
};

const processNodes = async (nodes, api) => {
  const floorNodes = [];
  const unitNodes = [];
  const roofNodes = [];
  const materialPromises = [];

  Object.values(nodes).forEach((node) => {
    const nodeName = node.name || '';

    if (nodeName.startsWith('floor_') && node.type === 'Group') {
      floorNodes.push(node);
    }

    if (nodeName.startsWith('Hover_') || nodeName.startsWith('Unit_')) {
      unitNodes.push(node);

      if (nodeName.startsWith('Hover_')) {
        const unitName = nodeName.split('_')[1];
        const unitIndex = allUnitsName.value.indexOf(unitName);

        if (unitIndex >= 0) {
          materialPromises.push(assignMaterialToUnit(node, allUnits.value[unitIndex].status, api));
        }
      }
    }

    if (nodeName.startsWith('Roof_')) {
      roofNodes.push(node);
    }
  });

  // Process materials in parallel
  try {
    await Promise.all(materialPromises);
  } catch (error) {
    console.error('Error assigning materials:', error);
  }

  floorList.value = floorNodes.sort();
  nodesList.value = unitNodes.sort();
  roofList.value = roofNodes.sort();
  isModelLoaded.value = true;
};
selectedFloor.value;
const centerCamera = ref({ x: null, y: null, z: null })
const targetCamera = ref({ x: null, y: null, z: null })

const initializeModelNodes = (api) => {
  api.getNodeMap((err, nodes) => {
    if (err) {
      console.error('Error getting node map:', err);
      return;
    }
    processNodes(nodes, api).catch((error) => {
      console.error('Error processing nodes:', error);
    });
  });
  api.getCameraLookAt(function (err, camera) {
    if (!err) {
      window.console.log('camera.position', camera.position) // [x, y, z]
      window.console.log('camera.target', camera.target) // [x, y, z]
      centerCamera.value.x = camera.position[0]
      centerCamera.value.y = camera.position[1]
      centerCamera.value.z = camera.position[2]
      targetCamera.value.x = camera.target[0]
      targetCamera.value.y = camera.target[1]
      targetCamera.value.z = camera.target[2]
    }
  })
  console.log('center', centerCamera.value)
  console.log('target', targetCamera.value)


  api.addEventListener('click', handleNodeClick);
};

// Floor Management
const handleFloorChange = () => {
  const targetFloor = parseInt(selectedFloor.value === 'reset' ? 38 : selectedFloor.value);
  const currentFloorNum = parseInt(currentFloor.value);

  if (currentFloorNum > targetFloor) {
    hideFloors(currentFloorNum, targetFloor);
  } else if (currentFloorNum < targetFloor) {
    showFloors(currentFloorNum, targetFloor);
  }

  currentFloor.value = targetFloor;
};

const hideFloors = (from, to) => {
  let delayCounter = 0
  for (let i = from; i >= to; i--) {
    setTimeout(() => {
      hideFloorAndUnits(i);
    }, delayCounter * TRANSITION_DELAY_MS);
    delayCounter++
  }
  setTimeout(() => {
    findFloorUnits(to);
  }, 2000)
};

const showFloors = (from, to) => {
  let delayCounter = 0
  for (let i = from; i <= to; i++) {
    setTimeout(() => {
      showFloorAndUnits(i);
    }, delayCounter * TRANSITION_DELAY_MS);
    delayCounter++
  }
  setTimeout(() => {
    findFloorUnits(to);
  }, 2000)
};

const findFloorUnits = (floorNumber) => {
  const expectedLength = floorNumber.toString().length === 2 ? 10 : 9;
  const units = nodesList.value.filter((node) => {
    const unitName = node?.name || '';
    return (
      (unitName.includes(`Unit_W${floorNumber}`) || unitName.includes(`Unit_E${floorNumber}`)) &&
      unitName.length === expectedLength
    );
  });

  for (let i = 0; i < units.length; i++) {
    const unit = units[i];
    apiInstance.value.show(unit.instanceID);
    if (unit.type === 'MatrixTransform') {
      const position = {
        x: unit.localMatrix[12],
        y: unit.localMatrix[13],
        z: unit.localMatrix[14],
      };
      apiInstance.value.translate(unit.instanceID, [position.x, position.y, position.z * 100], {
        relative: true,
        duration: 0,
      });
      setTimeout(() => {
        apiInstance.value.translate(unit.instanceID, [position.x, position.y, position.z], {
          relative: true,
          duration: 3,
        });
      }, 1000);
    }
  }

    // Get scene graph to calculate model bounds and floor position
  apiInstance.value.getSceneGraph((err) => {
    if (!err) {
            // Find the actual floor position by looking for floor nodes
      const floorExpectedLength = floorNumber.toString().length === 2 ? 8 : 7;
      const currentFloorNode = floorList.value.find((node) =>
        node.name.includes(`floor_${floorNumber}`) && node.name.length === floorExpectedLength
      );

      let floorZPosition = 0; // Default floor position
      let modelCenterX = 0;
      let modelCenterY = 0;

      console.log('Looking for floor:', `floor_${floorNumber}`, 'Expected length:', floorExpectedLength);
      console.log('Available floor nodes:', floorList.value.map(n => n.name));
      console.log('Found floor node:', currentFloorNode);

      if (currentFloorNode && currentFloorNode.localMatrix) {
        // Get the actual Z position of the cut floor
        floorZPosition = currentFloorNode.localMatrix[14];
        // For X and Y, use a more conservative approach to stay centered
        modelCenterX = 0; // Keep camera at model center X
        modelCenterY = 0; // Keep camera at model center Y
        console.log('Floor matrix positions - Z:', floorZPosition);
        console.log('Using model center X,Y: 0, 0 to stay above model');
      } else {
        console.log('No floor node found or no localMatrix, using defaults');
        // Try to get center from any visible unit for Z position
        const anyUnit = units.find(unit => unit.localMatrix);
        if (anyUnit) {
          floorZPosition = anyUnit.localMatrix[14];
          console.log('Using unit Z position as reference:', floorZPosition);
        }
        // Keep X,Y at center regardless
        modelCenterX = 0;
        modelCenterY = 0;
      }

      // Calculate camera height based on floor spacing
      // Use a more reliable approach for camera height
      let cameraHeight = 50; // Default height

      try {
        // Try to calculate based on floor numbers
        const floorNumbers = floorsArray.map(f => parseInt(f.name)).filter(n => !isNaN(n));
        if (floorNumbers.length > 0) {
          const maxFloor = Math.max(...floorNumbers);
          const minFloor = Math.min(...floorNumbers);
          const estimatedFloorHeight = 3.5; // Estimated height per floor
          const totalModelHeight = (maxFloor - minFloor) * estimatedFloorHeight;

          if (!isNaN(totalModelHeight) && totalModelHeight > 0) {
            cameraHeight = floorZPosition + totalModelHeight * 0.8;
          } else {
            // Fallback: use current floor number for height calculation
            cameraHeight = floorZPosition + (parseInt(floorNumber) * 2) + 30;
          }
        } else {
          // Fallback: use current floor number for height calculation
          cameraHeight = floorZPosition + (parseInt(floorNumber) * 2) + 30;
        }
      } catch (error) {
        console.warn('Error calculating camera height, using fallback:', error);
        cameraHeight = floorZPosition + (parseInt(floorNumber) * 2) + 30;
      }

      // Ensure camera height is valid
      if (isNaN(cameraHeight) || cameraHeight <= floorZPosition) {
        cameraHeight = floorZPosition + 50; // Safe fallback
      }

      // Position camera above model center, always targeting the model center
      apiInstance.value.setCameraLookAt(
        [
          modelCenterX,   // X: center of model
          modelCenterY,   // Y: center of model
          cameraHeight    // Z: above the model
        ],
        [
          modelCenterX,   // Target X: center of model
          modelCenterY,   // Target Y: center of model
          0               // Target Z: always center of model (0)
        ],
        4.3,
        function (err) {
          if (!err) {
            window.console.log('Camera moved to top view')
            window.console.log('Camera position:', [modelCenterX, modelCenterY, cameraHeight])
            window.console.log('Camera target (model center):', [modelCenterX, modelCenterY, 0])
            window.console.log('Floor Z position:', floorZPosition)
          }
        }
      );
    } else {
      console.error('Error getting scene graph:', err);
      // Fallback to simple positioning
      apiInstance.value.setCameraLookAt(
        [0, 0, 50], // Simple top view
        [0, 0, 0],  // Look at center
        4.3
      );
    }
  });
};

const hideFloorAndUnits = (floorNumber) => {
  const unitExpectedLength = floorNumber.toString().length === 2 ? 10 : 9;
  const hoverExpectedLength = floorNumber.toString().length === 2 ? 11 : 10;
  const floorExpectedLength = floorNumber.toString().length === 2 ? 8 : 7;
  const roofExpectedLength = floorNumber.toString().length === 2 ? 7 : 6;

  floorList.value.forEach((node) => {
    if (node.name.includes(`floor_${floorNumber}`) && node.name.length === floorExpectedLength) {
      apiInstance.value.hide(node.instanceID);
    }
  });

  roofList.value.forEach((node) => {
    if (node.name.includes(`Roof_${floorNumber}`) && node.name.length === roofExpectedLength) {
      apiInstance.value.hide(node.instanceID);
    }
  });

  const floorUnits = nodesList.value.filter(
    (node) =>
      ((node?.name?.includes(`Unit_W${floorNumber}`) ||
        node?.name?.includes(`Unit_E${floorNumber}`)) &&
        node.name.length === unitExpectedLength) ||
      ((node?.name?.includes(`Hover_W${floorNumber}`) ||
        node?.name?.includes(`Hover_E${floorNumber}`)) &&
        node.name.length === hoverExpectedLength),
  );

  floorUnits.forEach((node) => apiInstance.value.hide(node.instanceID));
};

const showFloorAndUnits = (floorNumber) => {
  // const unitExpectedLength = floorNumber.toString().length === 2 ? 10 : 9;
  const hoverExpectedLength = floorNumber.toString().length === 2 ? 11 : 10;
  const floorExpectedLength = floorNumber.toString().length === 2 ? 8 : 7;
  const roofExpectedLength = floorNumber.toString().length === 2 ? 7 : 6;

  // const floorUnits = nodesList.value.filter(
  //   (node) =>
  //     (node?.name?.includes(`Unit_W${floorNumber}`) ||
  //       node?.name?.includes(`Unit_E${floorNumber}`)) &&
  //     node.name.length === unitExpectedLength,
  // );

  // floorUnits.forEach((node) => apiInstance.value.show(node.instanceID));

  if (floorNumber === Number(selectedFloor.value)) {
    return;
  }

  const floorHovers = nodesList.value.filter(
    (node) =>
      (node?.name?.includes(`Hover_W${floorNumber}`) ||
        node?.name?.includes(`Hover_E${floorNumber}`)) &&
      node.name.length === hoverExpectedLength,
  );

  floorHovers.forEach((node) => apiInstance.value.show(node.instanceID));

  floorList.value.forEach((node) => {
    if (node.name.includes(`floor_${floorNumber}`) && node.name.length === floorExpectedLength) {
      apiInstance.value.show(node.instanceID);
    }
  });

  roofList.value.forEach((node) => {
    if (node.name.includes(`Roof_${floorNumber}`) && node.name.length === roofExpectedLength) {
      apiInstance.value.show(node.instanceID);
    }
  });
};





// Event Handlers
const handleNodeClick = (item) => {
  console.log('Node clicked:', item);
};

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
});

// Initialization
onMounted(() => {
  allUnits.value = processUnitsData(modelData);
  allUnitsName.value = filterValidUnitNames(allUnits.value);
  initializeModel();
});
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