<script setup lang="ts">
import localforage from 'localforage';
import { ref, reactive, onMounted } from 'vue';
import { ElMessage, genFileId } from 'element-plus';
import type { UploadInstance, UploadProps, UploadRawFile, UploadFile } from 'element-plus';

interface MakerMusicState {
  /** 音乐 Base64 */
  data?: string;
  /** 文件名 */
  filename?: string;
}

const state = reactive<MakerMusicState>({});
const audioRef = ref<HTMLAudioElement | null>(null);
const uploadRef = ref<UploadInstance | null>(null);

/**
 * 覆盖前一个文件
 *
 * @param files - 上传的文件列表
 */
const handleExceed: UploadProps['onExceed'] = files => {
  uploadRef.value!.clearFiles();
  const file = files[0] as UploadRawFile;
  file.uid = genFileId();
  uploadRef.value!.handleStart(file);
};

/**
 * 文件转 base64
 *
 * @param file - 音乐文件
 */
const fileToBase64 = (file: File): Promise<string> => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();

    reader.addEventListener('load', () => {
      resolve(<string>reader.result);
    });
    reader.addEventListener('error', () => {
      reject();
    });
    reader.readAsDataURL(file);
  });
};

/**
 * 选取音乐
 *
 * @param file - 音乐文件
 */
const chooseMusic = async (file: UploadFile) => {
  const rawFile = file.raw as File;
  const data = await fileToBase64(rawFile);

  state.filename = file.name;
  state.data = await fileToBase64(rawFile);

  localforage.setItem('music', {
    data,
    filename: file.name,
  });
};

/**
 * 音乐进度控制
 *
 * @param type - (播放/暂停)/停止/快进/倒带
 */
const handleMusic = (type: 'play' | 'stop' | 'rewind' | 'forward'): void => {
  if (!audioRef.value || !state.data) {
    ElMessage.error('你还没选歌呢！');
    return;
  }
  const { currentTime } = audioRef.value;

  switch (type) {
    case 'play':
      audioRef.value.paused ? audioRef.value.play() : audioRef.value.pause();
      break;
    case 'stop':
      audioRef.value.pause();
      audioRef.value.currentTime = 0;
      break;
    case 'rewind':
      audioRef.value.currentTime = currentTime + 5;
      audioRef.value.play();
      break;
    case 'forward':
      audioRef.value.currentTime = currentTime - 5;
      audioRef.value.play();
      break;
  }
};

/**
 * 获取当前歌曲播放时间
 */
const getCurrentTime = (): number => audioRef.value!.currentTime;

defineExpose({
  handleMusic,
  getCurrentTime,
});

onMounted(() => {
  // 初始化默认音量
  audioRef.value!.volume = 0.2;
});
</script>

<template>
  <div class="maker-music-container">
    <audio ref="audioRef" :src="state.data"></audio>

    <el-input class="input-with-select" v-model="state.filename" placeholder="请选择歌曲" readonly>
      <template #append>
        <el-upload
          ref="uploadRef"
          accept="audio/*"
          :limit="1"
          :auto-upload="false"
          :show-file-list="false"
          :on-change="chooseMusic"
          :on-exceed="handleExceed"
        >
          <el-button type="primary">选择</el-button>
        </el-upload>
      </template>
    </el-input>
  </div>
</template>

<style lang="scss">
.maker-music-container {
  .input-with-select {
    .el-input-group__append {
      color: var(--el-color-white);
      background-color: var(--el-color-primary);

      &:hover {
        background-color: var(--el-color-primary-light-3);
      }
    }
  }
}
</style>
