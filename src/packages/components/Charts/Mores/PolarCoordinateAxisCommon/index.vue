<template>
<div>
  <div ref="vChartRef" id="chart" class="chart" style="width: 100%;height: 100%;"></div>
</div>

</template>

<script setup lang="ts">
import {ref, nextTick, computed, watch, PropType, onMounted, onBeforeUnmount, onUpdated, toRefs} from 'vue'

import { VChart as VIChart,IVChart, IRoseChartSpec} from '@visactor/vchart';
// 引入柱状图
import { registerBarChart } from '@visactor/vchart';
// 引入坐标轴、Tooltip、CrossHair组件
import { registerTooltip, registerCartesianCrossHair, registerDomTooltipHandler } from '@visactor/vchart';

// 注册图表和组件
VIChart.useRegisters([registerBarChart, registerTooltip, registerDomTooltipHandler, registerCartesianCrossHair]);

import { use } from 'echarts/core'
import { CanvasRenderer } from 'echarts/renderers'
import { BarChart } from 'echarts/charts'
import config, { includes, seriesItem } from './config'
import { mergeTheme } from '@/packages/public/chart'
import { useChartDataFetch } from '@/hooks'
import { CreateComponentType } from '@/packages/index.d'
import { useChartEditStore } from '@/store/modules/chartEditStore/chartEditStore'
import { isPreview } from '@/utils'
import { DatasetComponent, GridComponent, TooltipComponent, LegendComponent } from 'echarts/components'
import isObject from 'lodash/isObject'
//
// const props = defineProps({
//   themeSetting: {
//     type: Object,
//     required: true
//   },
//   themeColor: {
//     type: Object,
//     required: true
//   },
//   chartConfig: {
//     type: Object as PropType<config>,
//     required: true
//   },
//   colors:{
//     type: String[],
//     required: true
//   }
// })

use([DatasetComponent, CanvasRenderer, BarChart, GridComponent, TooltipComponent, LegendComponent])

const vChartRef = ref<HTMLElement>()
const replaceMergeArr = ref<string[]>()
//
// const option = computed(() => {
//   return mergeTheme(props.chartConfig.option, props.themeSetting, includes)
// })
interface LineChartProps {
  color?: string[]

}
const props = defineProps({
  chartConfig: {
    type: Object as PropType<CreateComponentType>,
    required: true
  },
    themeSetting: {
    type: Object,
    required: true
  },
  themeColor: {
    type: Object,
    required: true
  },
  lineChartProps:{
    type: Object as PropType<IRoseChartSpec>,
    required:true
  }
})

let chart: IVChart;

let {
  background
}  = toRefs(props.chartConfig.option.lineOptions)


let {
  legendsVisible,
  legendsPosition,
  legendsOrient
}  = toRefs(props.chartConfig.option.lineOptions.legends)

const parseSpec = (chartProps: LineChartProps) => {
  // console.log(chartProps)
  // console.log(props.themeColor)
  // console.log(props.chartConfig.option.dataset.data)
  // console.log(props.chartConfig.option.lineOptions.legends)
  // console.log(legendsOrient.value)
  const colors = chartProps.color ?? [
    "#6690F2",
    "#70D6A3",
    "#B4E6E2",
    "#63B5FC",
    "#FF8F62",
    "#FFDC83",
    "#BCC5FD",
    "#A29BFE",
    "#63C4C7",
    "#F68484",
  ];
  return {
    type: "rose",
    data: props.chartConfig.option.dataset.source,
    color: {
      type: "ordinal",
      domain: [],
      range: colors,
    },

    background:background.value,
    padding: {
      top: 50
    },
    radius: 1,
    innerRadius: 0.6,
    categoryField: ['State', 'type'],
    valueField: 'population',
    seriesField: 'age',
    stack: true,
    axes: [
      {
        orient: 'radius',
        visible: true,
        label: {
          visible: true,
          formatMethod: (val:any) => {
            return val / 1000000 + 'M';
          },
          style: {
            fill: 'rgba(182,182,182,0.5)'
          }
        },
        grid: {
          visible: true,
          smooth: true,
          style: {
            lineDash: [0]
          }
        },
        // title: {
        //   visible: true,
        //   position: 'end',
        //   text: 'population',
        //   style: {
        //     textAlign: 'start',
        //     dy: -10,
        //     dx: -10
        //   }
        // }
      },
      {
        orient: 'angle',
        visible: true,
        inside: true, // 将圆弧轴显示在内圆
        // 配置圆弧轴的标题
        title: {
          visible: true,
          position: 'center', // 标题显示位置为圆心
          text: 'State',
          padding: 4,
          background: {
            visible: true,
            style: {
              fill: 'rgba(0,0,0,0.5)',
              cornerRadius: 4
            }
          }
        },
        tick: {
          visible: true,
          style: {
            stroke: '#ffffff'
          }
        },
        domainLine: {
          visible: true
        },
        label: {
          visible: true,
          style: {
            fill: 'rgba(255,255,255,0.85)'
          }
        }
      }
    ],
    legends: [{ visible: legendsVisible.value, position: legendsPosition.value, orient: legendsOrient.value }],
  } as IRoseChartSpec;
}

const createOrUpdateChart = (chartProps: LineChartProps) => {
  if (vChartRef.value && !chart) {
    chart = new VIChart(parseSpec(props.themeColor), {
      dom: vChartRef.value,
    });

    chart.renderAsync();
    return true;
  } else if (chart) {
    chart.updateSpec(parseSpec(props.themeColor));
    chart.renderAsync();

    return true;
  }
  return false;
}

onMounted(() => {
  // console.log(props.lineChartProps)
  createOrUpdateChart(props.chartConfig.option.lineOptions);
});

onUpdated(() => {
  createOrUpdateChart(props.chartConfig.option.lineOptions);
});

onBeforeUnmount(() => {
  if (chart) {
    chart.release();
  }
});


const stopWatch = watch(
    () => props.chartConfig.option.lineOptions,
    option => {
      createOrUpdateChart(props.chartConfig.option.lineOptions);
    },
    {
      immediate: true,
      deep: true
    }
)


// dataset 无法变更条数的补丁
watch(
  () => props.chartConfig.option.dataset,
  (newData: { dimensions: any }, oldData) => {
    try {
      if (!isObject(newData) || !('dimensions' in newData)) return
      if (Array.isArray(newData?.dimensions)) {
        const seriesArr = []
        for (let i = 0; i < newData.dimensions.length - 1; i++) {
          seriesArr.push(seriesItem)
        }
        replaceMergeArr.value = ['series']
        props.chartConfig.option.series = seriesArr
        nextTick(() => {
          replaceMergeArr.value = []
        })
      }
    } catch (error) {
      console.log(error)
    }
  },
  {
    deep: false
  }
)

// const { vChartRef } = useChartDataFetch(props.chartConfig, useChartEditStore)
</script>
