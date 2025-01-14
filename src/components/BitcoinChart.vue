<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { Line } from 'vue-chartjs';
import { Chart as ChartJS, CategoryScale, LinearScale, PointElement, LineElement, Title, Tooltip, Legend } from 'chart.js';
import axios from 'axios';

ChartJS.register(CategoryScale, LinearScale, PointElement, LineElement, Title, Tooltip, Legend);

const chartData = ref({
  labels: [],
  datasets: [{
    label: 'Bitcoin Price (USD)',
    data: [],
    borderColor: '#3b82f6',
    tension: 0.1
  }]
});

const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: {
      position: 'top' as const,
    },
    title: {
      display: true,
      text: 'Bitcoin Price History'
    }
  }
};

const loading = ref(true);
const error = ref('');

async function fetchBitcoinData() {
  try {
    const response = await axios.get(
      'https://api.coingecko.com/api/v3/coins/bitcoin/market_chart?vs_currency=usd&days=30&interval=daily'
    );
    
    const priceData = response.data.prices;
    chartData.value.labels = priceData.map((price: number[]) => 
      new Date(price[0]).toLocaleDateString()
    );
    chartData.value.datasets[0].data = priceData.map((price: number[]) => price[1]);
    loading.value = false;
  } catch (e) {
    error.value = 'Failed to fetch Bitcoin data';
    loading.value = false;
  }
}

onMounted(() => {
  fetchBitcoinData();
});
</script>

<template>
  <div class="w-full max-w-4xl mx-auto p-4">
    <div v-if="loading" class="text-center py-8">
      <div class="text-lg">Loading...</div>
    </div>
    <div v-else-if="error" class="text-center py-8">
      <div class="text-red-500">{{ error }}</div>
    </div>
    <div v-else class="bg-white rounded-lg shadow-lg p-6">
      <div class="h-[400px]">
        <Line :data="chartData" :options="chartOptions" />
      </div>
    </div>
  </div>
</template>