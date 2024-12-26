<script setup>
import {computed, onMounted, provide, ref} from "vue";
import moment from 'moment'
import CPU from "@/components/CPU.vue";
import Mem from "@/components/Mem.vue";
import NetIn from "@/components/NetIn.vue";
import NetOut from "@/components/NetOut.vue";
import axios from "axios";
import { Message } from "@arco-design/web-vue";
import StatsCard from "@/components/StatsCard.vue";
import {formatBytes, formatTimeStamp, formatUptime, calculateRemainingDays} from '@/utils/utils'
import highcharts from 'highcharts'
import accessibility from 'highcharts/modules/accessibility'

// 初始化 Highcharts 可访问性模块
accessibility(highcharts)

// 配置 Highcharts 全局选项
highcharts.setOptions({
  accessibility: {
    enabled: true,
    description: 'StellarFRP 节点监控图表'
  }
})

const socketURL = ref('')
const apiURL = ref('')

const theme = window.localStorage.getItem('theme') || 'light'
const dark = ref(theme !== 'light')

const handleChangeDark = () => {
  dark.value = !dark.value

  if (dark.value) {
    window.localStorage.setItem('theme','dark')
    document.body.setAttribute('arco-theme', 'dark')
  } else {
    // 恢复亮色主题
    window.localStorage.setItem('theme','light')
    document.body.removeAttribute('arco-theme');
   }
}

const area = ref([])
const selectArea = ref('all')

const type = ref('all')

const data = ref([])

const selectHost = ref('')

const charts = ref({})

const cpuRef = ref(null)
const memRef = ref(null)
const netInRef = ref(null)
const netOutRef = ref(null)

const host = computed(() => {
  if (selectArea.value === 'all') {
    return data.value
  }

  return data.value.filter(item => item.Host.Name.slice(0, 2) === selectArea.value)
})

const sortBy = ref('name') // 'name', 'upload', 'download', 'tunnels'
const sortOrder = ref('asc') // 'asc' 或 'desc'

const handleSort = (type) => {
  if (sortBy.value === type) {
    // 如果点击的是当前排序字段，则切换排序顺序
    sortOrder.value = sortOrder.value === 'asc' ? 'desc' : 'asc'
  } else {
    // 如果点击的是新字段，则设置为升序
    sortBy.value = type
    sortOrder.value = 'asc'
  }
}

const hosts = computed(() => {
  let filteredHosts = host.value
  if (type.value === 'online') {
    filteredHosts = host.value.filter(item => item.status)
  } else if (type.value === 'offline') {
    filteredHosts = host.value.filter(item => !item.status)
  }

  return filteredHosts.slice().sort((a, b) => {
    const order = sortOrder.value === 'asc' ? 1 : -1
    
    switch (sortBy.value) {
      case 'name':
        return a.Host.Name.localeCompare(b.Host.Name) * order
      case 'upload':
        return (a.State.NetOutSpeed - b.State.NetOutSpeed) * order
      case 'download':
        return (a.State.NetInSpeed - b.State.NetInSpeed) * order
      case 'tunnels':
        if (!tunnelConfig.value.enabled) return 0
        const aNodeName = nodeNameMap[a.Host.Name] || a.Host.Name
        const bNodeName = nodeNameMap[b.Host.Name] || b.Host.Name
        const aTunnels = simpleData.value.find(node => node.node === aNodeName)?.online || 0
        const bTunnels = simpleData.value.find(node => node.node === bNodeName)?.online || 0
        return (aTunnels - bTunnels) * order
      case 'load':
        return (a.State.Load1 - b.State.Load1) * order
      default:
        return 0
    }
  })
})

// 添加转换为MB的辅助函数
const convertToMB = (bytes) => {
  return bytes / (1024 * 1024)
}

const stats = computed(() => {
  const online = host.value.filter(item => item.status)
  let bandwidth_up = 0
  let bandwidth_down = 0
  let traffic_up = 0
  let traffic_down = 0

  host.value.forEach((item) => {
    // 先将所有流量转换为MB再累加
    bandwidth_up += convertToMB(item.State.NetOutSpeed)
    bandwidth_down += convertToMB(item.State.NetInSpeed)
    traffic_up += convertToMB(item.State.NetOutTransfer)
    traffic_down += convertToMB(item.State.NetInTransfer)
  })

  // 转回节以保持与原有格式化函数兼容
  return {
    total: host.value.length,
    online: online.length,
    offline: host.value.length - online.length,
    bandwidth_up: bandwidth_up * 1024 * 1024,
    bandwidth_down: bandwidth_down * 1024 * 1024,
    traffic_up: traffic_up * 1024 * 1024,
    traffic_down: traffic_down * 1024 * 1024,
    hosts: host.value
  }
})

let socket = null

let nowtime = (Math.floor(Date.now() / 1000))

// 修改 tunnelConfig 的定义
const tunnelConfig = ref({
  enabled: false,
  apiURL: ''
})

const fetchConfig = async () => {
  try {
    const res = await axios.get('/config.json')
    socketURL.value = res.data.socket
    apiURL.value = res.data.apiURL
    // 从配置中读取 tunnelConfig 对象
    tunnelConfig.value = res.data.tunnelConfig || { enabled: false, apiURL: '' }
  } catch (e) {
    Message.error('获取配置失败')
  }
}

const initScoket = async () => {
  await fetchConfig()

  socket = new WebSocket(socketURL.value);  // 替换为实际的 WebSocket 服务器 URL

  socket.onmessage = function(event) {
    try {
      const message = event.data;
      const res = JSON.parse(message.replace('data: ', ''))
      if (!res) {
        console.warn('WebSocket 返回空数据')
        return
      }
      if (!Array.isArray(res)) {
        console.warn('WebSocket 返回数据格式不正确，期望数组类型')
        return
      }
      area.value = Array.from(new Set(res.map(item => item?.Host?.Name?.slice(0, 2) || '').filter(Boolean)))
      data.value = res.map((host) => {
        if (!host?.Host?.Name) {
          console.warn('主机数据格式不正确:', host)
          return null
        }

        // 更新 selectedHost 数据
        if (selectedHost.value && selectedHost.value.Host.Name === host.Host.Name) {
          selectedHost.value = {
            ...host,
            status: (nowtime - host.TimeStamp <= 10) ? 1 : 0
          }
        }

        if (!charts.value[host.Host.Name]) {
          charts.value[host.Host.Name] = {
            cpu: [],
            mem: [],
            net_in: [],
            net_out: []
          }
        }

        if (charts.value[host.Host.Name].cpu.length == 2) {
          charts.value[host.Host.Name].cpu = charts.value[host.Host.Name].cpu.slice(1)
          charts.value[host.Host.Name].mem = charts.value[host.Host.Name].mem.slice(1)
          charts.value[host.Host.Name].net_in = charts.value[host.Host.Name].net_in.slice(1)
          charts.value[host.Host.Name].net_out = charts.value[host.Host.Name].net_out.slice(1)
        }

        charts.value[host.Host.Name].cpu.push([host.TimeStamp * 1000, host.State?.CPU || 0])
        charts.value[host.Host.Name].mem.push([host.TimeStamp * 1000, host.State?.MemUsed || 0])
        charts.value[host.Host.Name].net_in.push([host.TimeStamp * 1000, host.State?.NetOutSpeed || 0])
        charts.value[host.Host.Name].net_out.push([host.TimeStamp * 1000, host.State?.NetInSpeed || 0])

        return {
          ...host,
          status: (nowtime - host.TimeStamp <= 10) ? 1 : 0
        }
      }).filter(Boolean)

      setTimeout(() => sendPing(), 1000)

    } catch (error) {
      console.error('解析 WebSocket 消息时出错:', error);
      console.log('原始消息:', event.data);
    }
  };

  socket.onopen = function () {
    sendPing()
  }

  socket.onclose = function () {
    Message.warning('WebSocket已断连，正在重连...')

    initScoket()
  }
}

const sendPing = () => {
  nowtime = (Math.floor(Date.now() / 1000))
  socket.send('ping')
}

onMounted(async() => {
  if (dark.value) {
    document.body.setAttribute('arco-theme', 'dark')
  }

  await initScoket()
  handleFetchHostInfo()
  updateRunningTime()
  setInterval(updateRunningTime, 1000)
  
  // 只在首次加载时获取一次API数据
  fetchSimpleData()
})

const progressStatus = (value) => {
  if (value < 80) {
    return 'success';
  } else if (value < 90) {
    return 'warning';
  } else {
    return 'danger';
  }
}

const handleSelectArea = (area) => {
  selectArea.value = area
}

const handleSelectHost = (host) => {
  if (viewMode.value === 'list') {
    if (selectHost.value === host) {
      selectHost.value = ''
      return
    }
    selectHost.value = host
  } else {
    // 卡片视图模式
    const hostData = hosts.value.find(item => item.Host.Name === host)
    if (hostData) {
      selectedHost.value = hostData
      detailVisible.value = true
    }
  }
}

// 删除主机
const deleteVisible = ref(false)
const authSecret = ref('')
const deleteHostName = ref('')

const handleShowDelete = (name) => {
  authSecret.value =  window.localStorage.getItem('auth_secret') || ''
  deleteHostName.value = name
  deleteVisible.value = true
}

const handleDeleteHost = async () => {
  try {
    await axios.post(apiURL.value + '/delete', {
      "auth_secret": authSecret.value,
      "name": deleteHostName.value
    })

    window.localStorage.setItem('auth_secret', authSecret.value)

    Message.success('删除成功')

    deleteVisible.value = false
  } catch (e) {
    Message.error('删除失败，管理密钥错误')
  }
}

const handleClose = () => {
  deleteVisible.value = false
}

const hostInfo = ref({})

const handleFetchHostInfo = async () => {
  try {
    const res = await axios.get(apiURL.value + '/info')
    res.data.forEach((item) => {
      hostInfo.value[item.name] = item
    })
  } catch (e) {
    // Message.error('删除失败，管理密钥错误')
  }
}

const handleChangeType = (value) => {
  type.value = value
}

const editHostName = ref('')
const duetime = ref(0)
const buy_url = ref('')
const seller = ref('')
const price = ref('')

const editVisible = ref(false)

const handleShowEdit = (name) => {
  if (!hostInfo.value[name]) {
    editHostName.value = name
    duetime.value = 0
    buy_url.value = ''
    seller.value = ''
    price.value = ''
    editVisible.value = false
    authSecret.value = window.localStorage.getItem('auth_secret') || ''
    return
  }

  editHostName.value = name
  duetime.value = hostInfo.value[name].due_time
  buy_url.value = hostInfo.value[name].buy_url
  seller.value = hostInfo.value[name].seller
  price.value = hostInfo.value[name].price
  editVisible.value = true
  authSecret.value = window.localStorage.getItem('auth_secret') || ''

}

const handleEditHost = async () => {
  try {
    await axios.post(apiURL.value + '/info', {
      "name": editHostName.value,
      "due_time": new Date(duetime.value).getTime(),
      "buy_url": buy_url.value,
      "seller": seller.value,
      "auth_secret": authSecret.value,
      "price": price.value
    })

    window.localStorage.setItem('auth_secret', authSecret.value)

    Message.success('更新成功')

    handleFetchHostInfo()

    editVisible.value = false
  } catch (e) {
    Message.error('更新失败，管理密钥错误')
  }

}

const handleEditClose = () => {
  editVisible.value = false
}

provide('handleChangeType', handleChangeType)

const startTime = new Date('2024-12-20T19:00:00').getTime()
const runningTime = ref('')

const updateRunningTime = () => {
  const now = new Date().getTime()
  const diff = now - startTime
  
  const days = Math.floor(diff / (1000 * 60 * 60 * 24))
  const hours = Math.floor((diff % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60))
  const minutes = Math.floor((diff % (1000 * 60 * 60)) / (1000 * 60))
  const seconds = Math.floor((diff % (1000 * 60)) / 1000)
  
  runningTime.value = `${days}天${hours}小时${minutes}分${seconds}秒`
}

// 修改节点名称映射
const nodeNameMap = {
  '湖北节点-2': '湖北节点#2',
  '宁波节点-赞助者向南': '宁波节点'
}

const simpleData = ref([])

const fetchSimpleData = async () => {
  // 只在 tunnelConfig enabled 为 true 时获取数据
  if (!tunnelConfig.value.enabled) return
  
  try {
    const res = await axios.get(tunnelConfig.value.apiURL)
    if (res.data.code === 200) {
      simpleData.value = res.data.data
    }
  } catch (e) {
    console.error('获取简略数据失败:', e)
  }
}

// 修改默认视图为卡片
const viewMode = ref('card') // 'list' 或 'card'

const handleChangeView = () => {
  viewMode.value = viewMode.value === 'list' ? 'card' : 'list'
}

// 添加详细信息模态框相关变量
const detailVisible = ref(false)
const selectedHost = ref(null)

</script>

<template>
  <div class="max-container">
    <div class="header">
      <a class="logo" href="#">
        <img src="/favicon.ico" alt="StellarFRP" class="logo-icon" />
        <span>StellarFRP Status</span>
        <small style="font-weight: 400;opacity: .8"> ｜ StellarFrp-节点监控</small>
      </a>
      <div class="header-actions">
        <div class="sort-buttons">
          <a-dropdown>
            <a-button class="sort-btn">
              <template #icon>
                <icon-sort />
              </template>
              排序
              <template #icon-right>
                <icon-down />
              </template>
            </a-button>
            <template #content>
              <a-doption @click="handleSort('name')">
                按名称排序
                <icon-sort-ascending v-if="sortBy === 'name' && sortOrder === 'asc'" />
                <icon-sort-descending v-if="sortBy === 'name' && sortOrder === 'desc'" />
              </a-doption>
              <a-doption @click="handleSort('upload')">
                按上传流量排序
                <icon-sort-ascending v-if="sortBy === 'upload' && sortOrder === 'asc'" />
                <icon-sort-descending v-if="sortBy === 'upload' && sortOrder === 'desc'" />
              </a-doption>
              <a-doption @click="handleSort('download')">
                按下载流量排序
                <icon-sort-ascending v-if="sortBy === 'download' && sortOrder === 'asc'" />
                <icon-sort-descending v-if="sortBy === 'download' && sortOrder === 'desc'" />
              </a-doption>
              <a-doption @click="handleSort('tunnels')">
                按在线隧道排序
                <icon-sort-ascending v-if="sortBy === 'tunnels' && sortOrder === 'asc'" />
                <icon-sort-descending v-if="sortBy === 'tunnels' && sortOrder === 'desc'" />
              </a-doption>
              <a-doption @click="handleSort('load')">
                按负载排序
                <icon-sort-ascending v-if="sortBy === 'load' && sortOrder === 'asc'" />
                <icon-sort-descending v-if="sortBy === 'load' && sortOrder === 'desc'" />
              </a-doption>
            </template>
          </a-dropdown>
        </div>
        <a-button class="view-btn" :shape="'round'" @click="handleChangeView">
          <template #icon>
            <icon-apps v-if="viewMode === 'list'" />
            <icon-list v-else />
          </template>
        </a-button>
        <a-button class="theme-btn" :shape="'round'" @click="handleChangeDark">
          <template #icon>
            <icon-sun-fill v-if="!dark" />
            <icon-moon-fill v-else />
          </template>
        </a-button>
      </div>
    </div>

    <div class="area-tabs">
      <div class="area-tab-item" :class="selectArea === 'all' ? 'is-active' : ''" @click="handleSelectArea('all')">
        全部地区
      </div>
      <div class="area-tab-item" :class="selectArea === item ? 'is-active' : ''" v-for="(item, index) in area" :key="item" @click="handleSelectArea(item)">
        <span :class="`flag-icon flag-icon-${item.replace('UK', 'GB').toLowerCase()}`" style="margin-right: 3px;"></span> {{item}}
      </div>
    </div>

    <StatsCard :type="type" :stats="stats" @handleChangeType="handleChangeType" />
    
    <!-- 列表视图 -->
    <div class="monitor-card" v-if="viewMode === 'list'">
      <div class="monitor-item" :class="selectHost === item.Host.Name ? 'is-active' : ''" v-for="(item, index) in hosts" @click="handleSelectHost(item.Host.Name)" :key="index">
        <div class="name">
          <div class="title">
            <span :class="`flag-icon flag-icon-${item.Host.Name.slice(0, 2).replace('UK', 'GB').toLowerCase()}`"></span>
            {{item.Host.Name}}
          </div>
          <div class="status" :class="item.status ? 'online' : 'offline'">
            <span>{{item.status  ? '在线' : '离线'}}</span>
            <span style="margin-left: 6px;">{{formatUptime(item.State.Uptime)}}</span>
          </div>
        </div>
        <div class="platform">
          <div class="monitor-item-title">系统</div>
          <div class="monitor-item-value">{{item.Host.Platform}} {{item.Host.PlatformVersion}}</div>
        </div>
        <div class="cpu">
          <div class="monitor-item-title">CPU</div>
          <div class="monitor-item-value">{{item.State.CPU.toFixed(2) + '%'}}</div>
          <a-progress class="monitor-item-progress" :status="progressStatus(item.State.CPU)" :percent="item.State.CPU/100" :show-text="false" style="width: 60px" />
        </div>
        <div class="mem">
          <div class="monitor-item-title">内存使用情况</div>
          <div class="monitor-item-value">{{(item.State.MemUsed / item.Host.MemTotal * 100).toFixed(2) + '%'}}</div>
          <a-progress class="monitor-item-progress" :status="progressStatus(item.State.MemUsed / item.Host.MemTotal * 100)" :percent="item.State.MemUsed / item.Host.MemTotal" :show-text="false" style="width: 60px" />
        </div>
        <div class="network">
          <div class="monitor-item-title">网络速度（IN|OUT）</div>
          <div class="monitor-item-value">{{`${formatBytes(item.State.NetInSpeed)}/s | ${formatBytes(item.State.NetOutSpeed)}/s`}}</div>
        </div>
        <div class="average">
          <div class="monitor-item-title">负载平均值(1|5|15)</div>
          <div class="monitor-item-value">{{`${item.State.Load1} | ${item.State.Load5} | ${item.State.Load15}`}}</div>
        </div>
        <div class="uptime" style="width: 120px;" v-if="tunnelConfig.enabled">
          <div class="monitor-item-title">在线隧道</div>
          <div class="monitor-item-value">{{(() => {
            if (item.Host.Name.startsWith('Stellar-')) {
              return '-'
            }
            const apiNodeName = nodeNameMap[item.Host.Name] || item.Host.Name
            const nodeData = simpleData.find(node => node.node === apiNodeName)
            return nodeData?.online || 0
          })()}}</div>
        </div>
        <div class="uptime">
          <div class="monitor-item-title">上报时间</div>
          <div class="monitor-item-value">{{formatTimeStamp(item.TimeStamp)}}</div>
        </div>
        <div class="detail" v-if="selectHost === item.Host.Name">
          <a-row>
            <a-col :span="10" :xs="24" :sm="24" :md="10" :lg="10" :sl="10">
              <div class="detail-item-list">
                <div class="detail-item">
                  <div class="name">主机名</div>
                  <div class="value">{{item.Host.Name}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">地区</div>
                  <div class="value">
                    <span :class="`flag-icon flag-icon-${item.Host.Name.slice(0, 2).replace('UK', 'GB').toLowerCase()}`"></span>
                    {{item.Host.Name.slice(0, 2).toUpperCase()}}
                  </div>
                </div>
                <div class="detail-item">
                  <div class="name">系统</div>
                  <div class="value">{{item.Host.Platform}} {{item.Host.PlatformVersion}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">架构</div>
                  <div class="value">{{item.Host.Arch}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">虚拟化</div>
                  <div class="value">{{item.Host.Virtualization || '-'}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">CPU</div>
                  <div class="value">{{item.Host.CPU.join(',')}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">CPU占用</div>
                  <div class="value">{{item.State.CPU.toFixed(2) + '%'}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">内存使用情况</div>
                  <div class="value">{{(item.State.MemUsed / item.Host.MemTotal * 100).toFixed(2) + '%'}} ({{formatBytes(item.State.MemUsed)}} / {{formatBytes(item.Host.MemTotal)}})</div>
                </div>
                <div class="detail-item">
                  <div class="name">虚拟内存(Swap)</div>
                  <div class="value">{{formatBytes(item.State.SwapUsed)}} / {{formatBytes(item.Host.SwapTotal)}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">网络速度（IN|OUT）</div>
                  <div class="value">{{`${formatBytes(item.State.NetInSpeed)}/s | ${formatBytes(item.State.NetOutSpeed)}/s`}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">负载平均值(1|5|15)</div>
                  <div class="value">{{`${item.State.Load1} | ${item.State.Load5} | ${item.State.Load15}`}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">流量使用↑|↓</div>
                  <div class="value">{{formatBytes(item.State.NetOutTransfer)}} | {{formatBytes(item.State.NetInTransfer)}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">开机时间</div>
                  <div class="value">{{formatTimeStamp(item.Host.BootTime)}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">上报时间</div>
                  <div class="value">{{formatTimeStamp(item.TimeStamp)}}</div>
                </div>
                <div class="detail-item" v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].seller">
                  <div class="name">商家名称</div>
                  <div class="value">{{hostInfo[item.Host.Name].seller}}</div>
                </div>
                <div class="detail-item" v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].price">
                  <div class="name">主机价格</div>
                  <div class="value">{{hostInfo[item.Host.Name].price}}</div>
                </div>
                <div class="detail-item" v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].due_time">
                  <div class="name">到期时间</div>
                  <div class="value">{{moment(hostInfo[item.Host.Name].due_time).format('YYYY-MM-DD')}}</div>
                </div>
                <div class="detail-item" v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].buy_url">
                  <div class="name">购买链接</div>
                  <div class="value">
                    <a style="color: #0077ff" :href="hostInfo[item.Host.Name].buy_url" target="_blank" @click.stop="() => {}">{{hostInfo[item.Host.Name].buy_url}}</a>
                  </div>
                </div>
              </div>
            </a-col>
            <a-col :span="14" :xs="24" :sm="24" :md="14" :lg="14" :sl="14">
              <a-row :gutter="20">
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <CPU ref="cpuRef" style="margin-bottom: 20px;" :data="charts[item.Host.Name].cpu" />
                </a-col>
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <Mem ref="memRef" :max="item.Host.MemTotal" style="margin-bottom: 20px;" :data="charts[item.Host.Name].mem" />
                </a-col>
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <NetIn ref="netInRef" :data="charts[item.Host.Name].net_in" />
                </a-col>
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <NetOut ref="netOutRef" :data="charts[item.Host.Name].net_out" />
                </a-col>
              </a-row>
            </a-col>
          </a-row>
        </div>
        <div class="delete-btn" @click.stop="handleShowDelete(item.Host.Name)">
          <icon-delete />
        </div>
      </div>
    </div>

    <!-- 卡片视图 -->
    <div class="card-view" v-else>
      <div class="card-item" v-for="(item, index) in hosts" :key="index" @click="handleSelectHost(item.Host.Name)">
        <div class="card-header">
          <span class="card-title">
            <span :class="`flag-icon flag-icon-${item.Host.Name.slice(0, 2).replace('UK', 'GB').toLowerCase()}`"></span>
            {{item.Host.Name}}
          </span>
          <span :class="['status-badge', item.status ? 'online' : 'offline']">
            {{item.status ? '在线' : '离线'}}
          </span>
        </div>
        <div class="card-content">
          <div class="stat-row">
            <div class="stat-label">CPU</div>
            <div class="stat-value">
              {{item.State.CPU.toFixed(2) + '%'}}
              <a-progress :status="progressStatus(item.State.CPU)" :percent="item.State.CPU/100" :show-text="false" />
            </div>
          </div>
          <div class="stat-row">
            <div class="stat-label">内存</div>
            <div class="stat-value">
              {{(item.State.MemUsed / item.Host.MemTotal * 100).toFixed(2) + '%'}}
              <a-progress :status="progressStatus(item.State.MemUsed / item.Host.MemTotal * 100)" :percent="item.State.MemUsed / item.Host.MemTotal" :show-text="false" />
            </div>
          </div>
          <div class="stat-row">
            <div class="stat-label">负载平均值(1|5|15)</div>
            <div class="stat-value">{{`${item.State.Load1} | ${item.State.Load5} | ${item.State.Load15}`}}</div>
          </div>
          <div class="stat-row">
            <div class="stat-label">网络速度</div>
            <div class="stat-value">
              <div class="network-speed">
                <span class="speed-item">
                  <icon-up-circle style="margin-right: 4px;" /> {{formatBytes(item.State.NetOutSpeed)}}/s
                </span>
                <span class="speed-item">
                  <icon-down-circle style="margin-right: 4px;" /> {{formatBytes(item.State.NetInSpeed)}}/s
                </span>
              </div>
            </div>
          </div>
          <div class="stat-row" v-if="tunnelConfig.enabled">
            <div class="stat-label">在线隧道</div>
            <div class="stat-value">{{(() => {
              if (item.Host.Name.startsWith('Stellar-')) {
                return '-'
              }
              const apiNodeName = nodeNameMap[item.Host.Name] || item.Host.Name
              const nodeData = simpleData.find(node => node.node === apiNodeName)
              return nodeData?.online || 0
            })()}}</div>
          </div>
        </div>
        <div class="card-actions">
          <a-button size="small" status="danger" @click.stop="handleShowDelete(item.Host.Name)">
            <template #icon><icon-delete /></template>
          </a-button>
        </div>
      </div>
    </div>

    <a-modal v-model:visible="deleteVisible" :footer="false" :hide-title="true" width="360px">
      <div class="akile-modal-title">
        <span>删除主机</span>
        <a-button @click="handleClose">
          <template #icon>
            <icon-close/>
          </template>
        </a-button>
      </div>
      <div class="akile-modal-content">
        <a-input-password v-model="authSecret" placeholder="请输入管理密钥"></a-input-password>
        <div class="tips">提示：删除后无法恢复，请确定后再删除操作</div>
      </div>
      <div class="akile-modal-action">
        <a-button type="primary" status="danger" :long="true" @click="handleDeleteHost">确认删除</a-button>
      </div>
    </a-modal>
    <div class="footer" style="margin-top: 30px"><a href="https://www.stellarfrp.top">官网</a></div>
    <div class="footer" style="margin-bottom: 10px">Copyright  2024-{{new Date().getFullYear()}} StellarFRP.</div>
    <div class="footer" style="margin-bottom: 10px">本站已运行：{{runningTime}}</div>
    <div class="footer" style="margin-bottom: 30px">Powered by <a href="https://github.com/akile-network/akile_monitor" target="_blank" style="color: #005fe7;">Akile Monitor</a></div>

    <!-- 卡片视图的详细信息模态框 -->
    <a-modal v-model:visible="detailVisible" :footer="false" :hide-title="true" width="1200px">
      <div class="akile-modal-title">
        <span v-if="selectedHost">
          <span :class="`flag-icon flag-icon-${selectedHost.Host.Name.slice(0, 2).replace('UK', 'GB').toLowerCase()}`"></span>
          {{selectedHost.Host.Name}}
          <span :class="['status-badge', selectedHost.status ? 'online' : 'offline']">
            {{selectedHost.status ? '在线' : '离线'}}
          </span>
        </span>
        <a-button @click="detailVisible = false">
          <template #icon>
            <icon-close/>
          </template>
        </a-button>
      </div>
      <div class="akile-modal-content" v-if="selectedHost">
        <a-row :gutter="24">
          <a-col :span="8">
            <div class="detail-item-list">
              <div class="detail-item">
                <div class="name">系统版本</div>
                <div class="value">{{selectedHost.Host.Platform}} {{selectedHost.Host.PlatformVersion}}</div>
              </div>
              <div class="detail-item">
                <div class="name">CPU核心数</div>
                <div class="value">{{selectedHost.Host.CPU[0].ModelName}}</div>
              </div>
              <div class="detail-item">
                <div class="name">内存大小</div>
                <div class="value">{{formatBytes(selectedHost.Host.MemTotal)}}</div>
              </div>
              <div class="detail-item">
                <div class="name">���统盘使用率</div>
                <div class="value">{{(selectedHost.State.DiskUsed / selectedHost.State.DiskTotal * 100).toFixed(2)}}%</div>
              </div>
              <div class="detail-item">
                <div class="name">虚拟内存</div>
                <div class="value">{{formatBytes(selectedHost.Host.SwapTotal)}}</div>
              </div>
              <div class="detail-item">
                <div class="name">总流量</div>
                <div class="value">
                  <div class="network-stats">
                    <span><icon-up-circle /> {{formatBytes(selectedHost.State.NetOutTransfer)}}</span>
                    <span><icon-down-circle /> {{formatBytes(selectedHost.State.NetInTransfer)}}</span>
                  </div>
                </div>
              </div>
              <div class="detail-item">
                <div class="name">开机时间</div>
                <div class="value">{{formatTimeStamp(selectedHost.Host.BootTime)}}</div>
              </div>
              <div class="detail-item">
                <div class="name">上报时间</div>
                <div class="value">{{formatTimeStamp(selectedHost.TimeStamp)}}</div>
              </div>
              <div class="detail-item" v-if="hostInfo[selectedHost.Host.Name] && hostInfo[selectedHost.Host.Name].seller">
                <div class="name">商家名称</div>
                <div class="value">{{hostInfo[selectedHost.Host.Name].seller}}</div>
              </div>
              <div class="detail-item" v-if="hostInfo[selectedHost.Host.Name] && hostInfo[selectedHost.Host.Name].price">
                <div class="name">主机价格</div>
                <div class="value">{{hostInfo[selectedHost.Host.Name].price}}</div>
              </div>
              <div class="detail-item" v-if="hostInfo[selectedHost.Host.Name] && hostInfo[selectedHost.Host.Name].due_time">
                <div class="name">到期时间</div>
                <div class="value">{{moment(hostInfo[selectedHost.Host.Name].due_time).format('YYYY-MM-DD')}}</div>
              </div>
              <div class="detail-item" v-if="hostInfo[selectedHost.Host.Name] && hostInfo[selectedHost.Host.Name].buy_url">
                <div class="name">购买链接</div>
                <div class="value">
                  <a style="color: #0077ff" :href="hostInfo[selectedHost.Host.Name].buy_url" target="_blank" @click.stop="() => {}">{{hostInfo[selectedHost.Host.Name].buy_url}}</a>
                </div>
              </div>
            </div>
          </a-col>
          <a-col :span="16">
            <div class="charts-section">
              <a-row :gutter="[24, 24]">
                <a-col :span="12">
                  <div class="chart-container">
                    <div class="chart-title">CPU使用率</div>
                    <CPU ref="cpuRef" :data="charts[selectedHost.Host.Name].cpu" />
                  </div>
                </a-col>
                <a-col :span="12">
                  <div class="chart-container">
                    <div class="chart-title">内存使用率</div>
                    <Mem ref="memRef" :max="selectedHost.Host.MemTotal" :data="charts[selectedHost.Host.Name].mem" />
                  </div>
                </a-col>
                <a-col :span="12">
                  <div class="chart-container">
                    <div class="chart-title">网络下行速度</div>
                    <NetIn ref="netInRef" :data="charts[selectedHost.Host.Name].net_in" />
                  </div>
                </a-col>
                <a-col :span="12">
                  <div class="chart-container">
                    <div class="chart-title">网络上行速度</div>
                    <NetOut ref="netOutRef" :data="charts[selectedHost.Host.Name].net_out" />
                  </div>
                </a-col>
              </a-row>
            </div>
          </a-col>
        </a-row>
      </div>
    </a-modal>
  </div>
</template>

<style lang="scss">
body {
  margin: 0;
  background-color: #fafafa;
  font-family: Inter,-apple-system,BlinkMacSystemFont,Roboto,PingFang SC,Noto Sans CJK,WenQuanYi Micro Hei,Microsoft YaHei;
}

a {
  text-decoration: none;
}

.max-container {
  margin: 0 auto;
  width: 100%;
  max-width: 1440px;
}

.header {
  margin: 10px;
  display: flex;
  align-items: center;
  justify-content: space-between;

  .theme-btn {
    border: 1px solid #eeeeee!important;
    background-color: #ffffff!important;
    color: #333333!important;
  }
}

.area-tabs {
  margin: 20px 10px;

  .area-tab-item {
    margin-bottom: 10px;
    margin-right: 10px;
    padding: 6px 12px;
    border-radius: 6px;
    cursor: pointer;
    border: 1px solid #e5e5e5;
    background: #ffffff;
    box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
    display: inline-block;

    .flag-icon {
      border-radius: 3px;
      margin-top: -3px;
    }

    &.is-active {
      background: #005fe710;
      color: #054db4;
      border: 1px solid #005fe7;
    }
  }

}

.monitor-card {
  position: relative;
  margin: 0 auto;
  padding: 10px;

  .monitor-item {
    position: relative;
    margin-bottom: 12px;
    padding: 12px 24px;
    border-radius: 6px;
    border: 1px solid #e5e5e5;
    display: block;
    background: #ffffff;
    box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
    cursor: pointer;

    &.is-active {
      background: #e7e7e730;

      .delete-btn,
      .edit-btn {
        display: none!important;
      }

      &>.detail {
        margin-top: 15px;
        border-top: 1px solid #eeeeee;
        padding-top: 15px;
        display: block;
      }
    }

    &:hover {
      background: #e7e7e730;

      .delete-btn,
      .edit-btn {
        display: flex;
      }
    }

    .edit-btn {
      right: 60px!important;
      background: rgba(22, 131, 255, 0.13)!important;

      &:hover {
        background: rgba(22, 131, 255, 0.19)!important;
      }

      .arco-icon {
        color: #1673ff!important;
      }
    }

    .delete-btn,
    .edit-btn {
      position: absolute;
      right: 20px;
      top: calc(50% - 16px);
      cursor: pointer;
      background: #ff161620;
      border-radius: 100px;
      width: 32px;
      height: 32px;
      display: none;
      align-items: center;
      justify-content: center;
      transition: .15s background-color ease-in-out;

      &:hover {
        background: #ff161630;
      }

      .arco-icon {
        color: #ff1616;
      }
    }

    .flag-icon {
      margin-right: 5px;
      border-radius: 3px;
    }

    .monitor-item-title {
      margin-bottom: 3px;
      font-size: 11px;
      opacity: .6;
    }

    .monitor-item-value {
      font-weight: 500;
    }

    .monitor-item-progress {
      margin-top: 4px;
      height: 4px;
      display: block;
    }

    .detail {
      width: 100%;
    }

    .name {
      display: inline-block;
      vertical-align: middle;
      width: 250px;

      .title {
        margin-bottom: 5px;
        display: flex;
        align-items: center;
        font-weight: 600;
        font-size: 16px;
      }

      .status {
        display: flex;
        align-items: center;
        &::before {
          margin-right: 10px;
          position: relative;
          display: block;
          content: '';
          width: 8px;
          height: 8px;
          border-radius: 12px;
          background-color: #fb2c36;
        }

        &.online {
          &::before {
            background-color: #00c951;
          }
        }

        span {
          font-size: 13px;
          opacity: .6;
        }
      }
    }

    .platform {
      display: inline-block;
      vertical-align: top;
      width: 120px;
    }

    .cpu {
      display: inline-block;
      vertical-align: top;
      width: 120px;
    }

    .mem {
      display: inline-block;
      vertical-align: top;
      width: 120px;
    }

    .average {
      display: inline-block;
      vertical-align: top;
      width: 200px;
    }

    .network {
      display: inline-block;
      vertical-align: top;
      width: 200px;
    }

    .uptime {
      display: inline-block;
      vertical-align: middle;
      width: 200px;
    }

    .detail {
      display: none;

      .detail-item-list {
        margin-bottom: 20px;
      }

      .detail-item {
        .name {
          width: 20%;
          font-size: 12px;
          color: #666;
          margin-bottom: 5px;
          display: inline-block;
          vertical-align: top;
        }

        .value {
          width: 80%;
          font-size: 12px;
          font-weight: 500;
          display: inline-block;
          vertical-align: top;
        }
      }
    }
  }
}

.logo {
  margin-top: 20px;
  margin-bottom: 20px;
  position: relative;
  cursor: pointer;
  line-height: 54px;
  height: 54px;
  font-size: 16px;
  font-weight: 900;
  color: #333;
  display: flex;
  align-items: center;

  .logo-icon {
    margin-right: 5px;
    height: 28px;
    width: 28px;
  }
}

.monitor-action-bar {
  margin: 0 10px;
  display: inline-block;
  border: 1px solid #e5e5e5;
  background: #ffffff;
  box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
  border-radius: 4px;

  :deep(.arco-tabs-content) {
    display: none;
  }
}

.footer {
  line-height: 22px;
  text-align: center;
  color: #565656;

  a {
    color: rgba(var(--primary-6));
  }
}

.akile-modal-title {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px;
  border-bottom: 1px solid var(--color-border);
  
  .flag-icon {
    margin-right: 8px;
    border-radius: 3px;
  }
  
  .status-badge {
    margin-left: 8px;
    padding: 2px 8px;
    border-radius: 12px;
    font-size: 12px;
    
    &.online {
      background: #e6f7ed;
      color: #00c951;
    }
    
    &.offline {
      background: #ffece8;
      color: #ff4d4f;
    }
  }
}

.akile-modal-content {
  padding: 20px;
  
  .detail-item-list {
    background: var(--color-bg-2);
    border-radius: 8px;
    padding: 20px;
    height: 100%;
    
    .detail-item {
      margin-bottom: 16px;
      
      &:last-child {
        margin-bottom: 0;
      }
      
      .name {
        font-size: 13px;
        color: var(--color-text-3);
        margin-bottom: 4px;
      }
      
      .value {
        font-size: 14px;
        color: var(--color-text-1);
        font-weight: 500;
        word-break: break-all;
      }
    }
  }
  
  .charts-section {
    height: 100%;
    
    .chart-container {
      background: var(--color-bg-2);
      border-radius: 8px;
      padding: 20px;
      height: 100%;
      margin-bottom: 24px;
      
      &:last-child {
        margin-bottom: 0;
      }
      
      .chart-title {
        font-size: 14px;
        color: var(--color-text-2);
        margin-bottom: 12px;
        font-weight: 500;
      }
    }
  }
}

body[arco-theme='dark'] {
  background-color: #111111;

  .arco-modal {
    background-color: #0e0e0e;
    border: 1px solid rgba(255,255,255,0.05);
  }

  .header {
    .logo {
      span,
      small {
        color: #ffffff;
      }
    }

    .theme-btn {
      border: 1px solid #333333!important;
      background-color: #000000!important;
      color: #ffffff!important;
    }
  }

  .area-tabs {
    .area-tab-item {
      border-color: #333333;
      background: #000000;
      color: #ffffff;
      box-shadow: none;

      &.is-active {
        background: #005fe705;
        color: #005fe7;
        border: 1px solid #005fe7;
      }
    }
  }

  .footer {
    color: #ffffffAA;
  }

  .monitor-card {
    .monitor-item {
      border: 1px solid rgb(255 255 255 / 16%);
      box-shadow: none;
      background-color: #000000;
      color: #ffffff;

      &:hover {
        background-color: #101010;
      }

      .detail {
        border-color: #333333AA;
        .detail-item-list {
          .detail-item {
            .name {
              color: #999999;
            }
          }
        }
      }
    }
  }

}

@media screen and (max-width: 768px) {
  .monitor-item {
    &>div {
      margin-bottom: 10px;
    }
  }

  .detail {
    .detail-item {
      .name {
        width: 25%!important;
      }
      .value {
        width: 75%!important;
      }
    }
  }
}

@media screen and (max-width: 1920px) {
  .max-container {
    max-width: 1440px;
  }
}

@media screen and (max-width: 1280px) {
  .max-container {
    max-width: 1140px;
  }
}

@media screen and (max-width: 992px) {
  .max-container {
    max-width: 960px;
  }
}

@media screen and (max-width: 768px) {
  .max-container {
    max-width: 720px;
  }
}

@media screen and (max-width: 576px) {
  .max-container {
    max-width: 540px;
  }
}

.header-actions {
  display: flex;
  gap: 10px;
  align-items: center;
}

.view-btn {
  border: 1px solid #eeeeee!important;
  background-color: #ffffff!important;
  color: #333333!important;
}

.simple-view {
  padding: 20px;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
}

.simple-card {
  padding: 15px;
  border-radius: 8px;
  background: #ffffff;
  border: 1px solid #e5e5e5;
  box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);

  .node-header {
    display: flex;
    align-items: center;
    margin-bottom: 15px;
    gap: 10px;

    .node-name {
      font-size: 16px;
      font-weight: 600;
      flex: 1;
    }

    .node-status {
      padding: 2px 8px;
      border-radius: 12px;
      font-size: 12px;
      
      &.online {
        background: #e6f7ed;
        color: #00c951;
      }
      
      &.offline {
        background: #ffece8;
        color: #ff4d4f;
      }
    }

    .node-type {
      padding: 2px 8px;
      border-radius: 12px;
      font-size: 12px;
      background: #f0f0f0;
      color: #666;
    }
  }

  .node-stats {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 12px;

    .stat-item {
      .label {
        font-size: 12px;
        color: #666;
        margin-bottom: 4px;
      }

      .value {
        font-size: 14px;
        font-weight: 500;
      }
    }
  }
}

body[arco-theme='dark'] {
  .simple-card {
    background: #000000;
    border-color: #333333;
    
    .node-header {
      .node-type {
        background: #333333;
        color: #999;
      }
    }
    
    .node-stats {
      .stat-item {
        .label {
          color: #999;
        }
        .value {
          color: #fff;
        }
      }
    }
  }

  .view-btn {
    border: 1px solid #333333!important;
    background-color: #000000!important;
    color: #ffffff!important;
  }
}

// 修改卡片视图的样式
.card-view {
  padding: 20px;
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 20px;
}

// 调整响应式布局断点
@media screen and (max-width: 1600px) {
  .card-view {
    grid-template-columns: repeat(3, 1fr);
  }
}

@media screen and (max-width: 1200px) {
  .card-view {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media screen and (max-width: 768px) {
  .card-view {
    grid-template-columns: 1fr;
  }
}

.card-item {
  background: #ffffff;
  border: 1px solid #e5e5e5;
  border-radius: 8px;
  padding: 16px;
  box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
  cursor: pointer;
  transition: all 0.3s ease;

  &:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  }

  .card-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 16px;

    .card-title {
      font-size: 16px;
      font-weight: 600;
      display: flex;
      align-items: center;
      gap: 8px;

      .flag-icon {
        border-radius: 3px;
      }
    }

    .status-badge {
      padding: 4px 8px;
      border-radius: 12px;
      font-size: 12px;
      
      &.online {
        background: #e6f7ed;
        color: #00c951;
      }
      
      &.offline {
        background: #ffece8;
        color: #ff4d4f;
      }
    }
  }

  .card-content {
    .stat-row {
      margin-bottom: 12px;

      .stat-label {
        font-size: 12px;
        color: #666;
        margin-bottom: 4px;
      }

      .stat-value {
        font-size: 14px;
        font-weight: 500;

        .arco-progress {
          margin-top: 4px;
        }
      }
    }
  }

  .card-actions {
    margin-top: 16px;
    display: flex;
    justify-content: flex-end;
    gap: 8px;
  }
}

// 暗色主题支持
body[arco-theme='dark'] {
  .card-item {
    background: #000000;
    border-color: #333333;

    .card-content {
      .stat-row {
        .stat-label {
          color: #999;
        }
        .stat-value {
          color: #fff;
        }
      }
    }
  }
}

.network-speed {
  display: flex;
  flex-direction: column;
  gap: 4px;
  
  .speed-item {
    font-size: 13px;
    display: flex;
    align-items: center;

    .arco-icon {
      font-size: 14px;
    }
  }
}

.sort-buttons {
  margin-right: 8px;
  
  .sort-btn {
    border: 1px solid #eeeeee;
    background-color: #ffffff;
    color: #333333;
  }
}

// 暗色主题支持
body[arco-theme='dark'] {
  .sort-btn {
    border: 1px solid #333333!important;
    background-color: #000000!important;
    color: #ffffff!important;
  }
}

.host-detail-modal {
  .modal-title {
    display: flex;
    align-items: center;
    gap: 8px;
    
    .flag-icon {
      border-radius: 3px;
    }
    
    .status-badge {
      margin-left: auto;
    }
  }

  .detail-section {
    background: var(--color-bg-2);
    border-radius: 8px;
    padding: 20px;
    height: 100%;
  }

  .detail-item-list {
    .detail-item {
      margin-bottom: 16px;
      
      &:last-child {
        margin-bottom: 0;
      }
      
      .name {
        font-size: 13px;
        color: var(--color-text-3);
        margin-bottom: 4px;
      }
      
      .value {
        font-size: 14px;
        color: var(--color-text-1);
        font-weight: 500;
      }
    }
  }

  .charts-section {
    .chart-container {
      background: var(--color-bg-2);
      border-radius: 8px;
      padding: 16px;
      height: 100%;

      .chart-title {
        font-size: 14px;
        color: var(--color-text-2);
        margin-bottom: 12px;
        font-weight: 500;
      }
    }
  }

  .network-stats {
    display: flex;
    gap: 16px;

    span {
      display: flex;
      align-items: center;
      gap: 4px;
    }
  }
}

// 修改模态框全局样式
:deep(.arco-modal-wrapper) {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

:deep(.arco-modal) {
  position: relative;
  margin: 0 auto;
  max-height: 90vh;
  overflow-y: auto;
  z-index: 1001;
}

:deep(.arco-modal-mask) {
  position: fixed;
  inset: 0;
  background-color: rgba(0, 0, 0, 0.6);
  z-index: 1000;
}

</style>
