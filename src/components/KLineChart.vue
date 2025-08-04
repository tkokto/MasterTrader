<template>
  <div class="h-full w-full" style="height: 100vh;">
    <div class="controls mb-4 p-4 bg-gray-100 rounded-lg">
      <div class="flex gap-4 items-center">
        <button 
          @click="startRandomMode" 
          class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600"
          :disabled="isRandomMode"
        >
          开始随机模式
        </button>
        <button 
          @click="nextKLine" 
          class="px-4 py-2 bg-green-500 text-white rounded hover:bg-green-600"
          :disabled="!isRandomMode || currentIndex >= maxIndex"
        >
          下一根K线 ({{ currentIndex + 1 }}/{{ maxIndex + 1 }})
        </button>
        <button 
          @click="resetChart" 
          class="px-4 py-2 bg-gray-500 text-white rounded hover:bg-gray-600"
        >
          重置图表
        </button>
        <span class="text-sm text-gray-600">
          当前: {{ props.symbol }} | 随机起始日期: {{ randomStartDate }}
        </span>
      </div>
    </div>
    <div ref="chartContainer" class="flex-1" style="height: calc(100vh - 120px);"></div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, nextTick } from 'vue'
import { KLineChartPro, DefaultDatafeed } from '@klinecharts/pro'
import '@klinecharts/pro/dist/klinecharts-pro.css'

interface Props {
  symbol?: string
}

const props = withDefaults(defineProps<Props>(), {
  symbol: 'AAPL'
})

const chartContainer = ref<HTMLDivElement>()
let chart: any = null
let originalDatafeed: DefaultDatafeed | null = null
let fullHistoryData: any[] = []

// Random mode state
const isRandomMode = ref(false)
const currentIndex = ref(0)
const maxIndex = ref(0)
const randomStartDate = ref('')

// Polygon.io API key
const POLYGON_API_KEY = 'oULCENojkPnOxPvZ6P7C4Up8bum4F_fD'

// 自定义数据源类
class CustomDatafeed {
  private data: any[] = []
  private currentDataIndex = 0
  private chart: any = null
  
  constructor(data: any[], startIndex: number = 0) {
    this.data = data
    this.currentDataIndex = startIndex
  }
  
  setChart(chart: any) {
    this.chart = chart
  }
  
  async searchSymbols(userInput: string) {
    return [
      {
        exchange: 'XNYS',
        fullName: props.symbol,
        market: 'stocks',
        shortName: props.symbol,
        ticker: props.symbol,
        priceCurrency: 'usd'
      }
    ]
  }
  
  async getHistoryKLineData(symbol: any, period: any, from: number, to: number) {
    // 返回当前索引之前的数据
    const dataToReturn = this.data.slice(0, this.currentDataIndex + 1)
    
    return dataToReturn.map((item: any) => ({
      timestamp: item.timestamp,
      open: item.open,
      high: item.high,
      low: item.low,
      close: item.close,
      volume: item.volume
    }))
  }
  
  async subscribe(params: any, callback: Function) {
    // 不需要实时数据
  }
  
  async unsubscribe(params: any) {
    // 取消订阅
  }
  
  // 添加下一根K线数据
  addNextKLine() {
    if (this.currentDataIndex < this.data.length - 1) {
      this.currentDataIndex++
      
      // 获取新的K线数据
      const newKLine = this.data[this.currentDataIndex]
      if (newKLine && this.chart) {
        // 尝试通过图表API添加新数据
        try {
          this.chart.applyNewData?.([{
            timestamp: newKLine.timestamp,
            open: newKLine.open,
            high: newKLine.high,
            low: newKLine.low,
            close: newKLine.close,
            volume: newKLine.volume
          }])
        } catch (error) {
          console.warn('Failed to apply new data, trying refresh:', error)
          // 如果添加新数据失败，尝试刷新整个图表
          this.chart.refresh?.()
        }
      }
      
      return true
    }
    return false
  }
  
  getCurrentIndex() {
    return this.currentDataIndex
  }
  
  getMaxIndex() {
    return this.data.length - 1
  }
}

const initChart = async () => {
  if (!chartContainer.value) return
  
  await nextTick()
  
  try {
    // 创建原始数据源
    originalDatafeed = new DefaultDatafeed(POLYGON_API_KEY)
    
    // 按照官方文档的简单方式创建图表
    chart = new KLineChartPro({
      container: chartContainer.value,
      symbol: {
        exchange: 'XNYS',
        market: 'stocks',
        shortName: props.symbol,
        ticker: props.symbol,
        priceCurrency: 'usd'
      },
      period: { multiplier: 1, timespan: 'day', text: '1D' }, // 使用日K线更适合随机模式
      datafeed: originalDatafeed
    })
    
    console.log('KLineChart Pro initialized successfully')
  } catch (error) {
    console.error('Failed to initialize KLineChart Pro:', error)
  }
}

// 获取历史数据的辅助函数
const fetchHistoryData = async (): Promise<any[]> => {
  if (!originalDatafeed) return []
  
  try {
    const endDate = new Date()
    const startDate = new Date()
    startDate.setFullYear(endDate.getFullYear() - 2) // 获取2年的数据
    
    const historyData = await originalDatafeed.getHistoryKLineData(
      {
        exchange: 'XNYS',
        market: 'stocks',
        shortName: props.symbol,
        ticker: props.symbol,
        priceCurrency: 'usd'
      },
      { multiplier: 1, timespan: 'day', text: '1D' },
      Math.floor(startDate.getTime() / 1000),
      Math.floor(endDate.getTime() / 1000)
    )
    
    return historyData || []
  } catch (error) {
    console.error('Failed to fetch history data:', error)
    return []
  }
}

// 清理图表的辅助函数
const cleanupChart = () => {
  if (chart) {
    try {
      chart.dispose?.()
    } catch (error) {
      console.warn('Chart disposal failed:', error)
    }
    chart = null
  }
  
  // 清空容器内容
  if (chartContainer.value) {
    chartContainer.value.innerHTML = ''
  }
}

// 开始随机模式
const startRandomMode = async () => {
  try {
    // 获取完整历史数据
    fullHistoryData = await fetchHistoryData()
    
    if (fullHistoryData.length < 300) {
      console.warn('历史数据不足，需要至少300天的数据')
      return
    }
    
    // 随机选择起始点，确保至少有200天的后续数据
    const minStartIndex = 50 // 确保前面有一些数据
    const maxStartIndex = fullHistoryData.length - 200 // 确保后面至少有200天
    const randomStartIndex = Math.floor(Math.random() * (maxStartIndex - minStartIndex)) + minStartIndex
    
    // 设置随机模式状态
    isRandomMode.value = true
    currentIndex.value = randomStartIndex
    maxIndex.value = fullHistoryData.length - 1
    
    // 设置随机起始日期显示
    if (fullHistoryData[randomStartIndex]) {
      const date = new Date(fullHistoryData[randomStartIndex].timestamp * 1000)
      randomStartDate.value = date.toLocaleDateString()
    }
    
    // 清理现有图表
    cleanupChart()
    
    // 等待下一个tick确保DOM更新
    await nextTick()
    
    // 创建自定义数据源
    const customDatafeed = new CustomDatafeed(fullHistoryData, randomStartIndex)
    
    // 创建新图表使用自定义数据源
    if (chartContainer.value) {
      chart = new KLineChartPro({
        container: chartContainer.value,
        symbol: {
          exchange: 'XNYS',
          market: 'stocks',
          shortName: props.symbol,
          ticker: props.symbol,
          priceCurrency: 'usd'
        },
        period: { multiplier: 1, timespan: 'day', text: '1D' },
        datafeed: customDatafeed as any
      })
      
      // 将图表实例传递给数据源
      customDatafeed.setChart(chart)
    }
    
    console.log(`随机模式已开始，起始日期: ${randomStartDate.value}`)
  } catch (error) {
    console.error('Failed to start random mode:', error)
    console.error('启动随机模式失败')
  }
}

// 下一根K线
const nextKLine = async () => {
  if (!isRandomMode.value || !chart) return
  
  if (currentIndex.value >= maxIndex.value) {
    console.info('已经到达最新数据')
    return
  }
  
  currentIndex.value++
  
  // 清理现有图表
  cleanupChart()
  
  // 等待下一个tick确保DOM更新
  await nextTick()
  
  // 创建新的自定义数据源，包含到当前索引的所有数据
  const newCustomDatafeed = new CustomDatafeed(fullHistoryData, currentIndex.value)
  
  // 重新创建图表以显示新数据
  try {
    if (chartContainer.value) {
      chart = new KLineChartPro({
        container: chartContainer.value,
        symbol: {
          exchange: 'XNYS',
          market: 'stocks',
          shortName: props.symbol,
          ticker: props.symbol,
          priceCurrency: 'usd'
        },
        period: { multiplier: 1, timespan: 'day', text: '1D' },
        datafeed: newCustomDatafeed as any
      })
      
      newCustomDatafeed.setChart(chart)
    }
  } catch (error) {
    console.error('Failed to update chart:', error)
  }
  
  console.log(`已更新到第 ${currentIndex.value + 1} 根K线`)
}

// 重置图表
const resetChart = () => {
  isRandomMode.value = false
  currentIndex.value = 0
  maxIndex.value = 0
  randomStartDate.value = ''
  fullHistoryData = []
  
  // 清理现有图表
  cleanupChart()
  
  // 重新初始化图表
  initChart()
  
  console.log('图表已重置')
}

onMounted(() => {
  initChart()
})

onUnmounted(() => {
  cleanupChart()
})

// 暴露方法给父组件
defineExpose({
  changeSymbol: (newSymbol: string) => {
    if (chart) {
      chart.setSymbol({
        exchange: 'XNYS',
        market: 'stocks',
        shortName: newSymbol,
        ticker: newSymbol,
        priceCurrency: 'usd'
      })
    }
  },
  getCurrentPrice: () => {
    return 0
  }
})
</script>

<style scoped>
</style>