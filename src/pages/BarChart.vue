<template>
  <div
    ref="form"
    class="flex flex-center"
    style="height: 100vh;"
  >
    <q-card
      style="width: 500px;"
    >
      <q-card-section>Bar Chart</q-card-section>
      <q-card-section>
        <q-item>
          <q-file
            v-model="file"
            class="col-10"
            label="Pick one file"
            filled
            clearable
            style="max-width: 300px"
          />
          <q-btn
            class="col on-right"
            color="primary"
            label="Upload"
            @click="upload"
          />
        </q-item>
        <q-item>
          <q-select
            v-model="granularity"
            class="col"
            :options="granularityOptions"
            label="Granularity"
            emit-value
            map-options
            @update:model-value="onGranularityChange"
          />
        </q-item>
        <q-item>
          <q-select
            v-model="groupingBy"
            class="col"
            :options="groupingByOptions"
            label="Group By"
            emit-value
            map-options
            @update:model-value="onGroupingByChange"
          />
        </q-item>
        <q-item>
          <q-select
            v-model="chartMode"
            class="col"
            :options="chartModeOptions"
            label="Chart Mode"
            emit-value
            map-options
            @update:model-value="onChartModeChange"
          />
        </q-item>
        <q-item>
          <q-input
            v-model.number="maxLegendNum"
            class="col"
            label="Max Legend Num"
            @update:model-value="onMaxLegendNumChange"
          />
        </q-item>
      </q-card-section>
    </q-card>
  </div>

  <div
    class="bg-grey-1 flex flex-center"
  >
    <div
      id="chart"
      ref="chart"
      style="width: 50rem; height: calc(100vh - 50px);"
    />

    <q-btn
      class="absolute-bottom-right q-ma-md"
      rounded
      icon="arrow_upward"
      @click="scrollIntoView(form)"
    />
  </div>
</template>

<script setup>
import { onMounted, onRenderTracked, onRenderTriggered, ref } from 'vue'
import { debounce } from 'quasar'
import * as echarts from 'echarts'
import dayjs from 'dayjs'

const BASE_GRANULARITY = 60 / 5 // 5 minutes

const form = ref(null)
const chart = ref(null)

function scrollIntoView (element) {
  element.scrollIntoView({ behavior: 'smooth' })
}

// Create the echarts instance
let myChart = null

const file = ref(null)

const granularity = ref(1)
const granularityOptions = [
  { label: '1 hr', value: 1 },
  { label: '3 hrs', value: 3 },
  { label: '6 hrs', value: 6 },
  { label: '1 day', value: 24 }
]
function onGranularityChange (value) {
  const aggregatedData = aggregateData(groupedData.value, value)
  setAggregatedData(aggregatedData)
  const xAxisData = genXAxisData(aggregatedData)
  setXAsisData(xAxisData)
  const chartData = prepareChartData(aggregatedData, chartMode.value, maxLegendNum.value)
  setChartData(chartData)

  myChart.clear()
  drawChart(xAxisData, chartData)
}

const groupingBy = ref('user')
const groupingByOptions = [
  { label: 'User', value: 'user' },
  { label: 'Address', value: 'address' },
  { label: 'Country', value: 'country' },
  { label: 'Service', value: 'service' }
]
function onGroupingByChange (value) {
  const groupedData = groupBy(parsedData.value, value)
  setGroupedData(groupedData)
  const aggregatedData = aggregateData(groupedData, granularity.value)
  setAggregatedData(aggregatedData)
  const xAxisData = genXAxisData(aggregatedData)
  setXAsisData(xAxisData)
  const chartData = prepareChartData(aggregatedData, chartMode.value, maxLegendNum.value)
  setChartData(chartData)

  myChart.clear()
  drawChart(xAxisData, chartData)
}

const chartMode = ref('group')
const chartModeOptions = [
  { label: 'Group', value: 'group' },
  { label: 'Stack', value: 'stack' }
]
function onChartModeChange (value) {
  const chartData = prepareChartData(aggregatedData.value, value, maxLegendNum.value)
  setChartData(chartData)

  drawChart(xAxisData.value, chartData)
}

const maxLegendNum = ref(10)
const onMaxLegendNumChange = debounce((value) => {
  const chartData = prepareChartData(aggregatedData.value, chartMode.value, value)
  setChartData(chartData)

  myChart.clear()
  drawChart(xAxisData.value, chartData)
}, 1000)

/**
 * Parses the input file text and returns an array of objects representing the data.
 * Each object contains a date and a value.
 *
 * @param {string} fileText - The text content of the input file.
 * @returns {Array<object>} - An array of objects representing the data.
 */
function parseInputCsvFile (fileText) {
  const lines = fileText.trim().split('\n')

  const data = lines.map((line) => {
    /**
     * Note: The following code is not robust and assumes that the input file is always in the same format.
     * Actually, the when a csv field contains a comma, it is enclosed in double quotes.
     * But the input file does not follow this rule.
     */
    const splitedData = line.split(',').map((item) => item.trim())
    const [date, user, address, ...rest] = splitedData
    const [service, bps] = rest.slice(-2)
    const country = rest.slice(0, -2).join(', ')

    return {
      date: dayjs(date).valueOf(),
      user,
      address,
      country,
      service,
      bps: Number(bps)
    }
  })

  return data
}

/**
 * Groups an array of objects by a specific key.
 *
 * @param {Array<object>} array - The array to group.
 * @param {string} key - The key to group the array by.
 * @returns {Object} An object with keys being the unique values of the specified key,
 *  and values being arrays of objects that have that key's value.
 */
function groupBy (array, key) {
  return array.reduce((result, currentValue) => {
    const groupKey = currentValue[key]
    if (!result[groupKey]) {
      result[groupKey] = []
    }
    result[groupKey].push(currentValue)
    return result
  }, {})
}

/**
 * Aggregates data based on a certain granularity.
 *
 * @param {Object} groupedData - The data to be aggregated. Each key-value pair represents a group of data.
 * @param {number} granularity - The size of each group in the aggregation.
 *
 * @returns {Object} The aggregated data. Each key-value pair represents a group of aggregated data.
 */
function aggregateData (groupedData, granularity) {
  const aggregatedData = Object.entries(groupedData).reduce((result, [key, value]) => {
    const aggregatedValue = value.reduce((result, currentValue) => {
      const date = currentValue.date
      const bps = currentValue.bps
      const dateKey = Math.floor(date / (granularity * 60 * 60 * 1000))
      if (!result[dateKey]) {
        result[dateKey] = {
          date: dateKey * granularity * 60 * 60 * 1000,
          bps: 0
        }
      }
      result[dateKey].bps += bps
      return result
    }, {})
    result[key] = Object.values(aggregatedValue).map(({ date, bps }) => {
      const adjustedBps = bps / (BASE_GRANULARITY * granularity)
      const roundedBps = adjustedBps.toFixed(2)
      return { date, bps: roundedBps }
    })
    return result
  }, {})
  return aggregatedData
}

/**
 * Generates the data for the x-axis of a bar chart.
 *
 * @param {Object} aggregatedData - The aggregated data from which to generate the x-axis data.
 *  Each key-value pair represents a group of aggregated data.
 *
 * @returns {Array} The x-axis data.
 *  Each element is a string representing a date and time in the format 'YYYY-MM-DD HH:mm:ss'.
 */
function genXAxisData (aggregatedData) {
  const firstGroupKey = Object.keys(aggregatedData)[0]
  const firstGroupData = aggregatedData[firstGroupKey]
  const xAxisData = firstGroupData?.map(item => dayjs(item.date).format('YYYY-MM-DD HH:mm:ss')) || []
  return xAxisData
}

/**
 * Prepares data for use in a bar chart.
 *
 * @param {Object} aggregatedData - The aggregated data to be prepared.
 *  Each key-value pair represents a group of aggregated data.
 * @param {string} chartMode - The chart mode.
 *
 * @returns {Array} The prepared chart data. Each element is an object representing a series in the bar chart.
 */
function prepareChartData (aggregatedData, chartMode, maxLegendNum = 10) {
  const chartData = Object.entries(aggregatedData).slice(0, maxLegendNum).map(([key, value]) => {
    return {
      name: key,
      type: 'bar',
      stack: chartMode === 'stack' ? 'user' : null,
      data: value.map((item) => item.bps)
    }
  })
  return chartData
}

const parsedData = ref([])
function setParsedData (value) {
  parsedData.value = value
}

const groupedData = ref([])
function setGroupedData (value) {
  groupedData.value = value
}

const aggregatedData = ref([])
function setAggregatedData (value) {
  aggregatedData.value = value
}

const xAxisData = ref([])
function setXAsisData (value) {
  xAxisData.value = value
}

const chartData = ref([])
function setChartData (value) {
  chartData.value = value
}

/**
 * Uploads the file and logs its value.
 *
 * @returns {void}
 */
function upload () {
  if (file.value) {
    const reader = new FileReader()

    reader.onload = function readerOnLoad (e) {
      const parsedData = parseInputCsvFile(e?.target?.result?.toString() ?? '')
      setParsedData(parsedData)
      const groupedData = groupBy(parsedData, groupingBy.value)
      setGroupedData(groupedData)
      const aggregatedData = aggregateData(groupedData, granularity.value)
      setAggregatedData(aggregatedData)
      const xAxisData = genXAxisData(aggregatedData)
      setXAsisData(xAxisData)
      const chartData = prepareChartData(aggregatedData, chartMode.value, maxLegendNum.value)
      setChartData(chartData)

      drawChart(xAxisData, chartData)
      scrollIntoView(chart.value)
    }

    reader.readAsText(file.value)
  }
}

function drawChart (xAxisData, chartData) {
  try {
    // Draw the chart
    myChart.setOption({
      title: {
        // text: 'ECharts Getting Started Example'
      },
      tooltip: {},
      legend: {
        // Try 'horizontal'
        orient: 'vertical',
        right: 10,
        top: 'center',
        selected: chartData.reduce((result, currentValue, index) => {
          result[currentValue.name] = index < 5
          return result
        }, {})
      },
      xAxis: {
        data: xAxisData
      },
      yAxis: {
        type: 'value',
        axisLabel: {
          formatter: function (value, index) {
            // bps to kbps or mbps or gbps and so on...
            const units = ['bps', 'kbps', 'mbps', 'gbps', 'tbps']
            let unitIndex = 0
            let adjustedValue = value
            while (adjustedValue >= 1000) {
              adjustedValue /= 1000
              unitIndex++
            }
            return `${adjustedValue.toFixed(2)} ${units[unitIndex]}`
          }
        }
      },
      series: chartData
    })
  } catch (error) {
    console.error('🚀 - error:', error)
  }
}

onMounted(() => {
  myChart = echarts.init(document.getElementById('chart'))
})

onRenderTracked((event) => {
  debugger
})

onRenderTriggered((event) => {
  debugger
})
</script>
