<template>
  <main class="main-3d relative h-full">
    <span class="ml-3 absolute mt-4" v-if="mode === 'both'">
      <iconAnimation
        :plus="false"
        icon="floorplans/hide-model.svg"
        v-if="pageShowStatus !== 1"
        text="Hide Building"
        @click="$router.push({ name: 'floorplans', query: { full_3d: false } })"
      ></iconAnimation>
    </span>

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
      @load="iframeLoaded"
    ></iframe>
    <Skeleton v-if="!isIframeLoaded" class="bg-qooWhite-3 w-full h-full overflow-hidden">
    </Skeleton>

    <modelDetailPopup
      :hoveredUnitDetail="hoveredUnitDetail"
      @mouseoverModal="mouseoverModal"
      v-show="hoveredUnitDetail && Object.keys(hoveredUnitDetail).length != 0"
    />

    <div
      class="operations flex items-center absolute bottom-[5.5rem] md:bottom-10"
    >
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
  </main>

  <popup-modal
    w="624px"
    classList="p-5 sm:p-6 md:py-9 md:px-6"
    :title="selectedImageForModal.title"
    @close="closeImageModal($event)"
    v-show="isShowImageModal"
  >
    <div class="w-full relative">
      <img
        :src="selectedImageForModal.url"
        alt="image"
        class="max-h-96 mx-auto"
      />
      <div class="mt-3">
        <FullScreen
          @click="showFullScreen = true"
          fill="black"
          class="cursor-pointer"
        ></FullScreen>
      </div>
    </div>
    <div
      v-if="showFullScreen"
      class="fixed top-0 left-0 w-full z-[250] h-[100vh]"
    >
      <div class="w-full flex h-[100vh] bg-white justify-center items-center">
        <div class="w-full h-fit relative">
          <img :src="selectedImageForModal.url" alt="image" class="mx-auto" />
          <button
            @click="showFullScreen = false"
            class="fixed drop-shadow-[0_4px_3px_rgba(0,0,0,0.25)] top-3 right-3 cursor-pointer text-xl"
          >
            <svg
              width="24"
              height="24"
              viewBox="0 0 24 24"
              fill="none"
              xmlns="http://www.w3.org/2000/svg"
            >
              <path
                d="M18 6L6 18"
                stroke="#000"
                stroke-opacity="0.76"
                stroke-width="1.5"
                stroke-linecap="round"
              />
              <path
                d="M18 18L6 6"
                stroke="#000"
                stroke-opacity="0.76"
                stroke-width="1.5"
                stroke-linecap="round"
              />
            </svg>
          </button>
        </div>
      </div>
    </div>
  </popup-modal>
</template>

<script>
import { useModels } from "@/store/models.js";
import Sketchfab from "@sketchfab/viewer-api";
import Skeleton from "@/components/base/skeleton/Skeleton.vue";
import PopupModal from "@/components/base/PopupModalAnimation.vue";
import FullScreen from "@/components/svg/FullScreen.vue";
import modelDetailPopup from "./modelDetailPopup.vue";
import iconAnimation from "@/components/base/common/iconAnimation.vue";
import { useVerticalMenu } from "@/store/verticalMenu";
import { useWorksheets } from "@/store/worksheet.js";
import { useQuery } from "@tanstack/vue-query";
import { computed, onMounted, watchEffect, getCurrentInstance } from "vue";

import { pointers, pointersQuery } from "@/services/axios/pointers.service.js";
// const verticalMenuStore = useVerticalMenu();

export default {
  name: "sketchfabPage",
  props: {
    pageShowStatus: Number,
    model: Object,
    mode: String,
  },
  components: {
    modelDetailPopup,
    iconAnimation,
    PopupModal,
    FullScreen,
    Skeleton,
  },
  emits: ["goToBothView", "backToModel3d", "clickedUnit"],
  setup(props) {
    const instance = getCurrentInstance();
    const worksheetsStore = useWorksheets();

    const { data: modelsQuery, isLoading: getModelsLoading } = useQuery({
      queryKey: ["pointers"],
      queryFn: pointersQuery,
      select: (data) => data.data,
      enabled: true,
    });

    // worksheetsStore.isLoading = computed(() => getModelsLoading.value && props.mode == '3d');

    watchEffect(() => {
      if (modelsQuery.value) {
        setTimeout(() => {
          instance.proxy.onMountedQuery();
        }, 200);
      }
    });

    return {
      modelsQuery,
    };
  },
  data() {
    return {
      uid: "08345003736249b0a1c48d28bab802ba",
      allData: [],
      verticalMenuStore: useVerticalMenu(),
      allUnits: [],
      isIframeLoaded: false,
      allUnitsName: [],
      modelsStore: {},
      hoveredUnitDetail: "",
      showHighlights: true,
      showModal: true,
      showFullScreen: false,
      pointersArray: [],
      isShowImageModal: false,
      selectedImageForModal: {
        url: "",
        title: "",
      },
      popupPosition: { top: 0, left: 0 }, // Add this line
    };
  },
  async mounted() {
    this.modelsStore = useModels();
    this.initModel();
  },
  methods: {
    iframeLoaded(e) {
      this.isIframeLoaded = true;
    },
    async onMountedQuery() {
      this.getPointersList();

      if (Object.keys(this.model).length !== 0) {
        this.renderWithData([this.model]);
      }
    },
    initModel() {
      const iframe = document.getElementById("api-frame");
      const client = new Sketchfab(this.version, iframe);
      client.init(this.uid, {
        success: this.modelSuccess,
        error: function onError() {
          console.error("Viewer error");
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
        max_texture_size: 1024,
      });
    },
    mouseoverModal() {
      this.hoveredUnitDetail = {};
    },
    modelSuccess(api) {
      api.load();
      api.start();

      let selectModels = [];
      let nodesArray = [];
      const $this = this;

      function removeAllMaterials() {
        selectModels.forEach((item) => {
          if (item.name.indexOf("Unit_") >= 0) {
            item.channels.Opacity = {
              enable: true,
              factor: 0.0,
            };
            api.setMaterial(item);
          }
        });
        selectModels = [];
      }

      function getColorOfUnit(status) {
        let color = "";
        switch (status) {
          case "Available":
            color = [0.1059, 0.8078, 0.4314];
            break;
          case "Sold":
            color = [0.8078, 0.2745, 0.1059];
            break;
          case "Unavailable":
            color = [0.4784, 0.4784, 0.4784];
            break;
          case "Allocated":
            color = [0.1412, 0.3765, 0.902];
            break;
          case "Reserved":
            color = [0.3647, 0.302, 0.7569];
            break;
        }

        return color;
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
              api.assignMaterial(node, material.id);
              selectModels.push(material);
            }
          },
        );
      }

      function setUnits(nodes) {
        for (const index in nodes) {
          const nodeIsUnit = nodes[index].name.indexOf("Unit_");
          const nodeName = nodes[index].name.split("_");
          if (nodeIsUnit >= 0 && $this.allUnitsName.indexOf(nodeName[1]) >= 0) {
            assignMaterialToUnits(
              nodes[index],
              $this.allUnits[$this.allUnitsName.indexOf(nodeName[1])].status,
            );
          }
        }
      }

      function resetUnits() {
        removeAllMaterials();
        setUnits(nodesArray);
      }

      function toggleHighlights() {
        if ($this.showHighlights) {
          removeAllMaterials();
          $this.showHighlights = false;
        } else {
          setUnits(nodesArray);
          $this.showHighlights = true;
        }
      }

      api.addEventListener("viewerready", () => {
        this.cameraAnimation(api);

        function recenterCamera() {
          api.setCameraLookAt(
            [155.1972980893007, -76.54364457820951, 68.81421152337492],
            [7.231634037788717, 12.354523892724492, -2.562719243730042],
            4.3,
            function (err) {
              if (!err) {
                window.console.log("Camera moved");
              }
            },
          );
        }

        document
          .getElementById("camera-center")
          .addEventListener("click", recenterCamera);
        document
          .getElementById("renderUnits")
          .addEventListener("click", resetUnits);
        document
          .getElementById("toggle-highlights")
          .addEventListener("click", toggleHighlights);

        api.getNodeMap(function (err, nodes) {
          if (err) {
            return;
          }

          nodesArray = nodes;
          setUnits(nodes);
        });

        api.addEventListener(
          "click",
          function (item) {
            if (item?.material?.name.indexOf("Unit_") >= 0) {
              const untiName = item.material.name.split("_")[1];
              const unitDetail =
                this.allUnits.find((x) => x.box_name === untiName) || "";

              if (unitDetail) {
                this.$emit("clickedUnit", unitDetail.model_name);
              }
            }
          }.bind(this),
        );

        const boxModel = document.querySelector(".box-modal");
        const boxModalHeight = 500;
        api.addEventListener(
          "nodeMouseEnter",
          function (item) {
            if (item?.material?.name.indexOf("Unit_") >= 0) {
              const hoverdUntiName = item.material.name.split("_")[1];
              this.hoveredUnitDetail =
                this.allUnits.find((x) => x.box_name === hoverdUntiName) || "";
              const unitPosition = item.position2D;

              const topPosition = unitPosition[1] - 200;
              const bottomPosition = unitPosition[1] - boxModalHeight;

              if (bottomPosition > window.innerHeight) {
                boxModel.style.bottom = 300 + "px";
              } else if (topPosition < 0) {
                boxModel.style.top = "0px";
              } else {
                boxModel.style.top = topPosition + "px";
              }
              if (boxModel.style.top == "0px") {
                boxModel.style.top = "17px";
              }

              if (this.mode === "both") {
                if (
                  unitPosition[0] + boxModel.getBoundingClientRect().width >=
                  window.innerWidth / 2 + 50
                ) {
                  boxModel.style.left =
                    window.innerWidth / 3 - 300 + unitPosition[0] + "px";
                } else {
                  boxModel.style.left =
                    window.innerWidth / 3 + unitPosition[0] + "px";
                }
              } else {
                boxModel.style.left = unitPosition[0] + "px";
              }
            }
          }.bind(this),
        );

        api.addEventListener(
          "nodeMouseLeave",
          function (item) {
            this.hoveredUnitDetail = {};
            this.showModal = false;
          }.bind(this),
        );

        api.addEventListener(
          "annotationSelect",
          function (index) {
            if (index >= 0) {
              const item = this.pointersArray.find(
                (x) => x.identifier == index + 1,
              );
              if (item) {
                this.selectedImageForModal = {
                  url: item.img,
                  title: item.title,
                };
                this.openImageModal();
              }
            }
          }.bind(this),
        );
      });
    },

    getAllUnitsWithParentData(data) {
      const result = [];

      data.forEach((parent) => {
        parent.units.forEach((unit) => {
          const combinedUnit = {
            ...parent,
            ...unit,
          };
          delete combinedUnit.units;
          result.push(combinedUnit);
        });
      });
      return result;
    },

    setData(data) {
      this.allUnits = this.getAllUnitsWithParentData(data);
      // eslint-disable-next-line camelcase, array-callback-return
      this.allUnitsName = this.allUnits.map(({ box_name, status }) => {
        if (status !== "Conditional") {
          // eslint-disable-next-line camelcase
          return box_name;
        }
      });
    },

    renderWithData(data) {
      this.setData(data);
      document.getElementById("renderUnits").click();
    },

    cameraAnimation(api) {
      api.setCameraEasing("easeInOutQuad");

      api.setCameraLookAt(
        [155.1972980893007, -76.54364457820951, 68.81421152337492],
        [7.231634037788717, 12.354523892724492, -2.562719243730042],
        0.0001,
        function (err) {
          if (!err) {
            window.console.log("Camera moved");
          }
        },
      );

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
          };
          api.setCameraConstraints(cameraData, () => {
            api.setEnableCameraConstraints(
              true,
              { preventCameraConstraintsFocus: false },
              function (err) {
                if (!err) {
                }
              },
            );
          });
        }
      });
    },

    closeImageModal() {
      this.isShowImageModal = false;
    },

    openImageModal() {
      this.isShowImageModal = true;
    },

    pointers,
    async getPointersList() {
      const res = this.modelsQuery;

      if (!res.data.length) {
        return;
      }

      this.pointersArray = res.data;
    },

    // Add this method
    updatePopupPosition(event) {
      this.popupPosition = {
        top: event.clientY,
        left: event.clientX + 10,
      };
    },
  },

  computed: {
    modelsList: {
      get() {
        return this.modelsStore.getModelsFilteredList;
      },
    },
  },

  watch: {
    async modelsList(newVal) {
      if (newVal && (this.mode === "3d" || this.mode === "both")) {
        this.allData = newVal;
        this.renderWithData(this.allData);
      }
    },
    async model(newVal) {
      this.renderWithData([newVal]);
    },
    async mode(newVal) {
      if (newVal === "3d") {
        this.allData = this.modelsStore.getModelsFilteredList;
        this.renderWithData(this.allData);
      }
      this.hoveredUnitDetail = "";
    },
  },
};
</script>

<style lang="scss">
.main-3d {
  .bottom-status {
    .statuses {
      box-shadow: 0px 4px 8px 0px rgba(0, 0, 0, 0.04);
    }
  }
}

@media screen and (max-width: 1024px) {
  .main-3d {
    iframe {
      height: 100% !important;
    }
  }
}
</style>
