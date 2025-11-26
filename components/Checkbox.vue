<style scoped>
#filter_button {
  display: block;
  width: 100%;
  text-align: left;
  margin: 4px 0;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;

  /* 높이 조정 */
  padding: 8px 10px; /* 위아래 8px, 좌우 10px */
  line-height: 1.4; /* 줄 높이 */
  font-size: 14px; /* 글자 크기 조정 가능 */
  box-sizing: border-box; /* padding 포함해서 크기 계산 */

  /* 선택 시 구분 잘 되게 */
  border: 1px solid #ccc;
  border-radius: 4px;
  background: #f9f9f9;
}
</style>

<template>
  <button id="filter_button" @click="toggle">
    <input type="checkbox" :checked="checked" @click.stop />
    {{ label }}
  </button>
</template>

<script setup>
import { defineProps, defineEmits } from 'vue';

const props = defineProps({
  label: String,
  checked: Boolean,
});

const emit = defineEmits(['update:checked']);

function toggle() {
  emit('update:checked', !props.checked);
}
</script>
