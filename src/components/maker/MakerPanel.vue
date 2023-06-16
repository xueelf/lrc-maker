<script setup lang="ts">
import { invoke } from '@tauri-apps/api/tauri';
import { genFileId, ElMessage } from 'element-plus';
import { ref, reactive, computed, onMounted } from 'vue';
import type { UploadInstance, UploadProps, UploadRawFile, UploadFile } from 'element-plus';
// import MakerLyric from '~/makerLyric.vue';

interface MakerPanelState {
  music: string;
  filename: string;
  raw_lyric: string;
  timeAxis: string[];
  status: 'none' | 'read' | 'making' | 'paused';
  activeName: 'raw' | 'ripe';
}

const state = reactive<MakerPanelState>({
  music: '',
  filename: '',
  raw_lyric: '',
  timeAxis: [],
  status: 'none',
  activeName: 'raw',
});

const musicRef = ref<HTMLAudioElement>();
const uploadRef = ref<UploadInstance>();

const lyricArray = computed((): string[] => state.raw_lyric.split('\n'));
const lyric = computed((): string => {
  const lyrics = [];

  for (let i = 0; i < state.timeAxis.length; i++) {
    const timeAxi = state.timeAxis[i];
    const lyric = lyricArray.value[i];

    lyrics.push(`${timeAxi} ${lyric}`);
  }
  return lyrics.join('\n');
});

onMounted(() => {
  state.raw_lyric = localStorage.getItem('raw_lyric') ?? '';
});

const chooseMusic = (file: UploadFile) => {
  if (state.music) {
    state.raw_lyric = '';
    localStorage.removeItem('raw_lyric');
  }
  const url = window.URL.createObjectURL(<File>file.raw);

  state.filename = file.name;
  state.music = url;
  state.status = 'read';
  state.activeName = 'raw';
};

const handleExceed: UploadProps['onExceed'] = files => {
  uploadRef.value!.clearFiles();
  const file = files[0] as UploadRawFile;
  file.uid = genFileId();
  uploadRef.value!.handleStart(file);
};

/**
 * 歌词控制
 */
const handleLyric = () => {
  if (!state.raw_lyric.trim()) {
    ElMessage.error('没有歌词捏');
    return;
  }

  switch (state.status) {
    case 'read':
      listenKeyboard();

      musicRef.value!.volume = 0.2;
      musicRef.value!.play();
      state.status = 'making';
      state.activeName = 'ripe';
      break;

    case 'making':
      document.onkeyup = null;

      musicRef.value!.pause();
      state.status = 'paused';
      state.activeName = 'raw';
      break;
    case 'paused':
      listenKeyboard();

      musicRef.value!.play();
      state.status = 'making';
      state.activeName = 'ripe';
      break;
  }
};

/**
 * 终止制作
 */
const stopMake = () => {
  musicRef.value!.pause();
  musicRef.value!.currentTime = 0;

  state.status = 'read';
  state.activeName = 'raw';
  state.timeAxis.length = 0;
};

/**
 * 格式化数字
 *
 * @param number
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
 * 歌词控制
 */
const handleInputLyric = () => {
  localStorage.setItem('raw_lyric', state.raw_lyric);
};

/**
 * 时间轴控制
 *
 * @param type - 上一行/下一行
 */
const handleTimeAxis = (type: 'next' | 'previous') => {
  const { currentTime } = musicRef.value!;
  const { m, s, x } = getTimeAxiInfo(currentTime);

  switch (type) {
    case 'next':
      if (state.timeAxis.length > lyricArray.value.length - 1) {
        return;
      }
      const timeAxi = `[${m}:${s}.${x}]`;

      state.timeAxis.push(timeAxi);
      break;
    case 'previous':
      state.timeAxis.pop();
      break;
  }
};

/**
 * 音乐进度控制
 *
 * @param type - 快进/倒带
 */
const handleMusic = (type: 'forward' | 'rewind') => {
  const { currentTime } = musicRef.value!;

  switch (type) {
    case 'forward':
      musicRef.value!.currentTime = currentTime - 5;
      break;
    case 'rewind':
      musicRef.value!.currentTime = currentTime + 5;
      break;
  }
};

/**
 * 监听快捷键
 */
const listenKeyboard = () => {
  document.onkeyup = event => {
    const { code } = event;

    switch (code) {
      case 'ArrowUp':
        handleTimeAxis('next');
        break;
      case 'ArrowDown':
        handleTimeAxis('previous');
        break;
      case 'ArrowLeft':
        handleMusic('forward');
        break;
      case 'ArrowRight':
        handleMusic('rewind');
        break;
    }
  };
};
</script>

<template>
  <div class="maker-panel-container">
    <div class="music">
      <el-input v-model="state.filename" placeholder="请选择歌曲" readonly>
        <template #append>
          <el-upload
            ref="uploadRef"
            accept="audio/*"
            :show-file-list="false"
            :limit="1"
            :on-change="chooseMusic"
            :on-exceed="handleExceed"
            :auto-upload="false"
          >
            <el-button type="primary">选择</el-button>
          </el-upload>
        </template>
      </el-input>

      <audio ref="musicRef" :src="state.music" @ended="state.status = 'read'"></audio>
    </div>

    <div class="lyric">
      <el-tabs v-model="state.activeName" class="mb-10">
        <el-tab-pane label="歌词" name="raw">
          <el-input
            v-model="state.raw_lyric"
            :rows="14"
            type="textarea"
            placeholder="请输入歌词"
            @input="handleInputLyric"
          />
        </el-tab-pane>

        <el-tab-pane label="时间轴" name="ripe">
          <el-input :value="lyric" :rows="14" type="textarea" readonly placeholder="暂无歌词" />
        </el-tab-pane>
      </el-tabs>

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

      <div>
        <el-button v-if="state.status === 'none'" type="primary" disabled>制作歌词</el-button>
        <el-button v-if="state.status === 'read'" type="primary" @click="handleLyric"
          >制作歌词</el-button
        >
        <el-button v-else-if="state.status === 'making'" type="primary" @click="handleLyric"
          >暂停</el-button
        >
        <template v-else-if="state.status === 'paused'">
          <el-button type="primary" @click="handleLyric">继续</el-button>
          <el-popconfirm
            title="歌词将不会保存"
            confirm-button-text="确认"
            cancel-button-text="取消"
            @confirm="stopMake"
          >
            <template #reference>
              <el-button type="danger">终止</el-button>
            </template>
          </el-popconfirm>
        </template>
      </div>
    </div>
  </div>
</template>

<style scoped lang="scss">
.maker-panel-container {
}
</style>
