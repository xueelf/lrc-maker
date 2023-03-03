<script setup lang="ts">
import { ElMessage } from 'element-plus';
import { ref, reactive, computed } from 'vue';
import { invoke } from '@tauri-apps/api/tauri';

interface makerLyricProps {
}

const emit = defineEmits([
  'make',
]);
const props = defineProps<makerLyricProps>();
const state = reactive<{
  lyric: string;
}>({
  lyric: '',
});
</script>

<template>
  <div class="maker-lyric-container"></div>
</template>

<style scoped lang="scss">
.maker-lyric-container {}
</style>
