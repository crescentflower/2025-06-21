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
      <div
        v-if="state === STATE_ARRAY[1] || state === STATE_ARRAY[0]"
        style="width: 100%; height: 1px; background-color: var(--semi-color-stroke); position: absolute;"
        :style="{ backgroundColor: themeChartBgColor }"
      ></div>
      
      <LayoutContent
        class="layout-content"
        :style="{
          backgroundColor: themeChartBgColor,
          width: state === STATE_ARRAY[2] || state === STATE_ARRAY[3] ? '100%' : '100%',
          height: state === STATE_ARRAY[2] || state === STATE_ARRAY[3] ? '100%' : '100vh'
        }"
      >
        <div class="chart-container" ref="chartContainer">
          <canvas ref="klineCanvas" class="kline-canvas"></canvas>
          
          <div v-if="tooltipVisible" class="tooltip" :style="tooltipStyle">
            <h4>K线详情</h4>
            <div class="tooltip-row">
              <span class="tooltip-label">时间:</span>
              <span class="tooltip-value">{{ tooltipData.timestamp }}</span>
            </div>
            <div class="tooltip-row">
              <span class="tooltip-label">开盘:</span>
              <span class="tooltip-value">{{ tooltipData.open }}</span>
            </div>
            <div class="tooltip-row">
              <span class="tooltip-label">最高:</span>
              <span class="tooltip-value">{{ tooltipData.high }}</span>
            </div>
            <div class="tooltip-row">
              <span class="tooltip-label">最低:</span>
              <span class="tooltip-value">{{ tooltipData.low }}</span>
            </div>
            <div class="tooltip-row">
              <span class="tooltip-label">收盘:</span>
              <span class="tooltip-value">{{ tooltipData.close }}</span>
            </div>
            <div class="tooltip-row" v-if="tooltipData.volume">
              <span class="tooltip-label">成交量:</span>
              <span class="tooltip-value">{{ tooltipData.volume }}</span>
            </div>
            <div class="tooltip-row">
              <span class="tooltip-label">涨跌幅:</span>
              <span class="tooltip-value" :style="{ color: tooltipData.change >= 0 ? '#26A69A' : '#EF5350' }">
                {{ tooltipData.change >= 0 ? '+' : '' }}{{ tooltipData.change }}%
              </span>
            </div>
          </div>
          
          <div v-if="loading" class="loading-overlay">
            <div class="loading-spinner"></div>
            <div class="loading-text">正在加载K线数据...</div>
          </div>
          
          <div v-if="errorMessage" class="error-overlay">
            <div class="error-icon">⚠️</div>
            <div class="error-title">数据加载失败</div>
            <div class="error-detail">{{ errorMessage }}</div>
            <button class="btn btn-primary" @click="retryLoading" style="margin-top: 20px;">重试</button>
          </div>
        </div>
      </LayoutContent>

      <!-- 配置区域 -->
      <LayoutSider
        v-if="state === STATE_ARRAY[1] || state === STATE_ARRAY[0]"
        :class="{
          'layout-sider': true,
          'semi-always-dark': themeString === 'DARK',
          'semi-always-light': themeString === 'LIGHT'
        }"
        style="height: 100%; border-left: 1px solid var(--semi-color-stroke); flex: 0 0 340px;"
      >
        <div class="formPanel" style="width: 100%; height: 100%; display: flex; flex-direction: column">
          <div class="form" style="width: 100%; height: 100%; display: flex; flex-direction: column; overflow: auto;">
            <!-- 数据源选择 -->
            <a-form-item class="a-form-item">
              <div class="lable">选择数据表</div>
              <Select
                :optionList="tableOptionList"
                :value="targetTableId"
                @change="onTableIdChange"
                style="margin-top: 8px; width: 100%; background-color: transparent; border: 1px solid var(--semi-color-stroke); border-radius: 6px;"
              />
            </a-form-item>
            
            <!-- 视图选择 -->
            <a-form-item class="a-form-item">
              <div class="lable">选择视图</div>
              <Select
                :optionList="viewOptionList"
                :value="targetViewId"
                @change="onViewIdChange"
                style="margin-top: 8px; width: 100%; background-color: transparent; border: 1px solid var(--semi-color-stroke); border-radius: 6px;"
              />
            </a-form-item>
            
            <!-- 样式配置 -->
            <a-form-item class="a-form-item">
              <div class="lable">图表样式</div>
              <div class="style-container">
                <div>
                  <b-form-item class="b-form-item">
                    <div>上涨颜色</div>
                    <Input
                      class-name="backGroundColorInput"
                      size="default"
                      type="color"
                      style="margin-top: 8px; height: 34px; border: 1px solid var(--semi-color-stroke); border-radius: 6px;"
                      :value="upColor"
                      @change="onUpColorChange"
                    />
                  </b-form-item>
                  
                  <b-form-item class="b-form-item">
                    <div>下跌颜色</div>
                    <Input
                      class-name="backGroundColorInput"
                      size="default"
                      type="color"
                      style="margin-top: 8px; height: 34px; border: 1px solid var(--semi-color-stroke); border-radius: 6px;"
                      :value="downColor"
                      @change="onDownColorChange"
                    />
                  </b-form-item>
                </div>
                
                <div>
                  <b-form-item class="b-form-item">
                    <div>K线宽度</div>
                    <InputNumber
                      size="default"
                      style="margin-top: 8px; width: 100%; border: 1px solid var(--semi-color-stroke); border-radius: 6px;"
                      :min="1"
                      :max="20"
                      :value="candleWidth"
                      @change="onCandleWidthChange"
                    />
                  </b-form-item>
                  
                  <b-form-item class="b-form-item">
                    <div>时间范围</div>
                    <Select
                      :optionList="timeRangeOptions"
                      :value="timeRange"
                      @change="onTimeRangeChange"
                      style="margin-top: 8px; width: 100%; height: 34px; border: 1px solid var(--semi-color-stroke); border-radius: 6px;"
                    />
                  </b-form-item>
                </div>
              </div>
            </a-form-item>
          </div>
          
          <a-form-item class="a-form-item" style="height: 70px; display: flex; flex-direction: row-reverse">
            <Button
              @click="saveConfig"
              theme="solid"
              type="primary"
              style="width: 80px; border-radius: 4px"
            >确定</Button>
          </a-form-item>
        </div>
      </LayoutSider>
    </Layout>
  </div>
</template>

<script lang="ts" setup>
import { ref, onMounted, computed, nextTick } from 'vue';
import { dashboard, bitable, FieldType } from '@lark-base-open/js-sdk';
import { Layout, LayoutSider, LayoutContent, Button, Select, InputNumber, Input, Spin } from '@kousum/semi-ui-vue';

const STATE_ARRAY = ['Create', 'Config', 'View', 'FullScreen'];
type OptionItem = { label: string; value: string };

// 主题相关
const themeChartBgColor = ref('');
const themeString = ref('');

// 状态管理
const state = ref('Create');

// 数据源配置
const tableOptionList = ref<OptionItem[]>([]);
const viewOptionList = ref<OptionItem[]>([]);
const targetTableId = ref('');
const targetViewId = ref('');

// 样式配置
const upColor = ref('#26A69A');
const downColor = ref('#EF5350');
const candleWidth = ref(10);
const timeRange = ref('1M');
const timeRangeOptions = ref([
  { label: '1天', value: '1D' },
  { label: '1周', value: '1W' },
  { label: '1月', value: '1M' },
  { label: '3月', value: '3M' },
  { label: '1年', value: '1Y' }
]);

// 图表数据
const klineData = ref<any[]>([]);
const loading = ref(true);
const errorMessage = ref('');
const tooltipVisible = ref(false);
const tooltipData = ref({});
const tooltipStyle = ref({
  left: '0px',
  top: '0px',
  backgroundColor: 'rgba(0,0,0,0.85)',
  padding: '12px',
  borderRadius: '6px',
  color: '#fff',
  position: 'absolute',
  zIndex: 100
});

// 图表元素引用
const chartContainer = ref<HTMLElement | null>(null);
const klineCanvas = ref<HTMLCanvasElement | null>(null);
let ctx: CanvasRenderingContext2D | null = null;

// 主题变化监听
dashboard.onThemeChange(res => {
  themeChartBgColor.value = res.data.chartBgColor;
  if (res.data.chartBgColor === "var(--bg-body)") {
    themeChartBgColor.value = "#ffffff";
  }
  themeString.value = res.data.theme;
});

// 初始化表格选项 - 修复后的函数
const initTableOptionList = async () => {
  try {
    const tableList = await bitable.base.getTableList();
    tableOptionList.value = tableList.map(table => ({
      label: table.name,
      value: table.id
    }));
    
    console.log('获取到的表格列表:', tableOptionList.value);
    
    // 在创建状态下设置默认表格
    if (state.value === 'Create' && tableList.length > 0) {
      targetTableId.value = tableList[0].id;
      console.log('设置默认表格ID:', targetTableId.value);
      
      // 加载该表格的视图
      await loadViewList(targetTableId.value);
    }
  } catch (error) {
    console.error('获取表格列表失败:', error);
    errorMessage.value = '获取表格列表失败，请刷新重试';
  }
};

// 加载视图列表 - 修复后的函数
const loadViewList = async (tableId: string) => {
  try {
    if (!tableId) {
      console.warn('没有提供表格ID，无法加载视图列表');
      return;
    }
    
    const table = bitable.base.getTable(tableId);
    const viewList = await table.getViewList();
    
    console.log(`表格 ${tableId} 的视图列表:`, viewList);
    
    viewOptionList.value = viewList.map(view => ({
      label: view.name,
      value: view.id
    }));
    
    // 设置默认视图
    if (viewOptionList.value.length > 0) {
      targetViewId.value = viewOptionList.value[0].value;
      console.log('设置默认视图ID:', targetViewId.value);
    }
  } catch (error) {
    console.error('获取视图列表失败:', error);
    errorMessage.value = '获取视图列表失败';
  }
};

// 生成示例数据
const generateSampleData = () => {
  const data = [];
  const now = Date.now();
  const oneDay = 24 * 60 * 60 * 1000;
  
  for (let i = 0; i < 60; i++) {
    const open = 100 + Math.random() * 10;
    const close = open + (Math.random() - 0.5) * 10;
    const high = Math.max(open, close) + Math.random() * 5;
    const low = Math.min(open, close) - Math.random() * 5;
    const volume = Math.floor(Math.random() * 1000);
    const change = ((close - open) / open * 100).toFixed(2);
    
    data.push({
      timestamp: new Date(now - i * oneDay).toISOString(),
      open,
      high,
      low,
      close,
      volume,
      change
    });
  }
  
  return data;
};

// 绘制K线图
const drawChart = () => {
  if (!klineCanvas.value || !ctx || klineData.value.length === 0) return;
  
  const canvas = klineCanvas.value;
  const width = canvas.width;
  const height = canvas.height;
  const padding = {
    top: 40,
    right: 40,
    bottom: 60,
    left: 70
  };
  
  // 清除画布
  ctx.clearRect(0, 0, width, height);
  
  // 设置背景色
  ctx.fillStyle = themeString.value === 'DARK' ? '#1a1a1a' : '#ffffff';
  ctx.fillRect(0, 0, width, height);
  
  // 计算价格范围
  let min = Infinity;
  let max = -Infinity;
  
  klineData.value.forEach(data => {
    if (data.low < min) min = data.low;
    if (data.high > max) max = data.high;
  });
  
  // 添加余量
  const range = max - min;
  min -= range * 0.05;
  max += range * 0.05;
  
  // 计算坐标比例
  const availableWidth = width - padding.left - padding.right;
  const availableHeight = height - padding.top - padding.bottom;
  const xScale = availableWidth / klineData.value.length;
  const yScale = availableHeight / (max - min);
  
  // 绘制网格
  ctx.strokeStyle = themeString.value === 'DARK' ? '#333333' : '#e0e0e0';
  ctx.lineWidth = 0.5;
  
  // 水平网格线
  const gridLines = 5;
  for (let i = 0; i <= gridLines; i++) {
    const y = padding.top + i * (availableHeight / gridLines);
    ctx.beginPath();
    ctx.moveTo(padding.left, y);
    ctx.lineTo(width - padding.right, y);
    ctx.stroke();
    
    // 价格标签
    const price = (max - i * (range / gridLines)).toFixed(2);
    ctx.fillStyle = themeString.value === 'DARK' ? '#cccccc' : '#5f6368';
    ctx.font = "12px Arial";
    ctx.textAlign = "right";
    ctx.fillText(price, padding.left - 10, y + 4);
  }
  
  // 垂直网格线
  const verticalLines = Math.min(10, klineData.value.length);
  for (let i = 0; i <= verticalLines; i++) {
    const index = Math.floor(i * (klineData.value.length / verticalLines));
    const x = padding.left + index * xScale;
    ctx.beginPath();
    ctx.moveTo(x, padding.top);
    ctx.lineTo(x, height - padding.bottom);
    ctx.stroke();
  }
  
  // 绘制K线
  klineData.value.forEach((data, i) => {
    const x = padding.left + i * xScale + xScale / 2;
    
    // 计算坐标
    const highY = padding.top + (max - data.high) * yScale;
    const lowY = padding.top + (max - data.low) * yScale;
    const openY = padding.top + (max - data.open) * yScale;
    const closeY = padding.top + (max - data.close) * yScale;
    
    // 确定颜色
    const color = data.close >= data.open ? upColor.value : downColor.value;
    
    // 绘制影线
    ctx.strokeStyle = color;
    ctx.lineWidth = 1;
    ctx.beginPath();
    ctx.moveTo(x, highY);
    ctx.lineTo(x, lowY);
    ctx.stroke();
    
    // 绘制实体
    ctx.fillStyle = color;
    const candleTop = Math.min(openY, closeY);
    const candleHeight = Math.abs(openY - closeY);
    
    // 实体宽度
    const candleWidthVal = Math.max(1, candleWidth.value * 0.7);
    
    ctx.fillRect(
      x - candleWidthVal / 2,
      candleTop,
      candleWidthVal,
      Math.max(candleHeight, 1)
    );
  });
  
  // 绘制坐标轴
  ctx.strokeStyle = themeString.value === 'DARK' ? '#ffffff' : '#333333';
  ctx.lineWidth = 1.5;
  
  // Y轴
  ctx.beginPath();
  ctx.moveTo(padding.left, padding.top);
  ctx.lineTo(padding.left, height - padding.bottom);
  ctx.stroke();
  
  // X轴
  ctx.beginPath();
  ctx.moveTo(padding.left, height - padding.bottom);
  ctx.lineTo(width - padding.right, height - padding.bottom);
  ctx.stroke();
  
  // 标题
  ctx.fillStyle = themeString.value === 'DARK' ? '#64b5f6' : '#1a73e8';
  ctx.font = "bold 16px Arial";
  ctx.textAlign = "center";
  ctx.fillText("K线图", width / 2, padding.top - 15);
  
  // 时间标签
  if (klineData.value.length > 0) {
    const firstDate = new Date(klineData.value[0].timestamp);
    const lastDate = new Date(klineData.value[klineData.value.length - 1].timestamp);
    
    ctx.fillStyle = themeString.value === 'DARK' ? '#cccccc' : '#5f6368';
    ctx.font = "12px Arial";
    ctx.textAlign = "left";
    ctx.fillText(formatDate(firstDate), padding.left, height - padding.bottom + 20);
    
    ctx.textAlign = "right";
    ctx.fillText(formatDate(lastDate), width - padding.right, height - padding.bottom + 20);
  }
};

// 处理鼠标移动事件
const handleMouseMove = (event: MouseEvent) => {
  if (!klineCanvas.value || !klineData.value.length) return;
  
  const rect = klineCanvas.value.getBoundingClientRect();
  const x = event.clientX - rect.left;
  
  // 计算鼠标下的K线索引
  const padding = { left: 70, right: 40 };
  const availableWidth = klineCanvas.value.width - padding.left - padding.right;
  const index = Math.floor((x - padding.left) / (availableWidth / klineData.value.length));
  
  if (index >= 0 && index < klineData.value.length) {
    tooltipData.value = klineData.value[index];
    tooltipVisible.value = true;
    
    // 更新提示框位置
    tooltipStyle.value = {
      ...tooltipStyle.value,
      left: `${event.clientX + 15}px`,
      top: `${event.clientY + 15}px`
    };
  } else {
    tooltipVisible.value = false;
  }
};

// 格式化日期
const formatDate = (date: string | Date) => {
  if (!date) return "";
  const d = new Date(date);
  return `${d.getFullYear()}-${(d.getMonth() + 1).toString().padStart(2, "0")}-${d.getDate().toString().padStart(2, "0")}`;
};

// 保存配置
const saveConfig = async () => {
  try {
    const config = {
      dataConditions: {
        tableId: targetTableId.value,
        viewId: targetViewId.value
      },
      customConfig: {
        upColor: upColor.value,
        downColor: downColor.value,
        candleWidth: candleWidth.value,
        timeRange: timeRange.value
      }
    };
    
    await dashboard.saveConfig(config);
    
    // 通知宿主渲染完成
    await dashboard.setRendered();
    
    // 如果是创建状态，切换到展示状态
    if (state.value === 'Create') {
      state.value = 'View';
    }
    
    // 重新获取数据
    fetchTableData();
    
  } catch (error) {
    console.error('保存配置失败:', error);
    errorMessage.value = '保存配置失败: ' + error.message;
  }
};

// 初始化画布
const initCanvas = () => {
  if (!klineCanvas.value || !chartContainer.value) return;
  
  const canvas = klineCanvas.value;
  const container = chartContainer.value;
  
  // 确保容器有尺寸
  if (container.clientWidth > 0 && container.clientHeight > 0) {
    canvas.width = container.clientWidth;
    canvas.height = container.clientHeight;
    ctx = canvas.getContext('2d');
    
    // 添加鼠标事件
    canvas.addEventListener('mousemove', handleMouseMove);
    canvas.addEventListener('mouseleave', () => {
      tooltipVisible.value = false;
    });
    
    // 立即绘制图表
    drawChart();
  }
};

// 重试加载
const retryLoading = () => {
  errorMessage.value = '';
  fetchTableData();
};

// 获取表格数据 - 只在展示状态下调用
const fetchTableData = async () => {
  try {
    // 在配置状态下使用示例数据
    if (state.value !== 'View' && state.value !== 'FullScreen') {
      klineData.value = generateSampleData();
      loading.value = false;
      nextTick(() => {
        initCanvas();
        drawChart();
      });
      return;
    }

    loading.value = true;
    errorMessage.value = '';
    
    // 如果没有选择表格，使用示例数据
    if (!targetTableId.value) {
      klineData.value = generateSampleData();
      loading.value = false;
      nextTick(() => {
        initCanvas();
        drawChart();
      });
      return;
    }
    
    // 获取表格
    const table = bitable.base.getTable(targetTableId.value);
    
    // 获取视图中的记录
    const view = await table.getViewById(targetViewId.value);
    const records = await view.getRecords();
    
    // 尝试自动识别字段
    const fields = await table.getFieldList();
    
    // 寻找可能的字段
    const timestampField = fields.find(f => 
      f.name.includes('时间') || f.name.includes('日期') || f.name.includes('timestamp')
    );
    
    const openField = fields.find(f => 
      f.name.includes('开盘') || f.name.includes('open')
    );
    
    const highField = fields.find(f => 
      f.name.includes('最高') || f.name.includes('high')
    );
    
    const lowField = fields.find(f => 
      f.name.includes('最低') || f.name.includes('low')
    );
    
    const closeField = fields.find(f => 
      f.name.includes('收盘') || f.name.includes('close')
    );
    
    const volumeField = fields.find(f => 
      f.name.includes('成交量') || f.name.includes('volume')
    );
    
    // 处理数据
    const result = [];
    for (const record of records) {
      const data: any = {};
      
      if (timestampField) {
        const value = await record.getCellValue(timestampField.id);
        if (value) {
          data.timestamp = new Date(value).toISOString();
        }
      }
      
      if (openField) {
        const value = await record.getCellValue(openField.id);
        if (value) {
          data.open = parseFloat(value);
        }
      }
      
      if (highField) {
        const value = await record.getCellValue(highField.id);
        if (value) {
          data.high = parseFloat(value);
        }
      }
      
      if (lowField) {
        const value = await record.getCellValue(lowField.id);
        if (value) {
          data.low = parseFloat(value);
        }
      }
      
      if (closeField) {
        const value = await record.getCellValue(closeField.id);
        if (value) {
          data.close = parseFloat(value);
        }
      }
      
      if (volumeField) {
        const value = await record.getCellValue(volumeField.id);
        if (value) {
          data.volume = parseFloat(value);
        }
      }
      
      // 计算涨跌幅
      if (data.open && data.close) {
        data.change = ((data.close - data.open) / data.open * 100).toFixed(2);
      }
      
      // 确保有足够的数据
      if (data.open !== undefined && data.high !== undefined && 
          data.low !== undefined && data.close !== undefined) {
        result.push(data);
      }
    }
    
    // 按时间排序
    result.sort((a, b) => new Date(a.timestamp).getTime() - new Date(b.timestamp).getTime());
    
    // 根据时间范围过滤数据
    const now = Date.now();
    const rangeMap = {
      '1D': 24 * 60 * 60 * 1000,
      '1W': 7 * 24 * 60 * 60 * 1000,
      '1M': 30 * 24 * 60 * 60 * 1000,
      '3M': 90 * 24 * 60 * 60 * 1000,
      '1Y': 365 * 24 * 60 * 60 * 1000
    };
    
    if (timeRange.value in rangeMap) {
      const timeAgo = now - rangeMap[timeRange.value];
      klineData.value = result.filter(item => new Date(item.timestamp).getTime() > timeAgo);
    } else {
      klineData.value = result;
    }
    
  } catch (error) {
    console.error('获取数据失败:', error);
    errorMessage.value = '获取数据失败: ' + error.message;
    klineData.value = generateSampleData();
  } finally {
    loading.value = false;
    nextTick(() => {
      initCanvas();
      drawChart();
    });
  }
};

// 事件处理函数
const onTableIdChange = async (value: string) => {
  targetTableId.value = value;
  // 加载该表格的视图
  await loadViewList(value);
  // 刷新数据
  fetchTableData();
};

const onViewIdChange = (value: string) => {
  targetViewId.value = value;
  // 刷新数据
  fetchTableData();
};

const onUpColorChange = (value: string) => {
  upColor.value = value;
  drawChart();
};

const onDownColorChange = (value: string) => {
  downColor.value = value;
  drawChart();
};

const onCandleWidthChange = (value: number) => {
  candleWidth.value = value;
  drawChart();
};

const onTimeRangeChange = (value: string) => {
  timeRange.value = value;
  fetchTableData();
};

// 组件挂载时初始化 - 修复后的逻辑
onMounted(async () => {
  try {
    // 获取当前状态
    state.value = dashboard.state;
    console.log('当前插件状态:', state.value);
    
    // 获取主题
    const theme = await dashboard.getTheme();
    themeString.value = theme.theme;
    
    // 初始化表格选项 - 确保这一步最先执行
    await initTableOptionList();
    
    // 如果是创建状态，使用示例数据
    if (state.value === 'Create') {
      klineData.value = generateSampleData();
      loading.value = false;
    } else {
      // 尝试加载配置
      try {
        const config = await dashboard.getConfig();
        console.log('加载到的配置:', config);
        
        if (config.customConfig) {
          targetTableId.value = config.dataConditions?.tableId || '';
          targetViewId.value = config.dataConditions?.viewId || '';
          upColor.value = config.customConfig.upColor || '#26A69A';
          downColor.value = config.customConfig.downColor || '#EF5350';
          candleWidth.value = config.customConfig.candleWidth || 10;
          timeRange.value = config.customConfig.timeRange || '1M';
          
          console.log('从配置加载的表格ID:', targetTableId.value);
          console.log('从配置加载的视图ID:', targetViewId.value);
          
          // 加载视图列表
          if (targetTableId.value) {
            await loadViewList(targetTableId.value);
          }
        }
      } catch (error) {
        console.error('加载配置失败:', error);
      }
    }
    
    // 获取数据
    await fetchTableData();
    
    // 确保DOM渲染完成
    nextTick(() => {
      // 初始化画布
      initCanvas();
      
      // 监听窗口大小变化
      window.addEventListener('resize', () => {
        initCanvas();
        drawChart();
      });
    });
    
    // 监听数据变化
    dashboard.onDataChange(() => {
      fetchTableData();
    });
    
  } catch (error) {
    console.error('初始化失败:', error);
    errorMessage.value = '初始化失败: ' + error.message;
    klineData.value = generateSampleData();
    nextTick(() => {
      initCanvas();
      drawChart();
    });
  }
});
</script>

<style scoped>
.layout-container {
  transition: background-color 0.3s;
}

.layout-content {
  display: flex;
  align-items: center;
  justify-content: center;
}

.chart-container {
  position: relative;
  width: 100%;
  height: 100%;
  min-height: 300px; /* 确保有最小高度 */
}

.kline-canvas {
  width: 100%;
  height: 100%;
  display: block;
  background-color: transparent;
}

.tooltip {
  position: absolute;
  background: rgba(0, 0, 0, 0.85);
  color: white;
  padding: 12px;
  border-radius: 6px;
  font-size: 14px;
  pointer-events: none;
  z-index: 100;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  min-width: 200px;
  backdrop-filter: blur(4px);
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.tooltip h4 {
  margin-bottom: 8px;
  padding-bottom: 6px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.2);
  color: #64b5f6;
}

.tooltip-row {
  display: flex;
  justify-content: space-between;
  margin-bottom: 4px;
}

.tooltip-label {
  color: #bbbbbb;
}

.tooltip-value {
  font-weight: 500;
}

.loading-overlay, .error-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  background-color: rgba(255, 255, 255, 0.9);
  z-index: 10;
}

.loading-spinner {
  width: 50px;
  height: 50px;
  border: 4px solid rgba(26, 115, 232, 0.2);
  border-top: 4px solid #1a73e8;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin-bottom: 20px;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.loading-text {
  font-size: 1.1rem;
  color: #333;
  font-weight: 500;
}

.error-overlay {
  background-color: #ffebee;
}

.error-icon {
  font-size: 3rem;
  color: #d32f2f;
  margin-bottom: 15px;
}

.error-title {
  font-size: 1.3rem;
  color: #b71c1c;
  margin-bottom: 10px;
  font-weight: 600;
}

.error-detail {
  color: #333;
  text-align: center;
  max-width: 80%;
  line-height: 1.5;
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
  font-size: 14px;
  font-weight: 400;
  line-height: 22px;
  text-align: left;
  color: var(--semi-color-text-0);
}

.style-container {
  margin-top: 8px;
  width: calc(100% - 16px);
  padding: 16px 8px;
  gap: 20px;
  border-radius: 6px;
  background-color: rgba(135, 135, 135, 0.08);
  display: flex;
  flex-direction: column;
}

.style-container > div {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
}

.style-container > div > .b-form-item {
  width: 48%;
  font-size: 12px !important;
  color: var(--semi-color-text-1);
}

.btn {
  padding: 10px 20px;
  border-radius: 6px;
  border: none;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

.btn-primary {
  background: linear-gradient(135deg, #1a73e8 0%, #0d47a1 100%);
  color: white;
}

.btn-primary:hover {
  background: linear-gradient(135deg, #0d47a1 0%, #1a73e8 100%);
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(26, 115, 232, 0.3);
}
</style>