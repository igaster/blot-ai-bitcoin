<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { Line } from 'vue-chartjs';
import { Chart as ChartJS, CategoryScale, LinearScale, PointElement, LineElement, Title, Tooltip, Legend, ChartOptions } from 'chart.js';
import annotationPlugin from 'chartjs-plugin-annotation';
import type { ChartData } from 'chart.js';
import axios from 'axios';

ChartJS.register(CategoryScale, LinearScale, PointElement, LineElement, Title, Tooltip, Legend, annotationPlugin);

const selectedPeriod = ref('30d');

const chartData = ref<ChartData<'line'>>({
  labels: [],
  datasets: [{
    label: 'Bitcoin Price (USD)',
    data: [],
    borderColor: '#3b82f6',
    tension: 0.1,
    pointRadius: (context: any) => {
      const index = context.dataIndex;
      return growthPeriod.value.includes(index) ? 8 : 0;
    },
    pointBackgroundColor: (context: any) => {
      const index = context.dataIndex;
      return growthPeriod.value.includes(index) ? '#22c55e' : '#3b82f6';
    },
    pointBorderColor: (context: any) => {
      const index = context.dataIndex;
      return growthPeriod.value.includes(index) ? '#22c55e' : '#3b82f6';
    }
  }]
});

const growthPeriod = ref<number[]>([]);
const growthDetails = ref({
  startDate: '',
  days: 0,
  percentageGrowth: 0
});

const chartOptions = ref<ChartOptions<'line'>>({
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: {
      position: 'top' as const,
      labels: {
        color: () => document.documentElement.classList.contains('dark') ? '#fff' : '#000'
      }
    },
    title: {
      display: true,
      text: 'Bitcoin Price History',
      color: () => document.documentElement.classList.contains('dark') ? '#fff' : '#000'
    },
    tooltip: {
      callbacks: {
        label: (context: any) => {
          const index = context.dataIndex;
          let label = `Price: $${context.parsed.y.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })}`;
          if (growthPeriod.value.includes(index)) {
            label += ' 🔼';
          }
          return label;
        }
      }
    },
    annotation: {
      annotations: {
        growthLabel: {
          type: 'label',
          drawTime: 'afterDatasetsDraw',
          content: 'Longest Growth Period',
          color: '#22c55e',
          display: true,
          font: {
            size: 14,
            weight: 'bold'
          },
          position: {
            x: 'center',
            y: 'center'
          }
        }
      }
    }
  },
  scales: {
    x: {
      title: {
        display: true,
        text: 'Date',
        color: () => document.documentElement.classList.contains('dark') ? '#fff' : '#000'
      },
      ticks: {
        color: () => document.documentElement.classList.contains('dark') ? '#fff' : '#000',
        maxRotation: 45,
        minRotation: 45
      },
      grid: {
        color: () => document.documentElement.classList.contains('dark') ? '#374151' : '#e5e7eb'
      }
    },
    y: {
      title: {
        display: true,
        text: 'Price (USD)',
        color: () => document.documentElement.classList.contains('dark') ? '#fff' : '#000'
      },
      ticks: {
        color: () => document.documentElement.classList.contains('dark') ? '#fff' : '#000',
        callback: (value: number) => {
          return '$' + value.toLocaleString('en-US', { minimumFractionDigits: 0, maximumFractionDigits: 0 });
        }
      },
      grid: {
        color: () => document.documentElement.classList.contains('dark') ? '#374151' : '#e5e7eb'
      }
    }
  }
});

const loading = ref(true);
const error = ref('');

function findLongestGrowthPeriod(data: number[]): [number, number] {
  let currentStart = 0;
  let currentLength = 1;
  let maxStart = 0;
  let maxLength = 1;

  for (let i = 1; i < data.length; i++) {
    if (data[i] > data[i - 1]) {
      currentLength++;
      if (currentLength > maxLength) {
        maxLength = currentLength;
        maxStart = currentStart;
      }
    } else {
      currentStart = i;
      currentLength = 1;
    }
  }

  return [maxStart, maxStart + maxLength - 1];
}

async function fetchBitcoinData() {
  try {
    loading.value = true;
    error.value = '';
    
    const periodDays = selectedPeriod.value === '30d' ? 30 : 90;
    const response = await axios.get(
      `https://api.coingecko.com/api/v3/coins/bitcoin/market_chart?vs_currency=usd&days=${periodDays}&interval=daily`
    );
    
    const priceData = response.data.prices;
    const dates = priceData.map((price: number[]) => new Date(price[0]));
    chartData.value.labels = dates.map((date: Date) => 
      date.toLocaleDateString('en-US', { month: 'short', day: 'numeric' })
    );
    const prices = priceData.map((price: number[]) => price[1]);
    chartData.value.datasets[0].data = prices;

    // Find and set the growth period
    const [startIndex, endIndex] = findLongestGrowthPeriod(prices);
    growthPeriod.value = [startIndex, endIndex];

    // Calculate growth details
    const startPrice = prices[startIndex];
    const endPrice = prices[endIndex];
    const percentageGrowth = ((endPrice - startPrice) / startPrice) * 100;
    const growthDays = endIndex - startIndex + 1;
    
    growthDetails.value = {
      startDate: dates[startIndex].toLocaleDateString('en-US', { month: 'long', day: 'numeric', year: 'numeric' }),
      days: growthDays,
      percentageGrowth
    };

    // Update annotation position
    if (chartOptions.value.plugins?.annotation?.annotations?.growthLabel) {
      const annotation = chartOptions.value.plugins.annotation.annotations.growthLabel as any;
      annotation.xValue = chartData.value.labels[Math.floor((startIndex + endIndex) / 2)];
      annotation.yValue = (prices[startIndex] + prices[endIndex]) / 2;
    }

    loading.value = false;
  } catch (e) {
    error.value = 'Failed to fetch Bitcoin data';
    loading.value = false;
  }
}

function changePeriod(period: string) {
  selectedPeriod.value = period;
  fetchBitcoinData();
}

onMounted(() => {
  fetchBitcoinData();
});
</script>

<template>
  <div class="w-full max-w-4xl mx-auto p-4">
    <div class="mb-4 flex justify-center space-x-4">
      <button 
        @click="changePeriod('30d')"
        :class="[
          'px-4 py-2 rounded-lg transition-colors',
          selectedPeriod === '30d'
            ? 'bg-blue-500 text-white'
            : 'bg-gray-200 dark:bg-gray-700 text-gray-800 dark:text-white hover:bg-gray-300 dark:hover:bg-gray-600'
        ]"
      >
        Last 30 Days
      </button>
      <button 
        @click="changePeriod('90d')"
        :class="[
          'px-4 py-2 rounded-lg transition-colors',
          selectedPeriod === '90d'
            ? 'bg-blue-500 text-white'
            : 'bg-gray-200 dark:bg-gray-700 text-gray-800 dark:text-white hover:bg-gray-300 dark:hover:bg-gray-600'
        ]"
      >
        Last 3 Months
      </button>
    </div>
    <div v-if="loading" class="text-center py-8">
      <div class="text-lg dark:text-white">Loading...</div>
    </div>
    <div v-else-if="error" class="text-center py-8">
      <div class="text-red-500">{{ error }}</div>
    </div>
    <div v-else class="bg-white dark:bg-gray-800 rounded-lg shadow-lg p-6 transition-colors duration-200">
      <div class="h-[400px]">
        <Line :data="chartData" :options="chartOptions" />
      </div>
      <div class="mt-4 text-center text-sm text-gray-600 dark:text-gray-300">
        <p class="font-medium">Longest Growth Period:</p>
        <p>Starting {{ growthDetails.startDate }} ({{ growthDetails.days }} days)</p>
        <p>Total Growth: {{ growthDetails.percentageGrowth.toFixed(2) }}%</p>
      </div>
    </div>
  </div>
</template>