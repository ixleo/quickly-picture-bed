<template>
  <div class="album-item">
    <!-- <img class="album-item-cover" :src="album.cover_preview" /> -->
    <div class="album-item-cover">
      <template v-if="album.cover">
        <img v-if="loadError" :src="'/error.png'" />
        <v-lazy-image
          v-else
          :class="[bindClass]"
          :src="album.cover_preview"
          :src-placeholder="placeholder"
          :key="album.id"
          @load="handleLoad"
          @error="handleRenderError"></v-lazy-image>
      </template>
      <template v-else>
        <img src="/default.jpg" alt="">
      </template>
    </div>
    <div class="album-item__content">
      <div class="album-item-name">{{ album.name }}({{ album.count }})</div>
      <div class="album-item-divider"></div>
      <div class="album-item-desc">{{ album.desc }}</div>
    </div>
    <div class="album-item-action">
      <el-button type="success" @click.stop="handleClick('upload')">去上传</el-button>
      <el-button type="primary" @click.stop="handleClick('edit')">编辑</el-button>
      <el-button type="danger" @click.stop="handleClick('delete')">删除</el-button>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { AlbumInter } from '@/typings/interface';
import VLazyImage from 'v-lazy-image';
import { ref } from 'vue';
const placeholder = new URL('../../../views/gallery/loading.gif', import.meta.url).href

interface Props {
  album: AlbumInter
}
withDefaults(defineProps<Props>(), {
  album: () => ({
    id: '',
    name: '',
    desc: '',
    cover: '',
    background: '',
    count: 0
  } as AlbumInter)
})
const emit = defineEmits(['submit'])

const bindClass = ref('')

const handleClick = (type) => {
  emit('submit', type)
}

function handleLoad (e) {
  bindClass.value = 'loaded'
}
const loadError = ref(false)
function handleRenderError () {
  loadError.value = true
}
</script>

<style lang="scss">
.album-item {
  height: 250px;
  position: relative;
  cursor: pointer;
  overflow: hidden;
  // background: #000;
  background: #009688;
  border-radius: 8px;

  .album-item-cover {
    border-radius: 8px;
    display: flex;
    justify-content: center;
    align-items: center;
    width: 100%;
    height: 100%;
    .loaded {
      width: 100%;
      height: 100%;
      transition: all 0.35s;
      opacity: 0.8;
      object-fit: cover;
    }
  }

  .album-item__content {
    width: 100%;
    height: 100%;
    position: absolute;
    top: 0;
    left: 0;
    padding: 0%;
    padding: 24px;
    color: #fff;
    display: flex;
    flex-direction: column;

    .album-item-name {
      line-height: 1.5;
      font-size: 20px;
      font-weight: bold;
      position: relative;
      margin: 7px 0;
      overflow: hidden;
      -webkit-line-clamp: 2;
      display: -webkit-box;
      -webkit-box-orient: vertical;
    }

    .album-item-divider {
      height: 2px;
      background: #fff;
      margin: 5px 0;
      transition: all 0.35s;
      // transform: translate3d(-100%, 0, 0);
      opacity: 0;
      width: 0%;
    }

    .album-item-desc {
      padding: 7px 0 0;
      letter-spacing: 1px;
      font-size: 14px;
      line-height: 1.5;
      -webkit-line-clamp: 4;
      overflow: hidden;
      text-overflow: ellipsis;
      display: -webkit-box;
      -webkit-box-orient: vertical;
      transition: all 0.35s;
      transform: translate3d(100%, 0, 0);
      opacity: 0;
    }
  }

  .album-item-action {
    width: 100%;
    padding: 10px;
    position: absolute;
    bottom: 0;
    left: 0;
    display: flex;
    justify-content: center;
    opacity: 0;
    transition: all 0.35s;
    transform: translate3d(0, 100%, 0);
  }

  &:hover {
    .loaded {
      transform: scale(1.2);
      opacity: 0.60;
    }

    .album-item-divider {
      width: 100%;
      opacity: 1;
    }

    
    .album-item-desc,
    .album-item-action {
      opacity: 1;
      transform: translate3d(0, 0, 0);
    }
  }
}
</style>