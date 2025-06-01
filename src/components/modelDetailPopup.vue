<template>
  <Transition>
    <div class="box-modal w-254 hide-scrollbar h-[146px]" v-show="hoveredUnitDetail.status">
      <div class="status-tag text-white">
        <span class="status-color">
          <span
            :style="{
              backgroundColor: hoveredUnitDetail.color,
            }"
          ></span>
        </span>
        {{ hoveredUnitDetail.status }}
      </div>
      <div class="box-modal-header border-b py-2 px-3 flex items-center">
        <h4 class="text-14 text-white mr-2">{{ hoveredUnitDetail.name }}</h4>
        <p class="text-14 text-white">
          <span class="mr-1">|</span> {{ hoveredUnitDetail.model_name }}
        </p>
      </div>

      <div class="box-modal-body pt-1 px-3">
        <ul
          class="flex flex-wrap items-between"
          @mouseenter="$emit('mouseoverModal')"
          @mouseover="$emit('mouseoverModal')"
        >
          <li class="w-1-2 mb-2">
            <span class="title text-12 text-white opacity-40 block">Type</span>
            <span class="text-14 text-white">{{ hoveredUnitDetail.suite_style || '-' }}</span>
          </li>
          <li class="w-1-2 mb-2">
            <span class="title text-12 text-white opacity-40 block">Area</span>
            <span class="text-14 text-white">{{
              hoveredUnitDetail.area ? hoveredUnitDetail.area + ' sq.ft.' : '-'
            }}</span>
          </li>
          <li class="w-1-2">
            <span class="title text-12 text-white opacity-40 block">Price</span>
            <span class="text-14 text-white">{{
              hoveredUnitDetail.price ? '$' + hoveredUnitDetail.price.toLocaleString('en-US') : '-'
            }}</span>
          </li>
          <li class="w-1-2">
            <span class="title text-12 text-white opacity-40 block">Orientation</span>
            <span class="text-14 text-white">{{ hoveredUnitDetail.exposure || '-' }}</span>
          </li>
        </ul>
      </div>
    </div>
  </Transition>
</template>

<script setup>
import { defineProps } from 'vue';

defineProps({
  hoveredUnitDetail: {
    type: [Object, String],
    required: true,
  },
});
</script>

<style lang="scss">
.box-modal {
  background: rgba(0, 0, 0, 0.7);
  backdrop-filter: blur(16px);
  position: fixed;
  width: 254px;

  .status-tag {
    padding: 4px 12px;
    border-radius: 8px;
    color: #fff;
    font-size: 12px;
    display: flex;
    align-items: center;
    width: max-content;
    background-color: #636363;
    margin-left: 5px;
    margin-top: -20px;
    position: absolute;

    .status-color {
      display: inline-block;
      border-radius: 50%;
      width: 10px;
      height: 10px;
      margin-right: 7px;

      span {
        border-radius: 50%;
        display: inline-block;
        width: 10px;
        height: 10px;
        border: 2px solid #fff;
        float: left;
      }
    }
  }

  &.h-500 {
    height: 500px;
  }

  .bg-white {
    img {
      max-height: 150px;
    }
  }

  .box-modal-header {
    border-color: rgba(255, 255, 255, 0.2);
  }

  .py-2 {
    padding: 10px;
  }

  li {
    list-style: none;
  }

  .text-white {
    color: #fff;
  }

  .opacity-40 {
    opacity: 0.4;
  }

  .block {
    display: block;
  }

  .text-12 {
    font-size: 12px;
  }

  .text-14 {
    font-size: 14px;
  }

  .hide-scrollbar {
    scrollbar-width: none;
    -ms-overflow-style: none;
  }
  
  .flex {
    display: flex;
  }

  .flex-wrap {
    flex-wrap: wrap;
  }

  .items-between {
    justify-content: space-between;
  }

  .mb-2 {
    margin-bottom: 8px;
  }

  .w-1-2 {
    width: 50%;
  }

  .items-center {
    align-items: center;
  }

  .px-3 {
    padding: 10px 12px;
  }
}
</style>
