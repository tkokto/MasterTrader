<template>
  <div class="h-full w-full" style="height: 100vh;">
    <div class="controls mb-4 p-4 bg-gray-100 rounded-lg">
      <div class="flex gap-4 items-center flex-wrap">
        <button 
          @click="startRandomMode" 
          class="px-4 py-2 text-white rounded transition-colors"
          :class="isRandomMode ? 'bg-gray-400 cursor-not-allowed' : 'bg-blue-500 hover:bg-blue-600'"
          :disabled="isRandomMode"
        >
          {{ isRandomMode ? '随机模式中...' : '开始随机模式' }}
        </button>
        <button 
          @click="nextKLine" 
          class="px-4 py-2 text-white rounded transition-colors"
          :class="!isRandomMode || currentIndex >= maxIndex ? 'bg-gray-400 cursor-not-allowed' : 'bg-green-500 hover:bg-green-600'"
          :disabled="!isRandomMode || currentIndex >= maxIndex"
        >
          下一根K线 ({{ currentIndex + 1 }}/{{ maxIndex + 1 }})
        </button>
        <button 
          @click="switchStock" 
          class="px-4 py-2 bg-orange-500 text-white rounded hover:bg-orange-600"
        >
          随机换股
        </button>
        <button 
          @click="resetChart" 
          class="px-4 py-2 bg-gray-500 text-white rounded hover:bg-gray-600"
        >
          重置图表
        </button>
        <div class="text-sm text-gray-600 space-y-1">
          <div>当前: {{ currentSymbol }} ({{ currentStockName }})</div>
          <div v-if="isRandomMode">
            随机起始日期: {{ randomStartDate }} | 进度: {{ currentIndex + 1 }}/{{ maxIndex + 1 }}
          </div>
          <div v-else class="text-blue-600">
            点击"开始随机模式"来启用历史回放功能
          </div>
        </div>
      </div>
    </div>
    <div ref="chartContainer" class="flex-1" style="height: calc(100vh - 120px);"></div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, nextTick } from 'vue'
import { KLineChartPro } from '@klinecharts/pro'
import '@klinecharts/pro/dist/klinecharts-pro.css'

interface Props {
  symbol?: string
}

const props = withDefaults(defineProps<Props>(), {
  symbol: '000001.SS'
})

// 定义emits
const emit = defineEmits<{
  stockChanged: [symbol: string, name: string]
  priceChanged: [price: number]
}>()

// 股票缓存存储
interface StockInfo {
  symbol: string
  name: string
  market: string
}

// 全局股票列表缓存
let allStocksList: StockInfo[] = []
let stocksLoaded = false
let stocksLoading = false

// 初始备用股票列表（作为后备）
const fallbackStocks: StockInfo[] = [
  { symbol: '600519.SS', name: '贵州茅台', market: 'SH' },
  { symbol: '000858.SZ', name: '五粮液', market: 'SZ' },
  { symbol: '002594.SZ', name: '比亚迪', market: 'SZ' },
  { symbol: '600036.SS', name: '招商银行', market: 'SH' },
  { symbol: '000002.SZ', name: '万科A', market: 'SZ' },
  { symbol: '601318.SS', name: '中国平安', market: 'SH' },
  { symbol: '002415.SZ', name: '海康威视', market: 'SZ' },
  { symbol: '300059.SZ', name: '东方财富', market: 'SZ' },
  { symbol: '600276.SS', name: '恒瑞医药', market: 'SH' },
  { symbol: '000333.SZ', name: '美的集团', market: 'SZ' }
]

// 从多个数据源获取A股股票列表
const fetchAllStocks = async (): Promise<StockInfo[]> => {
  if (stocksLoading) {
    console.log('股票数据正在加载中...')
    return []
  }
  
  if (stocksLoaded && allStocksList.length > 0) {
    console.log(`从缓存返回${allStocksList.length}只股票`)
    return allStocksList
  }

  stocksLoading = true
  console.log('开始获取A股股票列表...')

  try {
    // 方法1: 尝试从新浪财经获取股票列表
    const sinaStocks = await fetchFromSina()
    if (sinaStocks.length > 0) {
      allStocksList = sinaStocks
      stocksLoaded = true
      stocksLoading = false
      console.log(`从新浪财经获取到${allStocksList.length}只股票`)
      
      // 缓存到localStorage
      try {
        localStorage.setItem('cachedStocks', JSON.stringify({
          data: allStocksList,
          timestamp: Date.now(),
          expiry: Date.now() + 24 * 60 * 60 * 1000 // 24小时过期
        }))
      } catch (e) {
        console.warn('缓存股票列表失败:', e)
      }
      
      return allStocksList
    }

    // 方法2: 尝试从本地缓存读取
    const cachedData = tryLoadFromCache()
    if (cachedData.length > 0) {
      allStocksList = cachedData
      stocksLoaded = true
      stocksLoading = false
      console.log(`从本地缓存获取到${allStocksList.length}只股票`)
      return allStocksList
    }

    // 方法3: 生成模拟的A股股票列表
    const generatedStocks = generateFullStocksList()
    allStocksList = generatedStocks
    stocksLoaded = true
    stocksLoading = false
    console.log(`生成了${allStocksList.length}只模拟股票`)
    return allStocksList

  } catch (error) {
    console.error('获取股票列表失败:', error)
    stocksLoading = false
    
    // 使用备用股票列表
    allStocksList = [...fallbackStocks]
    console.log('使用备用股票列表')
    return allStocksList
  }
}

// 从新浪财经获取股票列表
const fetchFromSina = async (): Promise<StockInfo[]> => {
  try {
    // 由于跨域限制，这里使用模拟数据
    // 在实际生产环境中，需要通过后端代理或CORS来解决
    console.log('尝试从新浪财经获取数据...')
    
    // 模拟网络延迟
    await new Promise(resolve => setTimeout(resolve, 1000))
    
    // 这里返回空数组，让程序使用其他方法
    return []
  } catch (error) {
    console.error('新浪财经API调用失败:', error)
    return []
  }
}

// 尝试从本地缓存加载
const tryLoadFromCache = (): StockInfo[] => {
  try {
    const cached = localStorage.getItem('cachedStocks')
    if (cached) {
      const parsedCache = JSON.parse(cached)
      
      // 检查缓存是否过期
      if (parsedCache.expiry && Date.now() < parsedCache.expiry) {
        return parsedCache.data || []
      } else {
        // 清除过期缓存
        localStorage.removeItem('cachedStocks')
      }
    }
  } catch (error) {
    console.error('读取缓存失败:', error)
    localStorage.removeItem('cachedStocks')
  }
  return []
}

// 生成完整的模拟A股股票列表
const generateFullStocksList = (): StockInfo[] => {
  const stocks: StockInfo[] = []
  
  // 上证主板 600000-603999
  for (let i = 0; i < 200; i++) {
    const num = (600000 + Math.floor(Math.random() * 4000)).toString()
    if (num.startsWith('600') || num.startsWith('601') || num.startsWith('603')) {
      stocks.push({
        symbol: `${num}.SS`,
        name: generateCompanyName(),
        market: 'SH'
      })
    }
  }
  
  // 深证主板 000001-000999
  for (let i = 0; i < 100; i++) {
    const num = Math.floor(Math.random() * 1000).toString().padStart(6, '000')
    stocks.push({
      symbol: `${num}.SZ`,
      name: generateCompanyName(),
      market: 'SZ'
    })
  }
  
  // 中小板 002000-002999
  for (let i = 0; i < 100; i++) {
    const num = (2000 + Math.floor(Math.random() * 1000)).toString().padStart(6, '0')
    stocks.push({
      symbol: `${num}.SZ`,
      name: generateCompanyName(),
      market: 'SZ'
    })
  }
  
  // 创业板 300000-300999
  for (let i = 0; i < 100; i++) {
    const num = (300000 + Math.floor(Math.random() * 1000)).toString()
    stocks.push({
      symbol: `${num}.SZ`,
      name: generateCompanyName(),
      market: 'SZ'
    })
  }
  
  // 科创板 688000-688999
  for (let i = 0; i < 50; i++) {
    const num = (688000 + Math.floor(Math.random() * 1000)).toString()
    stocks.push({
      symbol: `${num}.SS`,
      name: generateCompanyName(),
      market: 'SH'
    })
  }
  
  // 添加备用股票确保有知名股票
  stocks.unshift(...fallbackStocks)
  
  // 去重
  const uniqueStocks = stocks.filter((stock, index, arr) => 
    arr.findIndex(s => s.symbol === stock.symbol) === index
  )
  
  return uniqueStocks
}

// 生成随机公司名称
const generateCompanyName = (): string => {
  const prefixes = ['华', '中', '北', '南', '东', '西', '大', '小', '新', '老', '金', '银', '盛', '恒', '永', '昌', '兴', '发', '达', '富', '贵', '康', '安', '泰', '和', '美', '好', '优', '强', '力', '威', '雄', '宏', '广', '深', '高', '远', '长', '久', '联', '合', '同', '众', '多', '万', '千', '百', '十']
  const industries = ['科技', '实业', '控股', '集团', '股份', '发展', '投资', '建设', '制造', '能源', '材料', '医药', '化工', '机械', '电子', '软件', '通信', '汽车', '食品', '纺织', '服装', '家具', '房地产', '金融', '银行', '保险', '证券', '信托', '租赁', '担保']
  const suffixes = ['有限公司', '股份有限公司', '集团', '控股', '实业', '科技', '发展']
  
  const prefix = prefixes[Math.floor(Math.random() * prefixes.length)]
  const industry = industries[Math.floor(Math.random() * industries.length)]
  const suffix = suffixes[Math.floor(Math.random() * suffixes.length)]
  
  return `${prefix}${industry}${suffix}`.slice(0, 8) // 限制长度
}

// 获取随机股票（从缓存的股票列表中选择）
const getRandomStock = async (): Promise<{ symbol: string, name: string }> => {
  try {
    const stocks = await fetchAllStocks()
    
    if (stocks.length === 0) {
      // 如果没有股票数据，使用备用列表
      const fallbackStock = fallbackStocks[Math.floor(Math.random() * fallbackStocks.length)]
      return { symbol: fallbackStock.symbol, name: fallbackStock.name }
    }
    
    // 从完整股票列表中随机选择
    const randomStock = stocks[Math.floor(Math.random() * stocks.length)]
    return { symbol: randomStock.symbol, name: randomStock.name }
  } catch (error) {
    console.error('获取随机股票失败:', error)
    // 发生错误时使用备用股票
    const fallbackStock = fallbackStocks[Math.floor(Math.random() * fallbackStocks.length)]
    return { symbol: fallbackStock.symbol, name: fallbackStock.name }
  }
}

// 根据股票代码查找股票信息
const findStockInfo = (symbol: string): { symbol: string, name: string } => {
  // 先从缓存的股票列表中查找
  if (allStocksList.length > 0) {
    const found = allStocksList.find((stock: StockInfo) => stock.symbol === symbol)
    if (found) {
      return { symbol: found.symbol, name: found.name }
    }
  }
  
  // 再从备用列表中查找
  const fallbackFound = fallbackStocks.find(stock => stock.symbol === symbol)
  if (fallbackFound) {
    return { symbol: fallbackFound.symbol, name: fallbackFound.name }
  }
  
  // 如果都没找到，返回默认值
  return { symbol, name: symbol }
}

const chartContainer = ref<HTMLDivElement>()
let chart: any = null
let fullHistoryData: any[] = []

// Random mode state
const isRandomMode = ref(false)
const currentIndex = ref(0)
const maxIndex = ref(0)
const randomStartDate = ref('')

// Current stock state
const currentSymbol = ref(props.symbol)
const currentStockName = ref('上证指数')

// 自定义数据源类
class CustomDatafeed {
  private data: any[] = []
  private currentDataIndex = 0
  private chart: any = null
  private symbol: string = ''
  
  constructor(data: any[], startIndex: number = 0, symbol: string = '') {
    this.data = data
    this.currentDataIndex = startIndex
    this.symbol = symbol
  }
  
  setChart(chart: any) {
    this.chart = chart
  }
  
  async searchSymbols(_userInput: string) {
    const stockInfo = findStockInfo(this.symbol)
    return [
      {
        exchange: 'Shanghai',
        fullName: stockInfo.name,
        market: 'stocks',
        shortName: stockInfo.name,
        ticker: this.symbol,
        priceCurrency: 'cny'
      }
    ]
  }
  
  async getHistoryKLineData(_symbol: any, _period: any, _from: number, _to: number) {
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
  
  async subscribe(_params: any, _callback: Function) {
    // 不需要实时数据
  }
  
  async unsubscribe(_params: any) {
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

// 生成模拟股票数据
const generateMockStockData = (_symbol: string, days: number = 730): any[] => {
  const data: any[] = []
  const now = new Date()
  let currentPrice = Math.random() * 100 + 50 // 随机起始价格50-150
  
  for (let i = days; i >= 0; i--) {
    const date = new Date(now.getTime() - i * 24 * 60 * 60 * 1000)
    const timestamp = Math.floor(date.getTime() / 1000)
    
    // 模拟价格波动
    const change = (Math.random() - 0.5) * 0.1 // -5%到+5%的变化
    currentPrice = Math.max(currentPrice * (1 + change), 1) // 确保价格不会为负
    
    const open = currentPrice
    const volatility = Math.random() * 0.05 // 0-5%的日内波动
    const high = open * (1 + volatility)
    const low = open * (1 - volatility)
    const close = low + Math.random() * (high - low)
    const volume = Math.floor(Math.random() * 1000000) + 100000
    
    data.push({
      timestamp,
      open: parseFloat(open.toFixed(2)),
      high: parseFloat(high.toFixed(2)),
      low: parseFloat(low.toFixed(2)),
      close: parseFloat(close.toFixed(2)),
      volume
    })
    
    currentPrice = close
  }
  
  return data.sort((a, b) => a.timestamp - b.timestamp)
}

// 获取股票历史数据 (使用模拟数据)
const fetchStockData = async (symbol: string): Promise<any[]> => {
  try {
    // 这里可以替换为真实的API调用
    console.log(`获取股票数据: ${symbol}`)
    
    // 模拟网络延迟
    await new Promise(resolve => setTimeout(resolve, 500))
    
    return generateMockStockData(symbol, 730) // 生成2年的数据
  } catch (error) {
    console.error('Failed to fetch stock data:', error)
    return []
  }
}

const initChart = async () => {
  if (!chartContainer.value) return
  
  await nextTick()
  
  try {
    // 获取历史数据用于展示
    const historyData = await fetchStockData(currentSymbol.value)
    
    if (historyData.length > 0) {
      // 创建自定义数据源
      const customDatafeed = new CustomDatafeed(historyData, historyData.length - 1, currentSymbol.value)
      
      const stockInfo = findStockInfo(currentSymbol.value)
      
      // 按照官方文档的简单方式创建图表
      chart = new KLineChartPro({
        container: chartContainer.value,
        symbol: {
          exchange: 'Shanghai',
          market: 'stocks',
          shortName: stockInfo.name,
          ticker: currentSymbol.value,
          priceCurrency: 'cny'
        },
        period: { multiplier: 1, timespan: 'day', text: '1D' },
        datafeed: customDatafeed as any
      })
      
      // 设置红涨绿跌的颜色主题
      chart.setStyles({
        candle: {
          bar: {
            upColor: '#FF4444',      // 红色表示上涨
            downColor: '#00AA00',    // 绿色表示下跌
            noChangeColor: '#888888' // 灰色表示无变化
          }
        },
        indicator: {
          vol: {
            bar: {
              upColor: '#FF4444',      // 成交量红色表示上涨
              downColor: '#00AA00',    // 成交量绿色表示下跌
              noChangeColor: '#888888' // 成交量灰色表示无变化
            }
          }
        }
      })
      
      customDatafeed.setChart(chart)
      
      // 发射初始价格（最新收盘价）
      const latestPrice = historyData[historyData.length - 1]?.close || 0
      emit('priceChanged', latestPrice)
      
      console.log('KLineChart Pro initialized successfully with Yahoo Finance data')
    }
  } catch (error) {
    console.error('Failed to initialize KLineChart Pro:', error)
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
    console.log('开始获取股票数据...')
    // 获取完整历史数据
    fullHistoryData = await fetchStockData(currentSymbol.value)
    
    console.log(`获取到 ${fullHistoryData.length} 天的历史数据`)
    
    if (fullHistoryData.length < 100) {
      console.warn(`历史数据不足，需要至少100天的数据，当前只有${fullHistoryData.length}天`)
      return
    }
    
    // 随机选择起始点，确保至少有100天的后续数据
    const minStartIndex = 50 // 确保前面有一些数据
    const maxStartIndex = Math.max(minStartIndex, fullHistoryData.length - 100) // 确保后面至少有100天
    const randomStartIndex = Math.floor(Math.random() * (maxStartIndex - minStartIndex)) + minStartIndex
    
    console.log(`随机起始索引: ${randomStartIndex}, 范围: ${minStartIndex} - ${maxStartIndex}`)
    
    // 设置随机模式状态
    isRandomMode.value = true
    currentIndex.value = randomStartIndex
    maxIndex.value = fullHistoryData.length - 1
    
    console.log(`随机模式设置: currentIndex=${currentIndex.value}, maxIndex=${maxIndex.value}, 数据长度=${fullHistoryData.length}`)
    
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
    const customDatafeed = new CustomDatafeed(fullHistoryData, randomStartIndex, currentSymbol.value)
    
    const stockInfo = findStockInfo(currentSymbol.value)
    
    // 创建新图表使用自定义数据源
    if (chartContainer.value) {
      chart = new KLineChartPro({
        container: chartContainer.value,
        symbol: {
          exchange: 'Shanghai',
          market: 'stocks',
          shortName: stockInfo.name,
          ticker: currentSymbol.value,
          priceCurrency: 'cny'
        },
        period: { multiplier: 1, timespan: 'day', text: '1D' },
        datafeed: customDatafeed as any
      })
      
      // 设置红涨绿跌的颜色主题
      chart.setStyles({
        candle: {
          bar: {
            upColor: '#FF4444',      // 红色表示上涨
            downColor: '#00AA00',    // 绿色表示下跌
            noChangeColor: '#888888' // 灰色表示无变化
          }
        },
        indicator: {
          vol: {
            bar: {
              upColor: '#FF4444',      // 成交量红色表示上涨
              downColor: '#00AA00',    // 成交量绿色表示下跌
              noChangeColor: '#888888' // 成交量灰色表示无变化
            }
          }
        }
      })
      
      // 将图表实例传递给数据源
      customDatafeed.setChart(chart)
      
      // 发射随机起始点的价格
      const randomStartKLine = fullHistoryData[randomStartIndex]
      if (randomStartKLine) {
        emit('priceChanged', randomStartKLine.close)
        console.log(`随机模式起始价格: ${randomStartKLine.close}`)
      }
    }
    
    console.log(`随机模式已开始，起始日期: ${randomStartDate.value}`)
  } catch (error) {
    console.error('Failed to start random mode:', error)
    console.error('启动随机模式失败', error)
    // 即使出错也要重置状态
    isRandomMode.value = false
    currentIndex.value = 0
    maxIndex.value = 0
    randomStartDate.value = ''
  }
}

// 下一根K线
const nextKLine = async () => {
  if (!isRandomMode.value) {
    console.warn('请先开始随机模式')
    // 可以添加用户提示
    return
  }
  
  if (!chart) {
    console.warn('图表未初始化')
    return
  }
  
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
  const newCustomDatafeed = new CustomDatafeed(fullHistoryData, currentIndex.value, currentSymbol.value)
  
  const stockInfo = findStockInfo(currentSymbol.value)
  
  // 重新创建图表以显示新数据
  try {
    if (chartContainer.value) {
      chart = new KLineChartPro({
        container: chartContainer.value,
        symbol: {
          exchange: 'Shanghai',
          market: 'stocks',
          shortName: stockInfo.name,
          ticker: currentSymbol.value,
          priceCurrency: 'cny'
        },
        period: { multiplier: 1, timespan: 'day', text: '1D' },
        datafeed: newCustomDatafeed as any
      })
      
      // 设置红涨绿跌的颜色主题
      chart.setStyles({
        candle: {
          bar: {
            upColor: '#FF4444',      // 红色表示上涨
            downColor: '#00AA00',    // 绿色表示下跌
            noChangeColor: '#888888' // 灰色表示无变化
          }
        },
        indicator: {
          vol: {
            bar: {
              upColor: '#FF4444',      // 成交量红色表示上涨
              downColor: '#00AA00',    // 成交量绿色表示下跌
              noChangeColor: '#888888' // 成交量灰色表示无变化
            }
          }
        }
      })
      
      newCustomDatafeed.setChart(chart)
      
      // 发射当前K线的收盘价
      const currentKLine = fullHistoryData[currentIndex.value]
      if (currentKLine) {
        emit('priceChanged', currentKLine.close)
        console.log(`价格更新为: ${currentKLine.close}`)
      }
    }
  } catch (error) {
    console.error('Failed to update chart:', error)
  }
  
  console.log(`已更新到第 ${currentIndex.value + 1} 根K线`)
}

// 随机换股
const switchStock = async () => {
  // 使用新的随机股票选择函数
  const randomStock = await getRandomStock()
  currentSymbol.value = randomStock.symbol
  currentStockName.value = randomStock.name
  
  // 重置随机模式状态
  isRandomMode.value = false
  currentIndex.value = 0
  maxIndex.value = 0
  randomStartDate.value = ''
  fullHistoryData = []
  
  // 清理现有图表
  cleanupChart()
  
  // 重新初始化图表 (initChart会自动发射价格更新事件)
  await initChart()
  
  // 通知父组件股票已变化
  emit('stockChanged', randomStock.symbol, randomStock.name)
  
  console.log(`已切换到股票: ${randomStock.name} (${randomStock.symbol})`)
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

onMounted(async () => {
  // 异步预加载股票列表
  fetchAllStocks().then(stocks => {
    console.log(`股票列表预加载完成，共${stocks.length}只股票`)
  }).catch(error => {
    console.error('预加载股票列表失败:', error)
  })
  
  // 初始化图表
  await initChart()
})

onUnmounted(() => {
  cleanupChart()
})

// 暴露方法给父组件
defineExpose({
  changeSymbol: (newSymbol: string) => {
    const stockInfo = findStockInfo(newSymbol)
    currentSymbol.value = newSymbol
    currentStockName.value = stockInfo.name
    
    if (chart) {
      chart.setSymbol({
        exchange: 'Shanghai',
        market: 'stocks',
        shortName: stockInfo.name,
        ticker: newSymbol,
        priceCurrency: 'cny'
      })
    }
  },
  getCurrentPrice: () => {
    return 0
  },
  switchStock
})
</script>

<style scoped>
</style>