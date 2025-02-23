<template>
  <div class="bucket-container">
    <c-card :title="'存储桶'" v-loading="list.loading">
      <el-row>
        <!-- 新建 -->
        <el-col :xl="6" :lg="8" :md="12">
          <div class="bucket-item" @click="itemOperate(null, 'edit')">
            <div class="bucket-item-square">
              <span class="trialfont trial-bianji"></span>
            </div>
            <div class="bucket-item-tip">
              <span class="trialfont trial-bianji"></span>新建存储桶
            </div>
          </div>
        </el-col>

        <!-- 循环 -->
        <el-col :xl="6" :lg="8" :md="12" v-for="(item, index) in list.data" :key="index">
          <div class="bucket-item">
            <div class="bucket-item-square">
            <span
              :class="[configStore.systemConfig.system.icon_font, bucketIcons[item.type].value]"
              :style="{
                color: bucketIcons[item.type].color
              }"></span>
            </div>
            <div class="bucket-item-tags">
              <el-tag size="small" type="success">{{ item.tag }}</el-tag>
              <el-tag size="small" :type="item.visible ? 'primary' : 'error'">{{ item.visible ? '已启用' : '已禁用' }}</el-tag>
            </div>
            <div class="bucket-item-content">
              <div class="bucket-content-title">{{ item.name }}</div>
              <div class="bucket-content-count">
                <el-tag type="primary" size="small">版本号: {{ item.version }}</el-tag>
                <el-tag type="info" size="small">图片数量: {{ getStats(item, 'bucket_count') }}</el-tag>
                <el-tag type="info" size="small">占用存储: {{ getStats(item, 'bucket_storage') }}MB</el-tag>
              </div>
              <div class="bucket-content-time">
                <el-tooltip
                  v-if="getVersion(item).version_last !== item.version"
                  effect="dark"
                  :content="'有新版本，最新版本:' + getVersion(item).version_last"
                  placement="top">
                  <el-tag type="danger" effect="dark" size="small">New</el-tag>
                </el-tooltip>
                创建时间: {{ item.createdAt }}
              </div>
            </div>
            <div class="bucket-item-action">
              <div class="bucket-action-item" @click="itemOperate(item, 'edit')">编辑</div>
              <div class="bucket-action-item" @click="itemOperate(item, 'toggle')">{{ item.visible ? '禁用' : '启用' }}</div>
              <div class="bucket-action-item" @click="itemOperate(item, 'delete')">删除</div>
              <div class="bucket-action-item" @click="itemOperate(item, 'migrate')">数据迁移</div>
            </div>
          </div>
        </el-col>
      </el-row>
    </c-card>

    <!-- 编辑弹窗 -->
    <edit-dialog
      v-if="visible.edit"
      v-model="visible.edit"
      :detail="item.data"
      @submit="listGet"/>
    <!-- 数据迁移弹窗 -->
    <migrate-dialog
      v-if="visible.migrate"
      v-model="visible.migrate"
      :detail="item.data"
      @submit="listGet"/>
  </div>
</template>

<script lang="ts" setup>
import { reactive, ref } from 'vue'
import { BucketInter, DictInter, ListInter, UserInter } from '@/typings/interface'
import { useCtxInstance, useDeleteConfirm } from '@/hooks/global'
import Dict from '@/types/Dict'
import { PageResponse } from '@/typings/req-res'
import EditDialog from './EditDialog.vue'
import MigrateDialog from './MigrateDialog.vue'
import Bucket from '@/types/Bucket'
import useConfigStore from '@/store/config'
import { useFormat } from '@/hooks/date-time'
interface Stats {
  id: string
  bucket_storage: string
  bucket_count: number
}
interface Versions {
  id: string
  version: string
  version_last: string
}
/**
 * 实例
 */
const ctx = useCtxInstance()
const bucket = new Bucket()
const dict = new Dict()
const configStore = useConfigStore()
/**
 * 变量
 */
const list: ListInter<BucketInter, Stats, Versions> = reactive({
  total: 0,
  filters: {
    name: '',
    sort: 'updatedAt',
    order: 'desc'
  },
  data: [],
  stats: [],
  versions: [],
  loading: false
})
// 当前被操作项
let item = reactive({
  data: null
})
const visible = reactive({
  edit: false,
  migrate: false
})
// 存储桶图标
const bucketIcons = ref({})

/**
 * 逻辑处理
 */
// 获取数据
const listGet = () => {
  list.loading = true
  bucket.find({
    ...list.filters
  }).then((res: PageResponse<BucketInter, Stats, Versions>) => {
    list.total = res.total
    list.stats = res.stats.map(item => {
      item.bucket_storage = (parseInt(item.bucket_storage) / 1021 / 1024).toFixed(2)
      return item
    })
    list.versions = res.versions
    list.data = res.items.map(item => {
      item.createdAt = useFormat(item.createdAt)
      item.updatedAt = useFormat(item.updatedAt)
      const obj = JSON.parse(item.config)
      for (let key in obj) {
        obj[key] = obj[key].replace(/\$\{((config).*?)\}/g, (v, key) => {
          const keys = key.split('.')
          if (keys[0] === 'config') {
            return obj[keys[1]]
          }
        })
      }
      return {
        ...item,
        config_baseUrl: obj.baseUrl
      }
    })
    list.loading = false
  })
}
// 获取字典中的图标
const getBucketIcon = () => {
  dict.detailByPro('code', 'bucket_source_icon').then((res: DictInter) => {
    res.values.forEach(item => {
      bucketIcons.value[item.label] = {
        value: item.value,
        color: item.color
      }
    })
    listGet()
  })
}
getBucketIcon()
// 获取统计数据
const getStats = (item: BucketInter, key) => {
  const tmp = list.stats.find(el => el.id === item.id)
  if (tmp[key] && tmp[key] !== 'NaN') {
    return tmp[key]
  }
  return 0
}
// 获取版本
const getVersion = (item: BucketInter) => {
  return list.versions.find(el => el.id === item.id)
}

// 操作
const itemOperate = (data: BucketInter, type) => {
  item.data = data
  visible[type] = true
  if (type === 'delete') {
    useDeleteConfirm('确定要删除本存储桶吗？(删除后关联的图片将无法访问)').then(() => {
      bucket.delete(data.id, data.uid)
      .then(res => {
        ctx.$message({ message: '删除成功', type: 'success', duration: 1000 })
        listGet()
      })
    })
  }
  if (type === 'toggle') {
    useDeleteConfirm(`确定要${ data.visible ? '禁用' : '启用' }该存储桶吗？`).then(() => {
      bucket.toggle(data.id).then(res => {
        ctx.$message({ message: `${ data.visible ? '禁用' : '启用' }成功`, type: 'success', duration: 1000 })
        listGet()
      })
    })
  }
  // if (type === 'migrate') {
  //   console.log('数据迁移')
  // }
}
</script>

<style lang="scss">
@import '@/styles/flex-layout.scss';
$border: 1px solid #e9ecef;
$border-active: 1px solid #32cfaa;
$text-color: #747a80;
$text-color-active: #32cfaa;
@mixin border($active: false) {
  border-radius: 6px;
  border-style: solid;
  border-width: 1px;
  @if $active {
    border-color: #32cfaa;
  } @else {
    border-color: #e9ecef;
  }
}
.bucket-container {
  width: 100%;
  height: 100%;
  .el-card__body {
    padding-top: 50px;
    overflow: auto;
  }
  .bucket-item {
    height: 209px;
    @include border;
    position: relative;
    @include flex-layout(column);
    cursor: pointer;
    transition: transform .3s;
    .bucket-item-tip {
      width: 100%;
      height: 100%;
      color: #747a80;
      @include flex-layout-align(row, center, center);
      span {
        margin-right: 5px;
        font-size: 20px;
      }
    }
    .bucket-item-square {
      height: 50px;
      width: 50px;
      line-height: 50px;
      transform: rotate(45deg);
      transition: all 0.5s ease;
      text-align: center;
      @include border;
      top: -25px;
      position: absolute;
      left: 10px;
      background: #fff;
      @include flex-layout-align(row, center, center);
      span {
        font-size: 30px;
        transform: rotate(-45deg);
        color: #535353;
      }
    }
    .bucket-item-tags {
      position: absolute;
      right: 16px;
      top: 8px;
      text-align: right;
      .el-tag + .el-tag {
        margin-left: 5px;
      }
    }
    .bucket-item-content {
      flex: 1;
      padding: 40px 16px 16px;
      position: relative;
      overflow: hidden;
      .bucket-content-title {
        font-size: 16px;
        // text-align: right;
        font-weight: 600;
        color: #535353;
      }
      .bucket-content-count {
        margin-top: 10px;
        .el-tag {
          margin: 2px 4px 2px 0;
        }
      }
      .bucket-content-time {
        // line-height: 20px;
        height: 20px;
        display: flex;
        align-items: center;
        font-size: 14px;
        margin-top: 10px;
        color: rgba(0,0,0,.45);
        .el-tag {
          margin-right: 5px;
          background-color: red !important;
          border-color: red !important;
        }
      }
    }
    .bucket-item-action {
      height: 42px;
      flex-shrink: 0;
      border-top: $border;
      @include flex-layout(row);
      .bucket-action-item {
        flex: 1;
        @include flex-layout-align(row, center, center);
        color: $text-color;
        cursor: pointer;
        + .bucket-action-item {
          border-left: $border;
        }
      }
    }
    &:hover {
      @include border(true);
      transform: translateY(-10px);
      .bucket-item-square {
        @include border(true);
        background: #32cfaa;
        span {
          color: #fff;
        }
      }
      .bucket-item-action {
        border-top: $border-active;
        .bucket-action-item {
          color: $text-color-active;
          + .bucket-action-item {
            border-left: $border-active;
          }
        }
      }
    }
  }
  .el-row {
    .el-col {
      padding: 10px 15px 40px;
    }
  }
}
</style>