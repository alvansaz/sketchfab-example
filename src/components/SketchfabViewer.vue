<template>
  <div class="sketchfab-viewer">
    <iframe
      sandbox="allow-scripts allow-same-origin allow-popups allow-forms"
      id="api-frame"
      allow="autoplay; fullscreen; xr-spatial-tracking"
      allowfullscreen="allowfullscreen"
      mozallowfullscreen="true"
      webkitallowfullscreen="true"
      style="
        width: 100%;
        border-color: transparent;
        transition: height 500ms;
        height: 100%;
      "
    ></iframe>

    <select name="floors" id="floors" style="position: absolute;right: 50px;bottom: 50px;" @change="selectFloor()" v-model="selectedFloor">
      <option value="">select floor</option>
      <option v-for="floor in floorsArray" :key="floor.id" :value="floor.name">{{ floor.name }}</option>
    </select>
  </div>
</template>

<script setup>
import { onMounted, ref } from 'vue'
import Sketchfab from '@sketchfab/viewer-api'
import { modelData } from './modelData';
import { floorsArray } from './floorsData';

const UID = 'bdf40cfa5c4b466c9ccaf3869f64a3a8'
const iFrameElement = ref(null)
const allUnits = ref([])
const allUnitsName = ref([])
const floorList = ref([])
const selectModels = ref([])
const apiInstance = ref(null)
const selectedFloor = ref('')
const getAllUnitsWithParentData = (data) => {
  const result = []

  data.forEach((parent) => {
    parent.units.forEach((unit) => {
      const combinedUnit = {
        ...parent,
        ...unit,
      }
      delete combinedUnit.units
      result.push(combinedUnit)
    })
  })
  return result
}

const setData = (data) => {
  allUnits.value = getAllUnitsWithParentData(data)

  allUnitsName.value = allUnits.value
    .map(({ box_name, status }) => {
      if (status !== 'Conditional') {
        return box_name
      }
    })
    .filter(Boolean)
}

const renderWithData = (data) => {
  setData(data)
}

function getColorOfUnit(status) {
  let color = ''
  switch (status) {
    case 'Available':
      color = [0.1059, 0.8078, 0.4314]
      break
    case 'Sold':
      color = [0.8078, 0.2745, 0.1059]
      break
    case 'Unavailable':
      color = [0.4784, 0.4784, 0.4784]
      break
    case 'Allocated':
      color = [0.1412, 0.3765, 0.902]
      break
    case 'Reserved':
      color = [0.3647, 0.302, 0.7569]
      break
  }

  return color
}

function initModel() {
  iFrameElement.value = document.getElementById('api-frame')
  const client = new Sketchfab(iFrameElement.value)

  client.init(UID, {
    success: (api) => {
      apiInstance.value = api
      selectModels.value = []
      let nodesArray = [];
      api.load()
      api.start(() => {
        api.addEventListener('viewerready', () => {
          iFrameElement.value.classList.remove('hidden')

          api.getNodeMap((err, nodes) => {
            if (!err) {
              setUnits(nodes)
            }
          })

          api.addEventListener('click', function () {
            let currentFloor = 32;
            
            function hideNextFloor() {
              if (currentFloor <= 27) return;
              
              api.hide(floorList.value[currentFloor].instanceID, function(err) {
                if (err) {
                  console.error('Error hiding node:', err);
                } else {
                  const floorAvailableNodes = nodesArray.filter(node => node?.name?.includes('W' + Number(currentFloor + 5)));
                  for(let j = 0; j < floorAvailableNodes.length; j++) {
                    api.hide(floorAvailableNodes[j].instanceID, function(err) {
                      if (err) {
                        console.error('Error showing node:', err);
                      }
                    });
                  }

                  const roofsArray = nodesArray.filter(node => node?.name?.includes("Roof_0" + Number(currentFloor + 5)));
                  for(let j = 0; j < roofsArray.length; j++) {
                    api.hide(roofsArray[j].instanceID, function(err) {
                      if (err) {
                        console.error('Error showing node:', err);
                      }
                    });
                  }
                  
                  currentFloor--;
                  setTimeout(hideNextFloor, 500); // Wait 500ms before hiding next floor
                }
              });
            }
            
            hideNextFloor();
          })

          function setUnits(nodes) {
            nodesArray = [];
            for (const index in nodes) {
              const nodeIsHover = nodes[index].name?.startsWith('Hover_');
              const nodeIsUnit = nodes[index].name?.startsWith('Unit_');
              const nodeIsFloor = nodes[index].name?.startsWith('floor_');
              const nodeIsRoof = nodes[index].name?.startsWith('Roof_');
              const nodeName = nodes[index].name?.split('_')

              if(nodeIsRoof) {
                nodesArray.push(nodes[index]);
              }
              
              if (nodeIsFloor && nodes[index].type === "Group") {
                floorList.value.push(nodes[index])
              }

              floorList.value = floorList.value.sort();
              console.log(floorList.value)
              
              if (nodeIsHover && allUnitsName.value?.indexOf(nodeName[1]) >= 0) {
                assignMaterialToUnits(
                  nodes[index],
                  allUnits.value[allUnitsName?.value.indexOf(nodeName[1])].status,
                )
              }

              if (nodeIsUnit || nodeIsHover) {
                nodesArray.push(nodes[index]);
              }
            }
          }

          function assignMaterialToUnits(node, status) {
            api.createMaterial(
              {
                channels: {
                  AlbedoPBR: {
                    color: getColorOfUnit(status),
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
                    color: getColorOfUnit(status),
                    enable: true,
                    factor: 0.3819108768217719,
                  },
                },
                name: node.name,
              },
              function (err, material) {
                if (!err) {
                  api.assignMaterial(node, material.id)
                  selectModels.value.push(material)
                }
              },
            )
          }

        })
      })
    },
    error: (e) => {
      console.error('خطا در بارگذاری مدل:', e)
    },
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
}

const selectFloor = () => {
  console.log(selectedFloor.value)
}

onMounted(() => {
  renderWithData(modelData)
  initModel()
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
</style>
