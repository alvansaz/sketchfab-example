<template>
  <div class="sketchfab-viewer">
    <iframe
      class="hidden"
      src=""
      id="api-frame"
      allow="autoplay; fullscreen; xr-spatial-tracking"
      xr-spatial-tracking
      execution-while-out-of-viewport
      execution-while-not-rendered
      web-share
      allowfullscreen
      mozallowfullscreen="true"
      webkitallowfullscreen="true"
      width="1500px"
      height="700px"
    ></iframe>
    <button id="renderUnits" class="d-none absolute"></button>
  </div>
</template>

<script setup>
import { onMounted, ref } from 'vue'
import Sketchfab from '@sketchfab/viewer-api'
import { modelData } from './modelData'

const UID = 'bdf40cfa5c4b466c9ccaf3869f64a3a8'
const iFrameElement = ref(null)
const allUnits = ref([])
const allUnitsName = ref([])

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
  document.getElementById('renderUnits').click()
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
      let selectModels = []
      let nodesArray = []
      api.load()
      api.start(() => {
        api.addEventListener('viewerready', () => {
          iFrameElement.value.classList.remove('hidden')

          // get nodes data
          api.getNodeMap((err, nodes) => {
            if (!err) {
              nodesArray = nodes
              setUnits(nodes)
            }
          })

          function setUnits(nodes) {

            for (const index in nodes) {
              const nodeIsUnit = nodes[index].name?.startsWith('Hover_')

              const nodeName = nodes[index].name?.split('_')
              if (nodeIsUnit && allUnitsName.value?.indexOf(nodeName[1]) >= 0) {
                assignMaterialToUnits(
                  nodes[index],
                  allUnits.value[allUnitsName?.value.indexOf(nodeName[1])].status,
                )
              }
            }
          }
          function removeAllMaterials() {
            selectModels.forEach((item) => {
              if (item.name.indexOf('Unit_') >= 0) {
                item.channels.Opacity = {
                  enable: true,
                  factor: 0.0,
                }
                api.setMaterial(item)
              }
            })
            selectModels = []
          }
          function resetUnits() {
            removeAllMaterials()
            setUnits(nodesArray)
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
                  selectModels.push(material)
                }
              },
            )
          }
          document.getElementById('renderUnits').addEventListener('click', resetUnits)
        })
      })
    },
    error: (e) => {
      console.error('خطا در بارگذاری مدل:', e)
    },
  })
}

onMounted(() => {
  initModel()
  renderWithData(modelData)
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
