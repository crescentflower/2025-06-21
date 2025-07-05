<template>
  <div :style="{ backgroundColor: themeChartBgColor }">
    <Layout
      :class="{
        'layout-sider': true,
        'semi-always-dark': themeString === 'DARK',
        'semi-always-light': themeString === 'LIGHT'
      }"
      class="layout-container"
      :style="{ backgroundColor: themeChartBgColor }"
      style="overflow: hidden; width: 100vw; height: 100vh; display: flex; background-color: transparent;"
    >
      <!-- 主内容区 -->
      <LayoutContent
        class="layout-content"
        :style="{
          backgroundColor: themeChartBgColor,
          width: state === STATE_ARRAY[2] || state === STATE_ARRAY[3] ? '100% !important' : 'calc(100% - 340px) !important',
          height: state === STATE_ARRAY[2] || state === STATE_ARRAY[3] ? '100% !important' : '100vh !important',
        }"
      >
        <div class="kline-container" ref="klineContainer">
          <!-- 筛选控件区域 -->
          <div class="filters-container" v-if="(showTimeFilter || showSymbolFilter) && (state === STATE_ARRAY[0] || state === STATE_ARRAY[1])">
            <!-- 时间筛选控件 -->
            <div class="time-filter" v-if="showTimeFilter">
              <div class="filter-control">
                <label>开始时间:</label>
                <input type="datetime-local" v-model="startDate" @change="applyFilters">
              </div>
              <div class="filter-control">
                <label>结束时间:</label>
                <input type="datetime-local" v-model="endDate" @change="applyFilters">
              </div>
              <div class="filter-control">
                <Button @click="resetTimeFilter" size="small">重置时间</Button>
              </div>
              <!-- 新增快速选择按钮 -->
              <div class="quick-buttons">
                <Button @click="setToday" size="small">本日</Button>
                <Button @click="setThisWeek" size="small">本周</Button>
                <Button @click="setThisMonth" size="small">本月</Button>
              </div>
            </div>
            
            <!-- 品种筛选控件 -->
            <div class="symbol-filter" v-if="showSymbolFilter">
              <div class="filter-control">
                <label>品种筛选:</label>
                <select v-model="selectedSymbol" @change="applyFilters" style="padding: 8px; border-radius: 4px; border: 1px solid var(--semi-color-border); background-color: var(--semi-color-bg-0); color: var(--semi-color-text-0);">
                  <option value="">全部品种</option>
                  <option v-for="symbol in uniqueSymbols" :key="symbol" :value="symbol">{{ symbol }}</option>
                </select>
              </div>
              <div class="filter-control">
                <Button @click="resetSymbolFilter" size="small">重置品种</Button>
              </div>
            </div>
          </div>
          
          <div class="chart-container" ref="chartContainer">
            <div class="chart-wrapper" ref="chartWrapper" style="width: 100%; height: 100%;"></div>
          </div>
          
          <!-- 加载动画 -->
          <div v-if="isLoading" class="loading-overlay">
            <div class="loading-spinner"></div>
            <div class="loading-text">数据加载中...</div>
          </div>
        </div>

        <!-- 错误信息 -->
        <div v-if="errorMessage" class="error-message">
          {{ errorMessage }}
        </div>
        
        <!-- 无数据提示 -->
        <div v-if="!isLoading && filteredKlineData.length === 0 && !errorMessage" class="no-data">
          <div v-if="targetTableId && targetViewId">当前选择的数据表没有有效数据</div>
          <div v-else>请选择数据表以显示K线图</div>
        </div>
      </LayoutContent>

      <!-- 配置区域 -->
      <LayoutSider
        v-if="state === STATE_ARRAY[1] || state === STATE_ARRAY[0]"
        :class="{
          'layout-sider': true,
          'semi-always-dark': themeString == 'DARK',
          'semi-always-light': themeString == 'LIGHT'
        }"
        style="
          height: 100%;
          border-left: 1px solid var(--semi-color-stroke);
          flex: 0 0 340px;
        "
      >
        <div
          class="formPanel"
          style="width: 100%; height: 100%; display: flex; flex-direction: column"
        >
          <div
            class="form"
            style="
              width: 100%;
              height: 100%;
              display: flex;
              flex-direction: column;
              overflow: auto;
            "
          >
            <!-- 数据表选择 -->
            <div class="a-form-item">
              <div class="lable">数据表</div>
              <Select
                :optionList="tableOptionList"
                :value="targetTableId"
                @change="onTableIdChange"
                style="
                  margin-top: 8px;
                  width: 100%;
                  background-color: transparent;
                  border: 1px solid var(--semi-color-stroke);
                  border-radius: 6px;
                "
              />
            </div>

            <!-- 视图选择 -->
            <div class="a-form-item">
              <div class="lable">数据视图</div>
              <Select
                :optionList="viewOptionList"
                :value="targetViewId"
                @change="onViewIdChange"
                style="
                  margin-top: 8px;
                  width: 100%;
                  background-color: transparent;
                  border: 1px solid var(--semi-color-stroke);
                  border-radius: 6px;
                "
              />
            </div>
            
            <!-- 字段映射配置 -->
            <div class="a-form-item" v-if="targetTableId">
              <div class="lable">字段映射配置</div>
              <div class="field-mapping">
                <div class="mapping-item" v-for="field in fieldMappings" :key="field.key">
                  <div class="mapping-label">{{ field.label }}:</div>
                  <Select
                    :optionList="fieldOptionList"
                    :value="fieldMapping[field.key]"
                    @change="(value) => onFieldMappingChange(field.key, value)"
                    style="width: 100%; margin-top: 4px;"
                  />
                </div>
              </div>
            </div>
            
            <!-- 时间筛选开关 -->
            <div class="a-form-item">
              <div class="lable">时间筛选功能</div>
              <div style="display: flex; align-items: center; margin-top: 8px;">
                <Switch 
                  :checked="showTimeFilter" 
                  @change="onTimeFilterToggle" 
                  style="margin-right: 8px;"
                />
                <span>{{ showTimeFilter ? '已开启' : '已关闭' }}</span>
              </div>
            </div>
            
            <!-- 品种筛选开关 -->
            <div class="a-form-item">
              <div class="lable">品种筛选功能</div>
              <div style="display: flex; align-items: center; margin-top: 8px;">
                <Switch 
                  :checked="showSymbolFilter" 
                  @change="onSymbolFilterToggle" 
                  style="margin-right: 8px;"
                />
                <span>{{ showSymbolFilter ? '已开启' : '已关闭' }}</span>
              </div>
            </div>
            
            <!-- 样式配置 -->
            <div class="a-form-item">
              <div class="lable">图表样式设置</div>
              <div class="style-container">
                <div class="style-row">
                  <div class="style-item">
                    <div class="style-label">上涨颜色</div>
                    <Input
                      class-name="colorInput"
                      size="default"
                      type="color"
                      style="
                        margin-top: 8px;
                        height: 34px;
                        border: 1px solid var(--semi-color-stroke);
                        border-radius: 6px;
                      "
                      :value="upColor"
                      @change="onUpColorChange"
                    />
                  </div>

                  <div class="style-item">
                    <div class="style-label">下跌颜色</div>
                    <Input
                      class-name="colorInput"
                      size="default"
                      type="color"
                      style="
                        margin-top: 8px;
                        height: 34px;
                        border: 1px solid var(--semi-color-stroke);
                        border-radius: 6px;
                      "
                      :value="downColor"
                      @change="onDownColorChange"
                    />
                  </div>
                </div>

                <div class="style-row">
                  <div class="style-item">
                    <div class="style-label">显示均线</div>
                    <div style="display: flex; align-items: center; margin-top: 8px;">
                      <Switch 
                        :checked="showMA" 
                        @change="onShowMAChange" 
                        style="margin-right: 8px;"
                      />
                      <span>{{ showMA ? '显示' : '隐藏' }}</span>
                    </div>
                  </div>

                  <div class="style-item">
                    <div class="style-label">显示成交量</div>
                    <div style="display: flex; align-items: center; margin-top: 8px;">
                      <Switch 
                        :checked="showVolume" 
                        @change="onShowVolumeChange" 
                        style="margin-right: 8px;"
                      />
                      <span>{{ showVolume ? '显示' : '隐藏' }}</span>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <!-- 保存按钮 -->
          <div
            class="a-form-item"
            style="height: 70px; display: flex; flex-direction: row-reverse; padding: 0 20px;"
          >
            <Button
              @click="saveConfig"
              theme="solid"
              type="primary"
              style="width: 80px; border-radius: 4px"
              :disabled="!isConfigValid"
            >保存配置</Button>
          </div>
        </div>
      </LayoutSider>
    </Layout>
  </div>
</template>

<script lang="ts" setup>
import { ref, onMounted, watch, nextTick, computed, onUnmounted } from 'vue';
import { dashboard, bitable } from '@lark-base-open/js-sdk';
import * as echarts from 'echarts';
import { 
  Layout, 
  LayoutSider, 
  LayoutContent, 
  Select, 
  Button, 
  InputNumber, 
  Input,
  Switch
} from '@kousum/semi-ui-vue';

import * as myBase from "../utils/base";

// 状态管理
const STATE_ARRAY = ["Create", "Config", "View", "FullScreen"];
type OptionItem = { label: string; value: string };
const themeChartBgColor = ref('');
const themeString = ref('');
const state = ref('Create');

// 数据表选择相关
const tableOptionList = ref<OptionItem[]>([]);
const targetTableId = ref('');
const viewOptionList = ref<OptionItem[]>([]);
const targetViewId = ref('allData');

// K线图相关
const chartContainer = ref<HTMLDivElement | null>(null);
const chartWrapper = ref<HTMLDivElement | null>(null);
let chartInstance: echarts.ECharts | null = null;
let currentResizeObserver: ResizeObserver | null = null; // 用于监听容器大小变化
const klineData = ref<any[]>([]);
const filteredKlineData = ref<any[]>([]);
const tooltipVisible = ref(false);

// 时间筛选相关
const showTimeFilter = ref(false);
const startDate = ref('');
const endDate = ref('');

// 品种筛选相关
const showSymbolFilter = ref(false);
const selectedSymbol = ref('');
const uniqueSymbols = ref<string[]>([]);

// 配置选项
const upColor = ref('#00da3c'); // 上涨颜色 (ECharts样式)
const downColor = ref('#ec0000'); // 下跌颜色 (ECharts样式)
const showVolume = ref(true); // 是否显示成交量
const showMA = ref(true); // 是否显示均线

// 错误和加载状态
const isLoading = ref(false);
const errorMessage = ref('');

// 字段映射配置
const fieldOptionList = ref<OptionItem[]>([]);
const fieldMappings = ref([
  { key: 'timestamp', label: '时间字段' },
  { key: 'open', label: '开盘价' },
  { key: 'high', label: '最高价' },
  { key: 'low', label: '最低价' },
  { key: 'close', label: '收盘价' },
  { key: 'volume', label: '成交量 (可选)' },
  { key: 'symbol', label: '交易品种 (可选)' }
]);

const fieldMapping = ref({
  timestamp: '',
  open: '',
  high: '',
  low: '',
  close: '',
  volume: '',
  symbol: ''
});

// 计算属性
const isConfigValid = computed(() => {
  return !errorMessage.value && klineData.value.length > 0;
});

// 主题变化监听
dashboard.onThemeChange(res => {
  themeChartBgColor.value = res.data.chartBgColor;
  if (res.data.chartBgColor === "var(--bg-body)") {
    themeChartBgColor.value = "#ffffff";
  }
  themeString.value = res.data.theme;
  
  // 主题变化时重绘K线图
  if (filteredKlineData.value.length > 0) {
    renderKLineChart();
  }
});

// 时间格式化函数
const formatDateTime = (timestamp: string | number) => {
  if (!timestamp) return '无时间信息';
  
  // 尝试解析时间戳
  let date: Date;
  if (typeof timestamp === 'string') {
    if (/^\d+$/.test(timestamp)) {
      date = new Date(Number(timestamp));
    } else {
      date = new Date(timestamp);
    }
  } else {
    date = new Date(timestamp);
  }
  
  if (isNaN(date.getTime())) {
    return timestamp.toString();
  }
  
  return date.toLocaleString(undefined, {
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit'
  });
};

// 格式化选项列表
const formatOptionList = (oldOptionList: { name: string; id: string }[]) => {
  return oldOptionList.map(item => ({
    label: item.name,
    value: item.id
  }));
};

// 初始化数据表选项
const initTableOptionList = async () => {
  const tableList = await myBase.queryBaseTableMetaList();
  tableOptionList.value = formatOptionList(tableList);
};

// 查询字段列表
const queryFieldOptionList = async (tableId: string) => {
  try {
    const table = await bitable.base.getTable(tableId);
    const fieldMetaList = await table.getFieldMetaList();
    const fieldList = fieldMetaList.map(field => ({
      label: field.name,
      value: field.id
    }));
    
    fieldList.unshift({ label: '不选择', value: '' });
    return fieldList;
  } catch (error) {
    console.error('查询字段列表失败:', error);
    return [];
  }
};

// 查询视图列表
const queryViewOptionList = async (tableId: string) => {
  try {
    const table = await bitable.base.getTable(tableId);
    const viewMetaList = await table.getViewMetaList();
    const viewList = [{ name: '全部数据', id: "allData" }, ...viewMetaList];
    return formatOptionList(viewList);
  } catch (error) {
    console.error('查询视图列表失败:', error);
    return [];
  }
};

// 获取K线数据
const fetchKLineData = async (tableId: string, viewId: string) => {
  if (!tableId) return;
  
  isLoading.value = true;
  klineData.value = [];
  filteredKlineData.value = [];
  uniqueSymbols.value = [];
  errorMessage.value = '';
  
  try {
    const table = await bitable.base.getTable(tableId);
    let records: any[] = [];
    let pageToken: string | undefined = undefined;
    let fieldMetaList: any[];
    
    if (viewId === 'allData') {
      fieldMetaList = await table.getFieldMetaList();
    } else {
      const view = await table.getViewById(viewId);
      fieldMetaList = await view.getFieldMetaList();
    }
    
    const fieldMap = new Map<string, string>();
    for (const field of fieldMetaList) {
      fieldMap.set(field.id, field.name.toLowerCase());
    }
    
    do {
      const response = await table.getRecords({
        pageSize: 5000,
        pageToken,
        viewId: viewId !== 'allData' ? viewId : undefined
      });
      
      records = [...records, ...response.records];
      pageToken = response.pageToken;
    } while (pageToken);
    
    if (records.length > 0) {
      const kData = [];
      const symbolSet = new Set<string>();
      
      for (const record of records) {
        const fields = record.fields;
        const kPoint: any = {};
        
        for (const [key, fieldId] of Object.entries(fieldMapping.value)) {
          if (!fieldId) continue;
          
          let fieldValue = fields[fieldId];
          if (fieldValue === undefined || fieldValue === null) continue;
          
          if (key === 'symbol') {
            if (Array.isArray(fieldValue)) {
              fieldValue = fieldValue.map(item => item.text || '').join('');
            } else if (fieldValue && typeof fieldValue === 'object' && 'text' in fieldValue) {
              fieldValue = fieldValue.text;
            }
            fieldValue = String(fieldValue);
          }
          
          // 特殊处理时间戳字段
          if (key === 'timestamp') {
            // 尝试转换为时间戳
            if (typeof fieldValue === 'string' && /^\d+$/.test(fieldValue)) {
              fieldValue = Number(fieldValue);
            } else if (typeof fieldValue === 'string') {
              // 尝试解析日期字符串
              const date = new Date(fieldValue);
              if (!isNaN(date.getTime())) {
                fieldValue = date.getTime();
              }
            }
          }
          
          kPoint[key] = fieldValue;
          
          if (key === 'symbol' && fieldValue) {
            symbolSet.add(fieldValue);
          }
        }
        
        const requiredFields = ['timestamp', 'open', 'high', 'low', 'close'];
        let hasRequiredFields = true;
        for (const field of requiredFields) {
          if (kPoint[field] === undefined || kPoint[field] === null) {
            hasRequiredFields = false;
            break;
          }
        }
        
        if (hasRequiredFields) {
          // 确保所有价格字段都是数字
          ['open', 'high', 'low', 'close', 'volume'].forEach(field => {
            if (kPoint[field] !== undefined && kPoint[field] !== null) {
              const num = Number(kPoint[field]);
              kPoint[field] = isNaN(num) ? 0 : num;
            }
          });
          
          kData.push(kPoint);
        }
      }
      
      // 按时间戳排序
      kData.sort((a, b) => {
        return a.timestamp - b.timestamp;
      });
      
      if (kData.length === 0) {
        errorMessage.value = '没有找到有效数据';
      } else {
        klineData.value = kData;
        uniqueSymbols.value = Array.from(symbolSet).sort();
        
        if (kData.length > 0 && (!startDate.value || !endDate.value)) {
          const minTime = new Date(kData[0].timestamp);
          const maxTime = new Date(kData[kData.length - 1].timestamp);
          
          const defaultStart = new Date(maxTime);
          defaultStart.setDate(defaultStart.getDate() - 30);
          
          startDate.value = formatDateForInput(defaultStart);
          endDate.value = formatDateForInput(maxTime);
        }
        
        applyFilters();
      }
    } else {
      errorMessage.value = '当前视图没有数据';
    }
  } catch (error) {
    console.error('获取K线数据失败:', error);
    errorMessage.value = `数据加载失败: ${(error as Error).message}`;
  } finally {
    setTimeout(() => {
      isLoading.value = false;
    }, 500);
  }
};

// 将日期格式化为YYYY-MM-DDTHH:mm格式
const formatDateForInput = (date: Date) => {
  const year = date.getFullYear();
  const month = String(date.getMonth() + 1).padStart(2, '0');
  const day = String(date.getDate()).padStart(2, '0');
  const hours = String(date.getHours()).padStart(2, '0');
  const minutes = String(date.getMinutes()).padStart(2, '0');
  
  return `${year}-${month}-${day}T${hours}:${minutes}`;
};

// 应用筛选（时间和品种）
const applyFilters = () => {
  let filteredData = [...klineData.value];
  
  if (showTimeFilter.value && startDate.value && endDate.value) {
    const start = new Date(startDate.value).getTime();
    const end = new Date(endDate.value).getTime();
    
    filteredData = filteredData.filter(item => {
      const time = item.timestamp;
      return time >= start && time <= end;
    });
  }
  
  if (showSymbolFilter.value && selectedSymbol.value) {
    filteredData = filteredData.filter(item => {
      return item.symbol && String(item.symbol) === selectedSymbol.value;
    });
  }
  
  filteredKlineData.value = filteredData;
  
  renderKLineChart();
};

// 重置时间筛选
const resetTimeFilter = () => {
  if (klineData.value.length > 0) {
    const minTime = klineData.value[0].timestamp;
    const maxTime = klineData.value[klineData.value.length - 1].timestamp;
    
    startDate.value = formatDateForInput(new Date(minTime));
    endDate.value = formatDateForInput(new Date(maxTime));
  }
  
  applyFilters();
};

// 重置品种筛选
const resetSymbolFilter = () => {
  selectedSymbol.value = '';
  applyFilters();
};

// 新增：设置本日时间范围
const setToday = () => {
  const now = new Date();
  const start = new Date(now);
  start.setHours(0, 0, 0, 0);
  
  const end = new Date(now);
  end.setHours(23, 59, 59, 999);
  
  startDate.value = formatDateForInput(start);
  endDate.value = formatDateForInput(end);
  applyFilters();
};

// 新增：设置本周时间范围
const setThisWeek = () => {
  const now = new Date();
  const day = now.getDay();
  const start = new Date(now);
  
  start.setDate(now.getDate() - (day === 0 ? 6 : day - 1));
  start.setHours(0, 0, 0, 0);
  
  const end = new Date(now);
  end.setDate(now.getDate() + (day === 0 ? 0 : 7 - day));
  end.setHours(23, 59, 59, 999);
  
  startDate.value = formatDateForInput(start);
  endDate.value = formatDateForInput(end);
  applyFilters();
};

// 新增：设置本月时间范围
const setThisMonth = () => {
  const now = new Date();
  const start = new Date(now.getFullYear(), now.getMonth(), 1, 0, 0, 0, 0);
  const end = new Date(now.getFullYear(), now.getMonth() + 1, 0, 23, 59, 59, 999);
  
  startDate.value = formatDateForInput(start);
  endDate.value = formatDateForInput(end);
  applyFilters();
};

// 计算均线
const calculateMA = (dayCount: number, data: any[]) => {
  const result = [];
  for (let i = 0; i < data.length; i++) {
    if (i < dayCount) {
      result.push('-');
      continue;
    }
    let sum = 0;
    for (let j = 0; j < dayCount; j++) {
      sum += data[i - j][1]; // 收盘价在索引1的位置
    }
    result.push(+(sum / dayCount).toFixed(2));
  }
  return result;
};

// 渲染K线图 - 使用ECharts样式
const renderKLineChart = () => {
  if (!chartWrapper.value || filteredKlineData.value.length === 0) {
    return;
  }
  
  // 销毁旧图表实例和观察器
  if (chartInstance) {
    chartInstance.dispose();
    chartInstance = null;
  }
  
  if (currentResizeObserver) {
    currentResizeObserver.disconnect();
    currentResizeObserver = null;
  }
  
  // 创建新图表实例
  chartInstance = echarts.init(chartWrapper.value);
  
  // 设置ResizeObserver监听容器大小变化
  if (chartWrapper.value) {
    currentResizeObserver = new ResizeObserver(() => {
      if (chartInstance) {
        chartInstance.resize();
      }
    });
    currentResizeObserver.observe(chartWrapper.value);
  }
  
  // 准备ECharts数据格式
  const categoryData = [];
  const values = [];
  const volumes = [];
  
  // 直接使用原始数据点，用于tooltip显示
  const rawDataPoints = [];
  
  for (let i = 0; i < filteredKlineData.value.length; i++) {
    const data = filteredKlineData.value[i];
    const date = new Date(data.timestamp);
    
    // 格式化日期为YYYY-MM-DD
    const dateStr = `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}-${String(date.getDate()).padStart(2, '0')}`;
    
    categoryData.push(dateStr);
    
    // 确保所有价格字段都是数字
    const open = Number(data.open);
    const close = Number(data.close);
    const low = Number(data.low);
    const high = Number(data.high);
    const volume = data.volume ? Number(data.volume) : 0;
    
    // K线数据格式：[开盘价, 收盘价, 最低价, 最高价]
    values.push([open, close, low, high]);
    
    // 成交量数据 [索引, 成交量, 涨跌标记]
    volumes.push([
      i,
      volume,
      open > close ? 1 : -1  // 1表示下跌，-1表示上涨
    ]);
    
    // 保存原始数据点，用于tooltip显示
    rawDataPoints.push({
      date: dateStr,
      open: open,
      close: close,
      low: low,
      high: high,
      volume: volume
    });
  }
  
  // 计算均线
  const ma5 = showMA.value ? calculateMA(5, values) : [];
  const ma10 = showMA.value ? calculateMA(10, values) : [];
  const ma20 = showMA.value ? calculateMA(20, values) : [];
  const ma30 = showMA.value ? calculateMA(30, values) : [];
  
  // ECharts配置选项
  const option = {
    animation: false,
    legend: {
      bottom: 10,
      left: 'center',
      data: showMA.value ? ['K线', 'MA5', 'MA10', 'MA20', 'MA30'] : ['K线']
    },
    tooltip: {
      trigger: 'axis',
      axisPointer: {
        type: 'cross'
      },
      borderWidth: 1,
      borderColor: themeString.value === 'DARK' ? '#555' : '#ccc',
      padding: 10,
      textStyle: {
        color: themeString.value === 'DARK' ? '#eee' : '#333'
      },
      backgroundColor: themeString.value === 'DARK' ? 'rgba(50, 50, 50, 0.8)' : 'rgba(255, 255, 255, 0.9)',
      formatter: function (params: any) {
        // 获取当前数据点的索引
        const dataIndex = params[0].dataIndex;
        
        // 直接使用原始数据点
        const dataPoint = rawDataPoints[dataIndex];
        
        let html = `<div style="margin-bottom: 5px; font-weight: bold;">${dataPoint.date}</div>`;
        html += `<div>开盘价: ${dataPoint.open.toFixed(2)}</div>`;
        html += `<div>收盘价: ${dataPoint.close.toFixed(2)}</div>`;
        html += `<div>最低价: ${dataPoint.low.toFixed(2)}</div>`;
        html += `<div>最高价: ${dataPoint.high.toFixed(2)}</div>`;
        
        // 显示成交量
        if (showVolume.value) {
          html += `<div>成交量: ${dataPoint.volume}</div>`;
        }
        
        // 显示均线值
        if (showMA.value) {
          // 只显示当前点的均线值
          if (ma5[dataIndex] !== '-' && ma5[dataIndex] !== undefined) {
            html += `<div>MA5: ${ma5[dataIndex].toFixed(2)}</div>`;
          }
          if (ma10[dataIndex] !== '-' && ma10[dataIndex] !== undefined) {
            html += `<div>MA10: ${ma10[dataIndex].toFixed(2)}</div>`;
          }
          if (ma20[dataIndex] !== '-' && ma20[dataIndex] !== undefined) {
            html += `<div>MA20: ${ma20[dataIndex].toFixed(2)}</div>`;
          }
          if (ma30[dataIndex] !== '-' && ma30[dataIndex] !== undefined) {
            html += `<div>MA30: ${ma30[dataIndex].toFixed(2)}</div>`;
          }
        }
        
        return html;
      }
    },
    axisPointer: {
      link: [{ xAxisIndex: 'all' }],
      label: {
        backgroundColor: themeString.value === 'DARK' ? '#555' : '#777'
      }
    },
    grid: [
      {
        left: '10%',
        right: '8%',
        height: '50%'
      },
      {
        left: '10%',
        right: '8%',
        top: '63%',
        height: '16%'
      }
    ],
    xAxis: [
      {
        type: 'category',
        data: categoryData,
        boundaryGap: false,
        axisLine: { onZero: false },
        splitLine: { show: false },
        min: 'dataMin',
        max: 'dataMax',
        axisLabel: {
          color: themeString.value === 'DARK' ? '#aaa' : '#666'
        },
        axisPointer: {
          z: 100
        }
      },
      {
        type: 'category',
        gridIndex: 1,
        data: categoryData,
        boundaryGap: false,
        axisLine: { onZero: false },
        axisTick: { show: false },
        splitLine: { show: false },
        axisLabel: { show: false },
        min: 'dataMin',
        max: 'dataMax'
      }
    ],
    yAxis: [
      {
        scale: true,
        splitArea: {
          show: true
        },
        axisLabel: {
          color: themeString.value === 'DARK' ? '#aaa' : '#666'
        }
      },
      {
        scale: true,
        gridIndex: 1,
        splitNumber: 2,
        axisLabel: { show: false },
        axisLine: { show: false },
        axisTick: { show: false },
        splitLine: { show: false }
      }
    ],
    dataZoom: [
      {
        type: 'inside',
        xAxisIndex: [0, 1],
        start: 0,
        end: 100
      },
      {
        show: true,
        xAxisIndex: [0, 1],
        type: 'slider',
        top: '85%',
        start: 0,
        end: 100,
        backgroundColor: themeString.value === 'DARK' ? '#333' : '#f5f5f5',
        borderColor: themeString.value === 'DARK' ? '#555' : '#ddd',
        fillerColor: themeString.value === 'DARK' ? 'rgba(47, 69, 84, 0.3)' : 'rgba(230, 230, 230, 0.3)',
        textStyle: {
          color: themeString.value === 'DARK' ? '#ccc' : '#333'
        }
      }
    ],
    series: [
      {
        name: 'K线',
        type: 'candlestick',
        data: values,
        itemStyle: {
          color: upColor.value,
          color0: downColor.value,
          borderColor: undefined,
          borderColor0: undefined
        }
      },
      ...(showMA.value ? [
        {
          name: 'MA5',
          type: 'line',
          data: ma5,
          smooth: true,
          lineStyle: {
            width: 1,
            opacity: 0.8
          },
          showSymbol: false
        },
        {
          name: 'MA10',
          type: 'line',
          data: ma10,
          smooth: true,
          lineStyle: {
            width: 1,
            opacity: 0.8
          },
          showSymbol: false
        },
        {
          name: 'MA20',
          type: 'line',
          data: ma20,
          smooth: true,
          lineStyle: {
            width: 1,
            opacity: 0.8
          },
          showSymbol: false
        },
        {
          name: 'MA30',
          type: 'line',
          data: ma30,
          smooth: true,
          lineStyle: {
            width: 1,
            opacity: 0.8
          },
          showSymbol: false
        }
      ] : []),
      ...(showVolume.value ? [{
        name: '成交量',
        type: 'bar',
        xAxisIndex: 1,
        yAxisIndex: 1,
        data: volumes,
        itemStyle: {
          color: function (params: any) {
            return params.value[2] > 0 ? downColor.value : upColor.value;
          }
        }
      }] : [])
    ]
  };
  
  // 设置图表选项
  chartInstance.setOption(option);
};

// 保存配置 - 完全按照原始逻辑
const saveConfig = async () => {
  if (!isConfigValid.value) return;
  
  const config = {
    customConfig: {
      tableId: targetTableId.value,
      viewId: targetViewId.value,
      upColor: upColor.value,
      downColor: downColor.value,
      showVolume: showVolume.value,
      showTimeFilter: showTimeFilter.value,
      showSymbolFilter: showSymbolFilter.value,
      showMA: showMA.value,
      selectedSymbol: selectedSymbol.value,
      startDate: startDate.value,
      endDate: endDate.value,
      fieldMapping: JSON.stringify(fieldMapping.value)
    },
  };
  
  try {
    await dashboard.saveConfig(config);
    state.value = dashboard.state;
    applyAllConfigChanges();
  } catch (error) {
    console.error('保存配置失败:', error);
    errorMessage.value = `保存配置失败: ${(error as Error).message}`;
  }
};

// 应用所有配置更改
const applyAllConfigChanges = () => {
  applyFilters();
  nextTick(() => {
    renderKLineChart();
  });
};

// 初始化仪表盘
const initDashboard = async (config: any) => {
  const customConfig = config.customConfig || {};
  
  targetTableId.value = customConfig.tableId || '';
  targetViewId.value = customConfig.viewId || 'allData';
  upColor.value = customConfig.upColor || '#00da3c';
  downColor.value = customConfig.downColor || '#ec0000';
  showVolume.value = customConfig.showVolume !== undefined ? customConfig.showVolume : true;
  showMA.value = customConfig.showMA !== undefined ? customConfig.showMA : true;
  showTimeFilter.value = customConfig.showTimeFilter !== undefined ? customConfig.showTimeFilter : false;
  showSymbolFilter.value = customConfig.showSymbolFilter !== undefined ? customConfig.showSymbolFilter : false;
  selectedSymbol.value = customConfig.selectedSymbol || '';
  
  startDate.value = customConfig.startDate || '';
  endDate.value = customConfig.endDate || '';
  
  if (customConfig.fieldMapping) {
    try {
      fieldMapping.value = JSON.parse(customConfig.fieldMapping);
    } catch (e) {
      console.error('解析字段映射配置失败:', e);
      fieldMapping.value = {
        timestamp: '',
        open: '',
        high: '',
        low: '',
        close: '',
        volume: '',
        symbol: ''
      };
    }
  } else {
    fieldMapping.value = {
      timestamp: '',
      open: '',
      high: '',
      low: '',
      close: '',
      volume: '',
      symbol: ''
    };
  }
  
  if (targetTableId.value) {
    viewOptionList.value = await queryViewOptionList(targetTableId.value);
    fieldOptionList.value = await queryFieldOptionList(targetTableId.value);
  }
  
  if (targetTableId.value) {
    await fetchKLineData(targetTableId.value, targetViewId.value);
  }
};

// 数据表选择变化处理
const onTableIdChange = async (value: string) => {
  targetTableId.value = value;
  viewOptionList.value = await queryViewOptionList(value);
  fieldOptionList.value = await queryFieldOptionList(value);
  targetViewId.value = "allData";
  
  fieldMapping.value = {
    timestamp: '',
    open: '',
    high: '',
    low: '',
    close: '',
    volume: '',
    symbol: ''
  };
  
  await fetchKLineData(value, targetViewId.value);
};

// 视图选择变化处理
const onViewIdChange = async (value: string) => {
  targetViewId.value = value;
  await fetchKLineData(targetTableId.value, value);
};

// 字段映射配置变化处理
const onFieldMappingChange = (key: string, value: string) => {
  fieldMapping.value = {
    ...fieldMapping.value,
    [key]: value
  };
  
  if (targetTableId.value) {
    fetchKLineData(targetTableId.value, targetViewId.value);
  }
};

// 样式配置变化处理
const onUpColorChange = (e: any) => {
  upColor.value = e;
  renderKLineChart();
};

const onDownColorChange = (e: any) => {
  downColor.value = e;
  renderKLineChart();
};

const onShowVolumeChange = (checked: boolean) => {
  showVolume.value = checked;
  renderKLineChart();
};

const onShowMAChange = (checked: boolean) => {
  showMA.value = checked;
  renderKLineChart();
};

const onTimeFilterToggle = (checked: boolean) => {
  showTimeFilter.value = checked;
  applyFilters();
};

const onSymbolFilterToggle = (checked: boolean) => {
  showSymbolFilter.value = checked;
  if (!checked) {
    selectedSymbol.value = '';
  }
  applyFilters();
};

// 组件挂载时初始化
onMounted(async () => {
  try {
    state.value = dashboard.state;
    await initTableOptionList();
    
    const theme = await dashboard.getTheme();
    themeString.value = theme.theme;
    themeChartBgColor.value = theme.chartBgColor === "var(--bg-body)" ? "#ffffff" : theme.chartBgColor;
    
    if (state.value !== 'Create') {
      const config = await dashboard.getConfig();
      if (config) {
        await initDashboard(config);
      }
    }
  } catch (error) {
    console.error('初始化失败:', error);
    errorMessage.value = `初始化失败: ${(error as Error).message}`;
  }
});

onUnmounted(() => {
  if (chartInstance) {
    chartInstance.dispose();
  }
  if (currentResizeObserver) {
    currentResizeObserver.disconnect();
  }
});
</script>

<style scoped>
.layout-content {
  display: flex;
  flex-direction: column;
  padding: 20px;
  height: 100%;
  box-sizing: border-box;
}

.kline-container {
  position: relative;
  flex: 1;
  overflow: hidden;
  min-height: 300px;
  background-color: var(--semi-color-bg-0);
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.chart-container {
  position: relative;
  width: 100%;
  height: calc(100% - 40px);
  overflow: hidden;
}

.chart-wrapper {
  width: 100%;
  height: 100%;
}

.filters-container {
  display: flex;
  flex-wrap: wrap;
  gap: 15px;
  padding: 10px;
  background-color: rgba(0, 0, 0, 0.05);
  border-radius: 4px;
  margin-bottom: 10px;
}

.time-filter, .symbol-filter {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  align-items: center;
}

.filter-control {
  display: flex;
  align-items: center;
  gap: 5px;
}

.filter-control label {
  font-size: 14px;
  color: var(--semi-color-text-1);
  font-weight: 500;
  white-space: nowrap;
}

.filter-control input, .filter-control select {
  padding: 8px;
  border: 1px solid var(--semi-color-border);
  border-radius: 4px;
  font-size: 14px;
  background-color: var(--semi-color-bg-0);
  color: var(--semi-color-text-0);
  min-width: 180px;
}

.quick-buttons {
  display: flex;
  gap: 8px;
  margin-left: 8px;
}

.field-mapping {
  margin-top: 8px;
}

.mapping-item {
  margin-bottom: 12px;
}

.mapping-label {
  font-size: 14px;
  color: var(--semi-color-text-1);
  margin-bottom: 4px;
}

.loading-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  background-color: rgba(255, 255, 255, 0.8);
  z-index: 10;
}

.loading-spinner {
  width: 40px;
  height: 40px;
  border: 4px solid rgba(0, 0, 0, 0.1);
  border-radius: 50%;
  border-top: 4px solid var(--semi-color-primary);
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.loading-text {
  margin-top: 12px;
  font-size: 16px;
  color: var(--semi-color-text-1);
  font-weight: 500;
}

.error-message {
  color: #e74c3c;
  padding: 15px;
  border: 1px solid #e74c3c;
  border-radius: 5px;
  background-color: rgba(231, 76, 60, 0.1);
  margin-top: 20px;
  text-align: center;
  font-size: 14px;
}

.no-data {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
  color: var(--semi-color-text-2);
  font-size: 16px;
  text-align: center;
  padding: 20px;
  font-weight: 500;
}

.a-form-item {
  margin: 16px 0;
  padding: 0 20px;
  display: flex;
  flex-direction: column;
}

.lable {
  font-family: "Inter", -apple-system, BlinkMacSystemFont, "Segoe UI", "PingFang SC",
    "Hiragino Sans GB", "Microsoft YaHei", "Helvetica Neue", Helvetica, Arial, sans-serif;
  font-size: 16px;
  font-weight: 500;
  line-height: 22px;
  text-align: left;
  color: var(--semi-color-text-0);
  margin-bottom: 8px;
}

.style-container {
  margin-top: 8px;
  width: calc(100% - 16px);
  padding: 16px 8px;
  gap: 16px;
  border-radius: 6px;
  background-color: rgba(135, 135, 135, 0.08);
  display: flex;
  flex-direction: column;
}

.style-row {
  display: flex;
  gap: 16px;
  width: 100%;
}

.style-item {
  flex: 1;
  display: flex;
  flex-direction: column;
}

.style-label {
  font-size: 14px;
  color: var(--semi-color-text-1);
  margin-bottom: 6px;
}

.colorInput:deep(.semi-input) {
  width: 100%;
  padding: 2px 6px;
  border: none;
  background-color: transparent;
  border-radius: 0;
}

@media (max-width: 768px) {
  .layout-content {
    padding: 10px;
  }
  
  .filters-container {
    flex-direction: column;
  }
  
  .time-filter, .symbol-filter {
    flex-direction: column;
    align-items: flex-start;
  }
  
  .quick-buttons {
    margin-left: 0;
    margin-top: 8px;
    width: 100%;
  }
  
  .style-row {
    flex-direction: column;
    gap: 12px;
  }
  
  .style-item {
    width: 100%;
  }
}

@media (max-width: 480px) {
  .kline-container {
    height: 250px;
  }
  
  .filter-control {
    width: 100%;
  }
}
</style>