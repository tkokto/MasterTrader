<template>
  <div class="h-screen w-screen relative bg-black overflow-hidden">
    <!-- 全屏K线图 -->
    <div class="absolute inset-0 w-full h-full">
      <KLineChart 
        ref="chartRef"
        :symbol="currentSymbol"
        @stock-changed="handleStockChange"
        @price-changed="handlePriceChange"
      />
    </div>
    
    <!-- 浮动交易面板 -->
    <div 
      ref="panelRef"
      class="fixed z-50 select-none m-3"
      :style="{ 
        left: panelPosition.x + 'px', 
        top: panelPosition.y + 'px',
        width: isCollapsed ? '240px' : '380px',
        color: 'black'
      }"
    >
      <div class="backdrop-blur-xl bg-black/80 border border-white/20 rounded-2xl shadow-2xl overflow-hidden transition-all duration-300" style="backdrop-filter: blur(20px); -webkit-backdrop-filter: blur(20px);">
        <!-- 面板头部 -->
        <div class="flex items-center justify-between p-4 bg-white/10 cursor-move" @mousedown="startDrag">
          <div class="flex items-center space-x-2">
            <div class="w-3 h-3 bg-red-500 rounded-full"></div>
            <div class="w-3 h-3 bg-yellow-500 rounded-full"></div>
            <div class="w-3 h-3 bg-green-500 rounded-full"></div>
          </div>
          <div class="text-white font-medium text-sm">{{ currentSymbol }} 交易面板</div>
          <button 
            @click="toggleCollapse"
            class="text-white hover:text-gray-300 transition-colors"
          >
            <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path 
                stroke-linecap="round" 
                stroke-linejoin="round" 
                stroke-width="2" 
                :d="isCollapsed ? 'M19 9l-7 7-7-7' : 'M5 15l7-7 7 7'"
              />
            </svg>
          </button>
        </div>

        <!-- 展开模式内容 -->
        <div v-if="!isCollapsed" class="transition-all duration-300">
          <div class="p-8 space-y-8">
            <!-- 资金信息 -->
            <div>
              <h3 class="text-white font-semibold mb-4 text-base">账户信息</h3>
              <div class="space-y-4">
                <div class="flex justify-between py-2">
                  <span class="text-white/70">总资金:</span>
                  <span class="text-white font-medium text-lg">${{ totalFunds.toLocaleString() }}</span>
                </div>
                <div class="flex justify-between py-2">
                  <span class="text-white/70">可用资金:</span>
                  <span class="text-white font-medium text-lg">${{ availableFunds.toLocaleString() }}</span>
                </div>
                <div class="flex justify-between py-2">
                  <span class="text-white/70">持仓市值:</span>
                  <span class="text-white font-medium text-lg">${{ positionValue.toLocaleString() }}</span>
                </div>
              </div>
            </div>

            <!-- 持仓信息 -->
            <div v-if="position > 0">
              <h3 class="text-white font-semibold mb-4 text-base">持仓信息</h3>
              <div class="space-y-4">
                <div class="flex justify-between py-2">
                  <span class="text-white/70">持仓数量:</span>
                  <span class="text-white font-medium text-lg">{{ position }}</span>
                </div>
                <div class="flex justify-between py-2">
                  <span class="text-white/70">成本价:</span>
                  <span class="text-white font-medium text-lg">${{ averageCost.toFixed(2) }}</span>
                </div>
                <div class="flex justify-between py-2">
                  <span class="text-white/70">当前价:</span>
                  <span class="text-white font-medium text-lg">${{ currentPrice.toFixed(2) }}</span>
                </div>
                <div class="flex justify-between py-2">
                  <span class="text-white/70">盈亏:</span>
                  <span :class="pnl >= 0 ? 'text-green-400' : 'text-red-400'" class="font-bold text-xl">
                    {{ pnl >= 0 ? '+' : '' }}${{ pnl.toLocaleString() }}
                  </span>
                </div>
              </div>
            </div>

            <!-- 交易操作 -->
            <div>
              <h3 class="text-white font-semibold mb-4 text-base">交易操作</h3>
              
              <!-- 买入比例 -->
              <div class="mb-6">
                <div class="grid grid-cols-3 gap-3 mb-4">
                  <el-button 
                    size="default"
                    :type="buyRatio === 0.25 ? 'primary' : ''"
                    @click="setBuyRatio(0.25)"
                    class="h-10"
                  >
                    1/4
                  </el-button>
                  <el-button 
                    size="default"
                    :type="buyRatio === 0.33 ? 'primary' : ''"
                    @click="setBuyRatio(0.33)"
                    class="h-10"
                  >
                    1/3
                  </el-button>
                  <el-button 
                    size="default"
                    :type="buyRatio === 0.5 ? 'primary' : ''"
                    @click="setBuyRatio(0.5)"
                    class="h-10"
                  >
                    1/2
                  </el-button>
                </div>
                <el-button 
                  type="danger" 
                  size="large"
                  class="w-full h-12" 
                  @click="handleBuy"
                  :disabled="availableFunds <= 0"
                >
                  买入 (${{ (availableFunds * buyRatio).toLocaleString() }})
                </el-button>
              </div>

              <!-- 卖出 -->
              <el-button 
                type="success" 
                size="large"
                class="w-full h-12" 
                @click="handleSell"
                :disabled="position <= 0"
              >
                全部卖出
              </el-button>
            </div>
          </div>
        </div>

        <!-- 收缩模式 - 只显示核心信息 -->
        <div v-if="isCollapsed" class="p-6">
          <!-- 持仓盈亏 (仅在有持仓时显示) -->
          <div v-if="position > 0" class="text-center mb-6">
            <div class="text-white/60 text-sm mb-2">持仓盈亏</div>
            <div :class="pnl >= 0 ? 'text-green-400' : 'text-red-400'" class="font-bold text-2xl">
              {{ pnl >= 0 ? '+' : '' }}${{ Math.abs(pnl).toLocaleString() }}
            </div>
            <div class="text-white/50 text-sm mt-2">
              {{ position }} 股 @ ${{ currentPrice.toFixed(2) }}
            </div>
          </div>
          
          <!-- 快速交易按钮 -->
          <div class="space-y-3">
            <el-button 
              type="danger" 
              size="large"
              class="w-full h-12"
              @click="handleBuy"
              :disabled="availableFunds <= 0"
            >
              买入 ${{ (availableFunds * buyRatio).toLocaleString() }}
            </el-button>
            <el-button 
              type="success" 
              size="large"
              class="w-full h-12"
              @click="handleSell"
              :disabled="position <= 0"
            >
              全部卖出
            </el-button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { ElMessage } from 'element-plus'
import KLineChart from './KLineChart.vue'

// 交易数据
const totalFunds = ref(100000) // 初始资金10万
const availableFunds = ref(100000) // 可用资金
const position = ref(0) // 持仓数量
const averageCost = ref(0) // 平均成本
const buyRatio = ref(0.25) // 买入比例
const currentSymbol = ref('000001.SS') // 当前股票代码，默认为上证指数
const chartRef = ref<InstanceType<typeof KLineChart>>()

// 面板状态
const isCollapsed = ref(false)
const panelRef = ref<HTMLElement>()
const panelPosition = ref({ x: 50, y: 50 })
const isDragging = ref(false)
const dragStart = ref({ x: 0, y: 0 })

// 当前价格 (需要从实时数据获取)
const currentPrice = ref(150) // 默认价格，实际应该从图表获取

// 计算属性
const positionValue = computed(() => {
  return position.value * currentPrice.value
})

const pnl = computed(() => {
  if (position.value > 0) {
    return (currentPrice.value - averageCost.value) * position.value
  }
  return 0
})


// 面板折叠切换
const toggleCollapse = () => {
  isCollapsed.value = !isCollapsed.value
}

// 拖拽功能
const startDrag = (e: MouseEvent) => {
  isDragging.value = true
  dragStart.value = {
    x: e.clientX - panelPosition.value.x,
    y: e.clientY - panelPosition.value.y
  }
  document.addEventListener('mousemove', onDrag)
  document.addEventListener('mouseup', stopDrag)
  e.preventDefault()
}

const onDrag = (e: MouseEvent) => {
  if (!isDragging.value) return
  
  const newX = e.clientX - dragStart.value.x
  const newY = e.clientY - dragStart.value.y
  
  // 限制拖拽范围在屏幕内
  const maxX = window.innerWidth - (isCollapsed.value ? 240 : 380)
  const maxY = window.innerHeight - 200
  
  panelPosition.value = {
    x: Math.max(0, Math.min(newX, maxX)),
    y: Math.max(0, Math.min(newY, maxY))
  }
}

const stopDrag = () => {
  isDragging.value = false
  document.removeEventListener('mousemove', onDrag)
  document.removeEventListener('mouseup', stopDrag)
}


// 设置买入比例
const setBuyRatio = (ratio: number) => {
  buyRatio.value = ratio
}

// 买入操作
const handleBuy = () => {
  if (availableFunds.value <= 0) return
  
  const buyAmount = availableFunds.value * buyRatio.value
  const buyShares = Math.floor(buyAmount / currentPrice.value)
  
  if (buyShares > 0) {
    const actualCost = buyShares * currentPrice.value
    
    // 更新平均成本
    const totalCost = (position.value * averageCost.value) + actualCost
    const totalShares = position.value + buyShares
    averageCost.value = totalCost / totalShares
    
    // 更新持仓和资金
    position.value += buyShares
    availableFunds.value -= actualCost
    
    ElMessage.success(`买入成功！数量: ${buyShares}, 价格: $${currentPrice.value.toFixed(2)}`)
  }
}

// 卖出操作
const handleSell = () => {
  if (position.value <= 0) return
  
  const sellAmount = position.value * currentPrice.value
  availableFunds.value += sellAmount
  
  ElMessage.success(`卖出成功！数量: ${position.value}, 价格: $${currentPrice.value.toFixed(2)}, 盈亏: ${pnl.value >= 0 ? '+' : ''}$${pnl.value.toFixed(2)}`)
  
  position.value = 0
  averageCost.value = 0
}

// 监听股票变化事件
const handleStockChange = (symbol: string, name: string) => {
  currentSymbol.value = `${symbol} (${name})`
  // 重置交易数据
  position.value = 0
  averageCost.value = 0
  availableFunds.value = totalFunds.value
}

// 监听价格变化事件
const handlePriceChange = (price: number) => {
  currentPrice.value = price
  console.log(`交易面板接收到价格更新: ${price}`)
}

// 组件挂载时初始化面板位置
onMounted(() => {
  // 设置面板初始位置在右上角
  panelPosition.value = {
    x: window.innerWidth - 400,
    y: 20
  }
})

// 清理事件监听
onUnmounted(() => {
  document.removeEventListener('mousemove', onDrag)
  document.removeEventListener('mouseup', stopDrag)
})
</script>