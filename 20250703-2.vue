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
          
          <div class="canvas-wrapper" ref="canvasWrapper">
            <canvas ref="klineCanvas"></canvas>
          </div>
          
          <!-- 加载动画 -->
          <div v-if="isLoading" class="loading-overlay">
            <div class="loading-spinner"></div>
            <div class="loading-text">数据加载中...</div>
          </div>
          
          <div v-if="tooltipVisible" class="tooltip" :style="tooltipStyle">
            <div class="tooltip-header">{{ formatDateTime(tooltipData.timestamp) }}</div>
            <div class="tooltip-row"><span>开盘价:</span> {{ tooltipData.open }}</div>
            <div class="tooltip-row"><span>最高价:</span> {{ tooltipData.high }}</div>
            <div class="tooltip-row"><span>最低价:</span> {{ tooltipData.low }}</div>
            <div class="tooltip-row"><span>收盘价:</span> {{ tooltipData.close }}</div>
            <div v-if="showVolume" class="tooltip-row"><span>成交量:</span> {{ tooltipData.volume }}</div>
            <div v-if="tooltipData.symbol" class="tooltip-row"><span>交易品种:</span> {{ tooltipData.symbol }}</div>
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
                    <div class="style-label">K线宽度(%)</div>
                    <InputNumber
                      size="default"
                      style="
                        margin-top: 8px;
                        width: 100%;
                        border: 1px solid var(--semi-color-stroke);
                        border-radius: 6px;
                      "
                      :min="1"
                      :max="100"
                      :value="candleWidthPercent"
                      @change="onCandleWidthChange"
                    />
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
const klineCanvas = ref<HTMLCanvasElement | null>(null);
const canvasWrapper = ref<HTMLDivElement | null>(null);
const klineContainer = ref<HTMLDivElement | null>(null);
const klineData = ref<any[]>([]);
const filteredKlineData = ref<any[]>([]);
const klineCtx = ref<CanvasRenderingContext2D | null>(null);
const tooltipVisible = ref(false);
const tooltipStyle = ref({ top: '0px', left: '0px' });
const tooltipData = ref({
  timestamp: '',
  open: 0,
  high: 0,
  low: 0,
  close: 0,
  volume: 0,
  symbol: ''
});

// 时间筛选相关
const showTimeFilter = ref(false);
const startDate = ref('');
const endDate = ref('');

// 品种筛选相关
const showSymbolFilter = ref(false);
const selectedSymbol = ref('');
const uniqueSymbols = ref<string[]>([]);

// 配置选项
const upColor = ref('#e74c3c'); // 上涨颜色
const downColor = ref('#2ecc71'); // 下跌颜色
const candleWidthPercent = ref(70); // K线宽度百分比
const showVolume = ref(true); // 是否显示成交量

// 错误和加载状态
const isLoading = ref(false);
const errorMessage = ref('');

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
  } else {
    clearCanvas();
  }
});

// 时间格式化函数
const formatDateTime = (timestamp: string | number) => {
  if (!timestamp) return '无时间信息';
  
  // 尝试解析时间戳
  let date: Date;
  if (typeof timestamp === 'string') {
    // 尝试解析ISO格式或数字字符串
    if (/^\d+$/.test(timestamp)) {
      date = new Date(Number(timestamp));
    } else {
      date = new Date(timestamp);
    }
  } else {
    date = new Date(timestamp);
  }
  
  // 检查日期是否有效
  if (isNaN(date.getTime())) {
    return timestamp.toString();
  }
  
  // 格式化为本地时间字符串
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
    
    // 获取字段列表
    if (viewId === 'allData') {
      fieldMetaList = await table.getFieldMetaList();
    } else {
      const view = await table.getViewById(viewId);
      fieldMetaList = await view.getFieldMetaList();
    }
    
    // 创建字段ID到名称的映射（小写）
    const fieldMap = new Map<string, string>();
    for (const field of fieldMetaList) {
      fieldMap.set(field.id, field.name.toLowerCase());
    }
    
    // 分页获取所有记录
    do {
      const response = await table.getRecords({
        pageSize: 5000,
        pageToken,
        viewId: viewId !== 'allData' ? viewId : undefined
      });
      
      records = [...records, ...response.records];
      pageToken = response.pageToken;
    } while (pageToken);
    
    // 处理记录数据为K线格式
    if (records.length > 0) {
      const kData = [];
      const requiredFields = ['timestamp', 'open', 'high', 'low', 'close'];
      const symbolSet = new Set<string>();
      
      for (const record of records) {
        const fields = record.fields;
        const kPoint: any = {};
        let hasRequiredFields = true;
        
        // 处理每个字段
        for (const fieldId in fields) {
          const fieldName = fieldMap.get(fieldId);
          if (!fieldName) continue;
          
          // 只处理需要的字段
          if (requiredFields.includes(fieldName) || 
              ['volume', 'symbol', 'color'].includes(fieldName)) {
            let fieldValue = fields[fieldId];
            
            // 特殊处理交易品种字段：提取文本内容
            if (fieldName === 'symbol') {
              // 如果是数组（富文本数组），提取所有文本内容
              if (Array.isArray(fieldValue)) {
                fieldValue = fieldValue.map(item => item.text || '').join('');
              } 
              // 如果是对象，尝试提取text属性
              else if (fieldValue && typeof fieldValue === 'object' && 'text' in fieldValue) {
                fieldValue = fieldValue.text;
              }
              // 确保是字符串
              fieldValue = String(fieldValue);
            }
            
            kPoint[fieldName] = fieldValue;
            
            // 收集品种
            if (fieldName === 'symbol' && fieldValue) {
              symbolSet.add(fieldValue);
            }
          }
        }
        
        // 检查必需字段是否存在
        for (const field of requiredFields) {
          if (kPoint[field] === undefined || kPoint[field] === null) {
            console.warn(`记录 ${record.recordId} 缺少必填字段: ${field}`);
            hasRequiredFields = false;
            break;
          }
        }
        
        if (hasRequiredFields) {
          // 确保时间戳是数字类型
          if (typeof kPoint.timestamp === 'string' && /^\d+$/.test(kPoint.timestamp)) {
            kPoint.timestamp = Number(kPoint.timestamp);
          }
          kData.push(kPoint);
        }
      }
      
      // 按时间排序
      kData.sort((a, b) => {
        const aTime = new Date(a.timestamp).getTime();
        const bTime = new Date(b.timestamp).getTime();
        return aTime - bTime;
      });
      
      // 检查是否有有效数据
      if (kData.length === 0) {
        errorMessage.value = '没有找到有效数据';
        clearCanvas();
      } else {
        klineData.value = kData;
        uniqueSymbols.value = Array.from(symbolSet).sort();
        
        // 如果没有设置时间范围，则设置默认范围
        if (kData.length > 0 && (!startDate.value || !endDate.value)) {
          const minTime = new Date(kData[0].timestamp);
          const maxTime = new Date(kData[kData.length - 1].timestamp);
          
          // 设置默认时间范围（最近30天）
          const defaultStart = new Date(maxTime);
          defaultStart.setDate(defaultStart.getDate() - 30);
          
          startDate.value = formatDateForInput(defaultStart);
          endDate.value = formatDateForInput(maxTime);
        }
        
        // 应用筛选
        applyFilters();
      }
    } else {
      errorMessage.value = '当前视图没有数据';
      clearCanvas();
    }
  } catch (error) {
    console.error('获取K线数据失败:', error);
    errorMessage.value = `数据加载失败: ${(error as Error).message}`;
    clearCanvas();
  } finally {
    setTimeout(() => {
      isLoading.value = false;
      
      // 数据加载完成后渲染K线图
      if (filteredKlineData.value.length > 0) {
        nextTick(() => {
          renderKLineChart();
          // 重置容器滚动位置
          if (canvasWrapper.value) {
            canvasWrapper.value.scrollLeft = 0;
          }
        });
      } else {
        clearCanvas();
      }
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

// 清空画布
const clearCanvas = () => {
  if (!klineCanvas.value) return;
  
  const canvas = klineCanvas.value;
  const ctx = canvas.getContext('2d');
  
  if (!ctx) return;
  
  // 设置Canvas尺寸为容器尺寸
  const container = canvasWrapper.value;
  if (container) {
    canvas.width = container.clientWidth;
    canvas.height = container.clientHeight;
  }
  
  // 清除Canvas
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  
  // 设置背景色
  ctx.fillStyle = themeChartBgColor.value || '#ffffff';
  ctx.fillRect(0, 0, canvas.width, canvas.height);
  
  // 移除鼠标事件监听
  canvas.onmousemove = null;
  canvas.onmouseleave = null;
  
  // 隐藏工具提示
  tooltipVisible.value = false;
};

// 应用筛选（时间和品种）
const applyFilters = () => {
  let filteredData = [...klineData.value];
  
  // 应用时间筛选
  if (showTimeFilter.value && startDate.value && endDate.value) {
    const start = new Date(startDate.value).getTime();
    const end = new Date(endDate.value).getTime();
    
    filteredData = filteredData.filter(item => {
      const time = new Date(item.timestamp).getTime();
      return time >= start && time <= end;
    });
  }
  
  // 应用品种筛选
  if (showSymbolFilter.value && selectedSymbol.value) {
    filteredData = filteredData.filter(item => {
      // 确保品种字段存在并且匹配
      return item.symbol && String(item.symbol) === selectedSymbol.value;
    });
  }
  
  filteredKlineData.value = filteredData;
  
  // 渲染图表
  renderKLineChart();
  
  // 重置容器滚动位置
  nextTick(() => {
    if (canvasWrapper.value) {
      canvasWrapper.value.scrollLeft = 0;
    }
  });
};

// 重置时间筛选
const resetTimeFilter = () => {
  if (klineData.value.length > 0) {
    const minTime = new Date(klineData.value[0].timestamp);
    const maxTime = new Date(klineData.value[klineData.value.length - 1].timestamp);
    
    startDate.value = formatDateForInput(minTime);
    endDate.value = formatDateForInput(maxTime);
  }
  
  applyFilters();
};

// 重置品种筛选
const resetSymbolFilter = () => {
  selectedSymbol.value = '';
  applyFilters();
};

// 渲染K线图
const renderKLineChart = () => {
  if (!klineCanvas.value || !canvasWrapper.value || filteredKlineData.value.length === 0) {
    clearCanvas();
    return;
  }
  
  const canvas = klineCanvas.value;
  const ctx = canvas.getContext('2d');
  
  if (!ctx) return;
  
  klineCtx.value = ctx;
  
  // 获取容器尺寸
  const container = canvasWrapper.value;
  const containerWidth = container.clientWidth;
  const containerHeight = container.clientHeight;
  
  // 计算需要的canvas宽度（根据数据量）
  const barWidth = 10; // 每根K线的宽度
  const spacing = 2; // K线之间的间距
  const requiredWidth = Math.max(containerWidth, filteredKlineData.value.length * (barWidth + spacing));
  
  // 设置Canvas尺寸
  canvas.width = requiredWidth;
  canvas.height = containerHeight;
  
  // 清除Canvas
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  
  // 设置背景色
  ctx.fillStyle = themeChartBgColor.value || '#ffffff';
  ctx.fillRect(0, 0, canvas.width, canvas.height);
  
  // 计算绘图区域
  const margin = { top: 30, right: 30, bottom: showVolume.value ? 80 : 50, left: 60 };
  const width = canvas.width - margin.left - margin.right;
  const height = canvas.height - margin.top - margin.bottom;
  
  // 计算价格范围
  let minPrice = Infinity;
  let maxPrice = -Infinity;
  let maxVolume = 0;
  
  for (const data of filteredKlineData.value) {
    minPrice = Math.min(minPrice, data.low);
    maxPrice = Math.max(maxPrice, data.high);
    if (data.volume && showVolume.value) {
      maxVolume = Math.max(maxVolume, data.volume);
    }
  }
  
  // 添加10%的缓冲空间
  const priceRange = maxPrice - minPrice;
  minPrice = minPrice - priceRange * 0.1;
  maxPrice = maxPrice + priceRange * 0.1;
  
  // 计算K线宽度和间距
  const actualBarWidth = Math.max(1, (width / filteredKlineData.value.length) * (candleWidthPercent.value / 100));
  const barSpacing = (width / filteredKlineData.value.length) * ((100 - candleWidthPercent.value) / 100);
  
  // 创建价格比例尺
  const priceScale = (value: number) => {
    return margin.top + height - ((value - minPrice) / (maxPrice - minPrice)) * height;
  };
  
  // 创建时间比例尺
  const timeScale = (index: number) => {
    return margin.left + index * (actualBarWidth + barSpacing) + barSpacing / 2;
  };
  
  // 绘制网格
  ctx.strokeStyle = themeString.value === 'DARK' ? '#444' : '#eee';
  ctx.lineWidth = 1;
  
  // 水平网格线
  const priceSteps = 5;
  for (let i = 0; i <= priceSteps; i++) {
    const y = margin.top + (i / priceSteps) * height;
    ctx.beginPath();
    ctx.moveTo(margin.left, y);
    ctx.lineTo(margin.left + width, y);
    ctx.stroke();
    
    // 价格标签
    const priceValue = minPrice + (maxPrice - minPrice) * (1 - i / priceSteps);
    ctx.fillStyle = themeString.value === 'DARK' ? '#aaa' : '#666';
    ctx.font = '12px Arial';
    ctx.textAlign = 'right';
    ctx.fillText(priceValue.toFixed(2), margin.left - 5, y + 4);
  }
  
  // 绘制K线
  for (let i = 0; i < filteredKlineData.value.length; i++) {
    const data = filteredKlineData.value[i];
    const x = timeScale(i);
    const openY = priceScale(data.open);
    const closeY = priceScale(data.close);
    const highY = priceScale(data.high);
    const lowY = priceScale(data.low);
    
    // 修复颜色问题：收盘价高于开盘价为上涨，反之为下跌
    const isUp = data.close > data.open;
    const color = data.color || (isUp ? upColor.value : downColor.value);
    
    // 绘制影线（最高价到最低价的垂直线）
    ctx.strokeStyle = color;
    ctx.lineWidth = 1;
    ctx.beginPath();
    ctx.moveTo(x + actualBarWidth / 2, highY);
    ctx.lineTo(x + actualBarWidth / 2, lowY);
    ctx.stroke();
    
    // 绘制实体（开盘价到收盘价的矩形）
    const bodyHeight = Math.abs(openY - closeY);
    if (bodyHeight > 0) {
      ctx.fillStyle = color;
      ctx.fillRect(x, Math.min(openY, closeY), actualBarWidth, bodyHeight);
    } else {
      // 当开盘价等于收盘价时，绘制一条横线
      ctx.strokeStyle = color;
      ctx.lineWidth = 1;
      ctx.beginPath();
      ctx.moveTo(x, openY);
      ctx.lineTo(x + actualBarWidth, openY);
      ctx.stroke();
    }
    
    // 如果存在成交量，绘制成交量柱状图
    if (data.volume && maxVolume > 0 && showVolume.value) {
      const volumeHeight = (data.volume / maxVolume) * (height * 0.2);
      ctx.fillStyle = themeString.value === 'DARK' ? 'rgba(100, 100, 100, 0.5)' : 'rgba(200, 200, 200, 0.5)';
      ctx.fillRect(x, margin.top + height - volumeHeight, actualBarWidth, volumeHeight);
    }
    
    // 每10根K线绘制一个时间标签
    if (i % 10 === 0) {
      const date = new Date(data.timestamp);
      const dateStr = `${date.getMonth() + 1}/${date.getDate()}`;
      ctx.fillStyle = themeString.value === 'DARK' ? '#aaa' : '#666';
      ctx.font = '10px Arial';
      ctx.textAlign = 'center';
      ctx.fillText(dateStr, x + actualBarWidth / 2, canvas.height - 10);
    }
  }
  
  // 添加鼠标事件监听
  canvas.onmousemove = handleMouseMove;
  canvas.onmouseleave = () => {
    tooltipVisible.value = false;
  };
  
  // 添加滚动条指示器
  if (canvas.width > container.clientWidth) {
    const scrollRatio = container.clientWidth / canvas.width;
    const scrollPosition = container.scrollLeft / canvas.width;
    
    ctx.fillStyle = 'rgba(100, 100, 100, 0.5)';
    ctx.fillRect(
      container.clientWidth * scrollPosition,
      container.clientHeight - 5,
      container.clientWidth * scrollRatio,
      5
    );
  }
};

// 处理鼠标移动事件 - 专门为仪表盘组件优化
const handleMouseMove = (event: MouseEvent) => {
  if (!klineCanvas.value || !canvasWrapper.value || filteredKlineData.value.length === 0) return;
  
  const canvas = klineCanvas.value;
  const container = canvasWrapper.value;
  const rect = canvas.getBoundingClientRect();
  
  // 获取容器滚动偏移量
  const scrollLeft = container.scrollLeft;
  
  // 计算鼠标在canvas上的绝对位置（考虑滚动）
  const mouseX = (event.clientX - rect.left + scrollLeft);
  
  // 计算绘图区域
  const margin = { top: 30, right: 30, bottom: showVolume.value ? 80 : 50, left: 60 };
  const width = canvas.width - margin.left - margin.right;
  
  // 计算K线宽度和间距
  const actualBarWidth = Math.max(1, (width / filteredKlineData.value.length) * (candleWidthPercent.value / 100));
  const barSpacing = (width / filteredKlineData.value.length) * ((100 - candleWidthPercent.value) / 100);
  const barTotalWidth = actualBarWidth + barSpacing;
  
  // 计算鼠标位置对应的K线索引 - 使用更精确的计算方法
  const index = Math.min(
    filteredKlineData.value.length - 1,
    Math.max(0, Math.floor((mouseX - margin.left) / barTotalWidth))
  );
  
  if (index >= 0 && index < filteredKlineData.value.length) {
    const data = filteredKlineData.value[index];
    
    // 更新tooltip数据
    tooltipData.value = {
      timestamp: data.timestamp,
      open: data.open,
      high: data.high,
      low: data.low,
      close: data.close,
      volume: data.volume || 0,
      symbol: data.symbol || ''
    };
    
    // 计算tooltip位置（确保不会超出屏幕）
    const tooltipWidth = 220;
    const tooltipHeight = 180;
    
    // 使用当前鼠标位置作为基准
    let left = event.clientX + 15;
    let top = event.clientY + 15;
    
    // 检查右边界
    if (left + tooltipWidth > window.innerWidth) {
      left = event.clientX - tooltipWidth - 15;
    }
    
    // 检查下边界
    if (top + tooltipHeight > window.innerHeight) {
      top = event.clientY - tooltipHeight - 15;
    }
    
    // 更新tooltip位置
    tooltipStyle.value = {
      left: `${left}px`,
      top: `${top}px`
    };
    
    tooltipVisible.value = true;
  } else {
    tooltipVisible.value = false;
  }
};

// 保存配置
const saveConfig = async () => {
  if (!isConfigValid.value) return;
  
  const config = {
    customConfig: {
      tableId: targetTableId.value,
      viewId: targetViewId.value,
      upColor: upColor.value,
      downColor: downColor.value,
      candleWidthPercent: candleWidthPercent.value,
      showVolume: showVolume.value,
      showTimeFilter: showTimeFilter.value,
      showSymbolFilter: showSymbolFilter.value,
      selectedSymbol: selectedSymbol.value,
      startDate: startDate.value,
      endDate: endDate.value,
    },
  };
  
  await dashboard.saveConfig(config);
  
  // 更新状态
  state.value = dashboard.state;
  
  // 立即应用所有配置更改
  applyAllConfigChanges();
};

// 应用所有配置更改
const applyAllConfigChanges = () => {
  // 应用筛选
  applyFilters();
  
  // 强制更新视图
  nextTick(() => {
    if (canvasWrapper.value) {
      canvasWrapper.value.scrollLeft = 0;
    }
    
    if (klineCanvas.value && canvasWrapper.value) {
      klineCanvas.value.width = canvasWrapper.value.clientWidth;
      klineCanvas.value.height = canvasWrapper.value.clientHeight;
    }
    
    renderKLineChart();
  });
};

// 初始化仪表盘
const initDashboard = async (config: any) => {
  const customConfig = config.customConfig || {};
  
  // 从配置中恢复所有设置
  targetTableId.value = customConfig.tableId || '';
  targetViewId.value = customConfig.viewId || 'allData';
  upColor.value = customConfig.upColor || '#e74c3c';
  downColor.value = customConfig.downColor || '#2ecc71';
  candleWidthPercent.value = customConfig.candleWidthPercent || 70;
  showVolume.value = customConfig.showVolume !== undefined ? customConfig.showVolume : true;
  showTimeFilter.value = customConfig.showTimeFilter !== undefined ? customConfig.showTimeFilter : false;
  showSymbolFilter.value = customConfig.showSymbolFilter !== undefined ? customConfig.showSymbolFilter : false;
  selectedSymbol.value = customConfig.selectedSymbol || '';
  
  // 恢复时间筛选范围
  startDate.value = customConfig.startDate || '';
  endDate.value = customConfig.endDate || '';
  
  // 加载视图列表
  if (targetTableId.value) {
    viewOptionList.value = await queryViewOptionList(targetTableId.value);
  }
  
  // 如果有表ID，则加载数据
  if (targetTableId.value) {
    await fetchKLineData(targetTableId.value, targetViewId.value);
  } else {
    // 如果没有表ID，尝试渲染（可能已有数据）
    if (filteredKlineData.value.length > 0) {
      nextTick(renderKLineChart);
    }
  }
};

// 数据表选择变化处理
const onTableIdChange = async (value: string) => {
  targetTableId.value = value;
  viewOptionList.value = await queryViewOptionList(value);
  targetViewId.value = "allData";
  await fetchKLineData(value, targetViewId.value);
};

// 视图选择变化处理
const onViewIdChange = async (value: string) => {
  targetViewId.value = value;
  await fetchKLineData(targetTableId.value, value);
};

// 样式配置变化处理
const onUpColorChange = (e: any) => {
  upColor.value = e;
  if (filteredKlineData.value.length > 0) renderKLineChart();
};

const onDownColorChange = (e: any) => {
  downColor.value = e;
  if (filteredKlineData.value.length > 0) renderKLineChart();
};

const onCandleWidthChange = (value: number) => {
  candleWidthPercent.value = value;
  if (filteredKlineData.value.length > 0) renderKLineChart();
};

const onShowVolumeChange = (checked: boolean) => {
  showVolume.value = checked;
  if (filteredKlineData.value.length > 0) renderKLineChart();
};

const onTimeFilterToggle = (checked: boolean) => {
  showTimeFilter.value = checked;
  applyFilters();
};

const onSymbolFilterToggle = (checked: boolean) => {
  showSymbolFilter.value = checked;
  if (!checked) {
    // 关闭品种筛选时重置选择
    selectedSymbol.value = '';
  }
  applyFilters();
};

// 调整大小处理函数
const handleResize = () => {
  if (filteredKlineData.value.length > 0) {
    renderKLineChart();
  } else {
    clearCanvas();
  }
};

// 设置数据监听
const setupDataListeners = async () => {
  if (!targetTableId.value) return;
  
  try {
    const table = await bitable.base.getTable(targetTableId.value);
    
    // 监听记录变化
    table.onRecordAdd(async (event) => {
      console.log('记录添加:', event);
      await fetchKLineData(targetTableId.value, targetViewId.value);
    });
    
    table.onRecordModify(async (event) => {
      console.log('记录修改:', event);
      await fetchKLineData(targetTableId.value, targetViewId.value);
    });
    
    table.onRecordDelete(async (event) => {
      console.log('记录删除:', event);
      await fetchKLineData(targetTableId.value, targetViewId.value);
    });
    
    console.log('数据监听器已设置');
  } catch (error) {
    console.error('设置数据监听失败:', error);
  }
};

// 组件挂载时初始化
onMounted(async () => {
  try {
    state.value = dashboard.state;
    await initTableOptionList();
    
    // 获取主题
    const theme = await dashboard.getTheme();
    themeString.value = theme.theme;
    themeChartBgColor.value = theme.chartBgColor === "var(--bg-body)" ? "#ffffff" : theme.chartBgColor;
    
    // 如果不是创建状态，尝试加载保存的配置
    if (state.value !== 'Create') {
      const config = await dashboard.getConfig();
      if (config) {
        await initDashboard(config);
      } else {
        // 如果没有配置，尝试初始渲染
        nextTick(() => {
          if (filteredKlineData.value.length > 0) {
            renderKLineChart();
          }
        });
      }
    }
    
    // 添加窗口大小变化监听
    window.addEventListener('resize', handleResize);
    
    // 初始清空画布
    clearCanvas();
  } catch (error) {
    console.error('初始化失败:', error);
    errorMessage.value = `初始化失败: ${(error as Error).message}`;
    clearCanvas();
  }
});

// 当表格或视图变化时设置监听
watch([targetTableId, targetViewId], () => {
  if (targetTableId.value) {
    setupDataListeners();
  }
});

// 状态变化时重新渲染
watch(state, (newState) => {
  if (newState === STATE_ARRAY[2] || newState === STATE_ARRAY[3]) {
    if (filteredKlineData.value.length > 0) {
      nextTick(renderKLineChart);
    }
  }
});

onUnmounted(() => {
  // 移除事件监听
  window.removeEventListener('resize', handleResize);
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

/* 关键修改：解决滚动时tooltip错位问题 */
.canvas-wrapper {
  position: relative;
  width: 100%;
  height: calc(100% - 40px);
  overflow-x: auto;
  overflow-y: hidden;
}

.kline-container canvas {
  display: block;
  height: 100%;
}

/* 关键修改：确保tooltip在滚动时位置正确 */
.tooltip {
  position: fixed;
  z-index: 1000;
  background-color: rgba(0, 0, 0, 0.85);
  color: white;
  border-radius: 6px;
  padding: 12px 16px;
  font-size: 14px;
  pointer-events: none;
  backdrop-filter: blur(4px);
  min-width: 220px;
  box-shadow: 0 6px 16px rgba(0, 0, 0, 0.2);
  border: 1px solid rgba(255, 255, 255, 0.1);
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

.tooltip-header {
  font-weight: 600;
  margin-bottom: 8px;
  padding-bottom: 6px;
  font-size: 16px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.2);
}

.tooltip-row {
  display: flex;
  justify-content: space-between;
  margin-bottom: 8px;
  line-height: 1.5;
}

.tooltip-row span {
  font-weight: 500;
  margin-right: 10px;
  color: rgba(255, 255, 255, 0.7);
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
  
  .tooltip {
    font-size: 12px;
    padding: 10px;
    min-width: 180px;
  }
  
  .filter-control {
    width: 100%;
  }
}
</style>