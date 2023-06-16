<script setup lang="ts">
import localforage from 'localforage';
import { ElMessage } from 'element-plus';
import { reactive, computed } from 'vue';

type TimeAxisOptions =
  | {
      type: 'next';
      time: number;
    }
  | {
      type: 'previous';
    };

interface MakerLyricState {
  /** 歌词 */
  lyric: string;
  /** 时间轴 */
  timeAxis: string[];
}

const state = reactive<MakerLyricState>({
  lyric: '',
  timeAxis: [],
});
const lyricArray = computed((): string[] => state.lyric.split('\n'));

/**
 * 格式化数字
 *
 * @param number - 自然数
 */
const formatNumber = (number: number): string => {
  const prefix = number >= 10 ? '' : '0';
  return prefix + number;
};

/**
 * 获取时间轴信息
 *
 * @param time - 当前时间
 */
const getTimeAxiInfo = (time: number) => {
  const [integer, decimal] = time.toFixed(2).toString().split('.');
  const min = formatNumber(Math.floor(time / 60));
  const sec = formatNumber(Math.ceil(time % 60));
  const TimeAxi = {
    m: min,
    s: sec !== '60' ? sec : '00',
    x: decimal,
  };

  return TimeAxi;
};

/**
 * 时间轴控制
 *
 * @param options - 配置项
 */
const handleTimeAxis = (options: TimeAxisOptions): void => {
  if (!state.lyric) {
    ElMessage.error('没有歌词捏');
    return;
  }

  switch (options.type) {
    case 'next':
      if (state.timeAxis.length > lyricArray.value.length - 1) {
        ElMessage.warning('没有下一行啦~');
        return;
      }
      const { m, s, x } = getTimeAxiInfo(options.time);
      const timeAxi = `[${m}:${s}.${x}]`;

      state.timeAxis.push(timeAxi);
      break;
    case 'previous':
      if (!state.timeAxis.length) {
        ElMessage.warning('没有上一行啦~');
        return;
      }
      state.timeAxis.pop();
      break;
  }
  console.log(state.timeAxis);
};

defineExpose({
  handleTimeAxis,
});
</script>

<template>
  <div class="maker-lyric-container">
    <div class="lyric mb-10">
      <el-input v-model="state.lyric" :rows="14" type="textarea" placeholder="请输入歌词" />
    </div>

    <div class="overview">
      <el-input class="mb-10" :value="lyricArray[state.timeAxis.length - 2]" readonly>
        <template #prepend>上一行</template>
      </el-input>

      <el-input class="mb-10" :value="lyricArray[state.timeAxis.length - 1]" readonly>
        <template #prepend>当前行</template>
      </el-input>

      <el-input class="mb-10" :value="lyricArray[state.timeAxis.length]" readonly>
        <template #prepend>下一行</template>
      </el-input>
    </div>
  </div>
</template>

<style scoped lang="scss">
.maker-lyric-container { }
</style>
