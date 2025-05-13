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
    <div class="operations flex items-center absolute bottom-[5.5rem] md:bottom-10">
      <span class="ml-3">
        <iconAnimation
          :plus="false"
          icon="floorplans/Video.svg"
          customId="camera-center"
          text="Reset View"
        >
        </iconAnimation>
      </span>
      <span class="ml-3">
        <iconAnimation
          :plus="false"
          icon="floorplans/show-suites.svg"
          customId="toggle-highlights"
          :text="showHighlights ? 'Hide Suites' : 'Show Suites'"
        ></iconAnimation>
      </span>
      <span class="ml-3">
        <iconAnimation
          v-if="mode !== 'both'"
          :plus="false"
          icon="floorplans/floorplans.svg"
          :text="mode !== 'both' ? 'Floorplans' : 'Building'"
          @click="$emit('goToBothView')"
        ></iconAnimation>
        <iconAnimation
          v-else
          :plus="false"
          icon="floorplans/floorplans.svg"
          text="Building"
          @click="$emit('backToModel3d')"
        ></iconAnimation>
      </span>
    </div>
    <button id="renderUnits" class="d-none absolute"></button>
  </div>
</template>

<script setup>
import { onMounted, ref, defineEmits } from 'vue'
import Sketchfab from '@sketchfab/viewer-api'
import { modelData } from './modelData'

const emit = defineEmits(['clickedUnit'])

const UID = 'bdf40cfa5c4b466c9ccaf3869f64a3a8'
const iFrameElement = ref(null)
const allUnits = ref([])
const allUnitsName = ref([])
const showHighlights = ref(false)
const mode = ref('both')
const hoveredUnitDetail = ref({})
const showModal = ref(false)
const selectedImageForModal = ref({})
const pointersArray = ref([])
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
const cameraAnimation = (api) => {
  api.setCameraEasing('easeInOutQuad')

  api.setCameraLookAt(
    [155.1972980893007, -76.54364457820951, 68.81421152337492],
    [7.231634037788717, 12.354523892724492, -2.562719243730042],
    0.0001,
    function (err) {
      if (!err) {
        window.console.log('Camera moved')
      }
    },
  )

  api.setCameraLookAtEndAnimationCallback(function (err) {
    if (!err) {
      const cameraData = {
        fov: 45,
        nearFarRatio: 0.005,
        useCameraConstraints: true,
        usePanConstraints: true,
        useZoomConstraints: true,
        usePitchConstraints: true,
        useYawConstraints: false,
        zoomIn: 91.254882094374196,
        zoomOut: 337.6040204624173,
        left: -3.141593,
        right: 3.141593,
        up: 1.555089,
        down: 0,
      }
      api.setCameraConstraints(cameraData, () => {
        api.setEnableCameraConstraints(
          true,
          { preventCameraConstraintsFocus: false },
          function (err) {
            console.log('err', err)
          },
        )
      })
    }
  })
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
          cameraAnimation(api)

          function recenterCamera() {
            api.setCameraLookAt(
              [155.1972980893007, -76.54364457820951, 68.81421152337492],
              [7.231634037788717, 12.354523892724492, -2.562719243730042],
              4.3,
              function (err) {
                if (!err) {
                  console.log('Camera moved')
                }
                console.log(err)
              },
            )
          }
          // get nodes data
          api.getNodeMap((err, nodes) => {
            if (!err) {
              nodesArray = nodes
              setUnits(nodes)
            }
          })
          api.addEventListener('click', function (item) {
            if (item?.material?.name?.indexOf('Unit_') >= 0) {
              const untiName = item.material.name?.split('_')[1]
              const unitDetail = allUnits.value.find((x) => x.box_name === untiName) || ''
              if (unitDetail) {
                console.log('unitDetail', unitDetail)
                emit('clickedUnit', unitDetail.model_name)
              }
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
          function toggleHighlights() {
            if (showHighlights.value) {
              removeAllMaterials()
              showHighlights.value = false
            } else {
              setUnits(nodesArray)
              showHighlights.value = true
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
          document.getElementById('camera-center').addEventListener('click', recenterCamera)
          document.getElementById('renderUnits').addEventListener('click', resetUnits)
          document.getElementById('toggle-highlights').addEventListener('click', toggleHighlights)

          const boxModel = document.querySelector('.box-modal')
          const boxModalHeight = 500
          api.addEventListener(
            'nodeMouseEnter',
            function (item) {
              if (item?.material?.name?.startsWith('Unit_')) {
                const hoverdUntiName = item.material.name?.split('_')[1]
                hoveredUnitDetail.value =
                  allUnits.value.find((x) => x.box_name === hoverdUntiName) || ''
                const unitPosition = item.position2D

                const topPosition = unitPosition[1] - 200
                const bottomPosition = unitPosition[1] - boxModalHeight

                if (bottomPosition > window.innerHeight) {
                  boxModel.style.bottom = 300 + 'px'
                } else if (topPosition < 0) {
                  boxModel.style.top = '0px'
                } else {
                  boxModel.style.top = topPosition + 'px'
                }
                if (boxModel.style.top == '0px') {
                  boxModel.style.top = '17px'
                }

                if (mode.value === 'both') {
                  if (
                    unitPosition[0] + boxModel.getBoundingClientRect().width >=
                    window.innerWidth / 2 + 50
                  ) {
                    boxModel.style.left = window.innerWidth / 3 - 300 + unitPosition[0] + 'px'
                  } else {
                    boxModel.style.left = window.innerWidth / 3 + unitPosition[0] + 'px'
                  }
                } else {
                  boxModel.style.left = unitPosition[0] + 'px'
                }
              }
            }
          )

          api.addEventListener(
            'nodeMouseLeave',
            function () {
              hoveredUnitDetail.value = {}
              showModal.value = false
            },
          )

          api.addEventListener(
            'annotationSelect',
            function (index) {
              if (index >= 0) {
                const item = pointersArray.value.find((x) => x.identifier == index + 1)
                if (item) {
                  selectedImageForModal.value = {
                    url: item.img,
                    title: item.title,
                  }
                  this.openImageModal()
                }
              }
            }
          )
        })
      })
    },
    error: (e) => {
      console.error('خطا در بارگذاری مدل:', e)
    },
  })
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
