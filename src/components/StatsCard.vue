<script setup>
import {formatBandwithBytes, formatBytes} from "../utils/utils.js";
import {inject, ref} from "vue";

const handleChangeType = inject('handleChangeType')
const networkModalVisible = ref(false)

const { stats, type } = defineProps({
  stats: {
    type: Object,
    default: () => ({
      total: 0,
      online: 0,
      offline: 0,
      bandwidth_up: 0,
      bandwidth_down: 0,
      traffic_up: 0,
      traffic_down: 0,
      hosts: []
    }),
  },
  type: {
    type: String,
    default: "all",
  }
})

const handleNetworkClick = () => {
  networkModalVisible.value = true
}
</script>

<template>

  <div class="hero">
    <a-row :gutter="20">
      <a-col :span="6" :xs="24" :sm="24" :md="6" :lg="6" :sl="6">
        <div class="hero-card all" :class="type === 'all' ? 'is-active' :''" @click="handleChangeType('all')">
          <div class="title">服务器总数</div>
          <div class="value">
            <div class="status" style="background: #005fe7;"></div>
            <span class="num">{{stats.total}} 台</span>
          </div>
        </div>
      </a-col>
      <a-col :span="6" :xs="24" :sm="24" :md="6" :lg="6" :sl="6">
        <div class="hero-card online" :class="type === 'online' ? 'is-active' :''" @click="handleChangeType('online')">
          <div class="title">在线服务器</div>
          <div class="value">
            <div class="status" style="background: #1fb416;"></div>
            <span class="num">{{stats.online}} 台</span>
          </div>
        </div>
      </a-col>
      <a-col :span="6" :xs="24" :sm="24" :md="6" :lg="6" :sl="6">
        <div class="hero-card offline" :class="type === 'offline' ? 'is-active' :''" @click="handleChangeType('offline')">
          <div class="title">离线服务器</div>
          <div class="value">
            <div class="status" style="background: #b41616;"></div>
            <span class="num">{{stats.offline}} 台</span>
          </div>
        </div>
      </a-col>
      <a-col :span="6" :xs="24" :sm="24" :md="6" :lg="6" :sl="6">
        <div class="hero-card network" @click="handleNetworkClick">
          <div class="title">网络情况</div>
          <div class="value" style="display: block;">
            <div>
              流量
              <icon-arrow-up style="font-size: 14px;color: #d09453;" />
              <span style="font-size: 14px;color: #d09453;"> {{formatBytes(stats.traffic_up)}}</span>
              &nbsp;
              <icon-arrow-down style="font-size: 14px;color: #9a5fcd;" />
              <span style="font-size: 14px;color: #9a5fcd;">{{formatBytes(stats.traffic_down)}}</span>
            </div>
            <div>
              带宽
              <icon-up-circle style="font-size: 14px;" />
              <span style="font-size: 14px;"> {{formatBandwithBytes(stats.bandwidth_up)}}</span>
              &nbsp;
              <icon-down-circle style="font-size: 14px;" />
              <span style="font-size: 14px;"> {{formatBandwithBytes(stats.bandwidth_down)}}</span>
            </div>
          </div>
        </div>
      </a-col>
    </a-row>
  </div>

  <a-modal v-model:visible="networkModalVisible" title="网络流量排行" :footer="false" width="800px">
    <a-tabs default-active-key="upload">
      <a-tab-pane key="upload" title="上传排行">
        <div class="rank-list">
          <div class="rank-item" v-for="(host, index) in stats.hosts?.slice().sort((a, b) => b.State.NetOutSpeed - a.State.NetOutSpeed).slice(0, 10)" :key="index">
            <span class="rank-num">{{index + 1}}</span>
            <span class="host-name">{{host.Host.Name}}</span>
            <span class="speed">{{formatBytes(host.State.NetOutSpeed)}}/s</span>
          </div>
        </div>
      </a-tab-pane>
      <a-tab-pane key="download" title="下载排行">
        <div class="rank-list">
          <div class="rank-item" v-for="(host, index) in stats.hosts?.slice().sort((a, b) => b.State.NetInSpeed - a.State.NetInSpeed).slice(0, 10)" :key="index">
            <span class="rank-num">{{index + 1}}</span>
            <span class="host-name">{{host.Host.Name}}</span>
            <span class="speed">{{formatBytes(host.State.NetInSpeed)}}/s</span>
          </div>
        </div>
      </a-tab-pane>
      <a-tab-pane key="total-upload" title="总上传排行">
        <div class="rank-list">
          <div class="rank-item" v-for="(host, index) in stats.hosts?.slice().sort((a, b) => b.State.NetOutTransfer - a.State.NetOutTransfer).slice(0, 10)" :key="index">
            <span class="rank-num">{{index + 1}}</span>
            <span class="host-name">{{host.Host.Name}}</span>
            <span class="speed">{{formatBytes(host.State.NetOutTransfer)}}</span>
          </div>
        </div>
      </a-tab-pane>
      <a-tab-pane key="total-download" title="总下载排行">
        <div class="rank-list">
          <div class="rank-item" v-for="(host, index) in stats.hosts?.slice().sort((a, b) => b.State.NetInTransfer - a.State.NetInTransfer).slice(0, 10)" :key="index">
            <span class="rank-num">{{index + 1}}</span>
            <span class="host-name">{{host.Host.Name}}</span>
            <span class="speed">{{formatBytes(host.State.NetInTransfer)}}</span>
          </div>
        </div>
      </a-tab-pane>
    </a-tabs>
  </a-modal>
</template>

<style scoped lang="scss">
.hero {
  margin: 30px 10px 0px 10px;

  .hero-card {
    margin-bottom: 20px;
    padding: 12px 24px;
    border: 2px solid #e5e5e5;
    border-radius: 6px;
    background: #ffffff;
    min-height: 72px;
    box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
    cursor: pointer;

    &.is-active,
    &:hover {
      &.all {
        border-color: #005fe7;
      }

      &.online {
        border-color: #1fb416;
      }

      &.offline {
        border-color: #b41616;
      }
    }

    .title {
      margin-top: 6px;
      font-size: 16px;
      font-weight: 500;
      margin-bottom: 14px;
    }

    .value {
      display: flex;
      align-items: center;
      .status {
        margin-right: 6px;
        width: 8px;
        height: 8px;
        border-radius: 10px;
        background: #333333;
      }

      .num {
        font-size: 20px;
        font-weight: 600;
      }
    }
  }
}

body[arco-theme='dark'] {
  .hero-card {
    border: 2px solid rgb(255 255 255 / 16%);
    box-shadow: none;
    background-color: #000000;
    color: #ffffff;
  }
}

.hero {
  .hero-card {
    &.network {
      cursor: pointer;
      transition: all 0.3s ease;

      &:hover {
        transform: translateY(-2px);
        box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
      }
    }
  }
}

.rank-list {
  .rank-item {
    padding: 12px;
    display: flex;
    align-items: center;
    border-bottom: 1px solid #f0f0f0;

    &:last-child {
      border-bottom: none;
    }

    .rank-num {
      width: 24px;
      height: 24px;
      line-height: 24px;
      text-align: center;
      background: #f5f5f5;
      border-radius: 12px;
      margin-right: 12px;
      font-size: 12px;
      color: #666;

      &:nth-child(1) {
        background: #fff1f0;
        color: #ff4d4f;
      }

      &:nth-child(2) {
        background: #fff7e6;
        color: #ffa940;
      }

      &:nth-child(3) {
        background: #f6ffed;
        color: #73d13d;
      }
    }

    .host-name {
      flex: 1;
      font-weight: 500;
    }

    .speed {
      color: #666;
    }
  }
}

body[arco-theme='dark'] {
  .rank-list {
    .rank-item {
      border-bottom-color: #333;

      .rank-num {
        background: #333;
        color: #999;
      }

      .speed {
        color: #999;
      }
    }
  }
}
</style>