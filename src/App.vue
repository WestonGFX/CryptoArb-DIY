<template>
      <div id="app">
        <h1>Crypto Trading and Arbitrage</h1>
        <div class="container">
          <div class="trading-section">
            <h2>Trading</h2>
            <div class="trade-form">
              <label for="crypto-select">Select Crypto:</label>
              <select id="crypto-select" v-model="selectedCrypto">
                <option v-for="crypto in cryptos" :key="crypto">{{ crypto }}</option>
              </select>
              <label for="amount-input">Amount:</label>
              <input type="number" id="amount-input" v-model.number="tradeAmount" />
              <button @click="executeTrade">Trade</button>
            </div>
            <div class="trade-history">
              <h3>Trade History</h3>
              <ul>
                <li v-for="trade in tradeHistory" :key="trade.id">
                  {{ trade.crypto }} - {{ trade.amount }} - {{ trade.timestamp }}
                </li>
              </ul>
            </div>
          </div>
          <div class="arbitrage-section">
            <h2>Arbitrage Opportunities</h2>
            <div v-if="arbitrageOpportunities.length > 0">
              <ul>
                <li v-for="opportunity in arbitrageOpportunities" :key="opportunity.id">
                  {{ opportunity.exchange1 }} - {{ opportunity.exchange2 }} - {{ opportunity.profit }}
                </li>
              </ul>
            </div>
            <div v-else>
              <p>No arbitrage opportunities found.</p>
            </div>
          </div>
          <div class="chart-section">
            <h2>Price Chart</h2>
            <canvas ref="chartCanvas"></canvas>
          </div>
        </div>
      </div>
    </template>

    <script>
    import { ref, onMounted, watch } from 'vue';
    import axios from 'axios';
    import { Database } from 'libsql';
    import { Chart, registerables } from 'chart.js';
    Chart.register(...registerables);

    export default {
      setup() {
        const cryptos = ref(['BTC', 'ETH', 'LTC']);
        const selectedCrypto = ref('BTC');
        const tradeAmount = ref(0);
        const tradeHistory = ref([]);
        const arbitrageOpportunities = ref([]);
        const chartCanvas = ref(null);
        let chartInstance = null;

        const db = new Database('/home/project/trades.db');

        const initDatabase = async () => {
          await db.exec('CREATE TABLE IF NOT EXISTS trades (id INTEGER PRIMARY KEY AUTOINCREMENT, crypto TEXT, amount REAL, timestamp TEXT)');
        };

        const fetchTradeHistory = async () => {
          const trades = await db.all('SELECT * FROM trades');
          tradeHistory.value = trades;
        };

        const executeTrade = async () => {
          const timestamp = new Date().toISOString();
          await db.run('INSERT INTO trades (crypto, amount, timestamp) VALUES (?, ?, ?)', [selectedCrypto.value, tradeAmount.value, timestamp]);
          fetchTradeHistory();
        };

        const fetchArbitrageOpportunities = async () => {
          try {
            const exchange1 = 'binance';
            const exchange2 = 'coinbase';
            const response1 = await axios.get(`https://api.binance.com/api/v3/ticker/price?symbol=${selectedCrypto.value}USDT`);
            const response2 = await axios.get(`https://api.coinbase.com/v2/prices/${selectedCrypto.value}-USD/spot`);

            const price1 = parseFloat(response1.data.price);
            const price2 = parseFloat(response2.data.data.amount);

            const profit = price2 - price1;

            if (Math.abs(profit) > 0.1) {
              arbitrageOpportunities.value = [{
                id: Date.now(),
                exchange1: exchange1,
                exchange2: exchange2,
                profit: profit.toFixed(2)
              }];
            } else {
              arbitrageOpportunities.value = [];
            }
          } catch (error) {
            console.error('Error fetching arbitrage data:', error);
            arbitrageOpportunities.value = [];
          }
        };

        const fetchChartData = async () => {
          try {
            const response = await axios.get(`https://api.binance.com/api/v3/klines?symbol=${selectedCrypto.value}USDT&interval=1h&limit=24`);
            const prices = response.data.map(item => parseFloat(item[4]));
            const labels = response.data.map(item => new Date(item[0]).toLocaleTimeString());

            if (chartInstance) {
              chartInstance.destroy();
            }

            const ctx = chartCanvas.value.getContext('2d');
            chartInstance = new Chart(ctx, {
              type: 'line',
              data: {
                labels: labels,
                datasets: [{
                  label: `${selectedCrypto.value} Price`,
                  data: prices,
                  borderColor: 'rgb(75, 192, 192)',
                  tension: 0.1
                }]
              },
              options: {
                responsive: true,
                maintainAspectRatio: false,
              }
            });
          } catch (error) {
            console.error('Error fetching chart data:', error);
          }
        };

        onMounted(async () => {
          await initDatabase();
          await fetchTradeHistory();
          await fetchArbitrageOpportunities();
          await fetchChartData();
          setInterval(fetchArbitrageOpportunities, 60000);
          setInterval(fetchChartData, 60000);
        });

        watch(selectedCrypto, async () => {
          await fetchArbitrageOpportunities();
          await fetchChartData();
        });

        return {
          cryptos,
          selectedCrypto,
          tradeAmount,
          tradeHistory,
          arbitrageOpportunities,
          chartCanvas,
          executeTrade
        };
      }
    };
    </script>

    <style scoped>
    #app {
      font-family: sans-serif;
      padding: 20px;
    }

    .container {
      display: flex;
      flex-wrap: wrap;
      justify-content: space-around;
    }

    .trading-section,
    .arbitrage-section,
    .chart-section {
      width: 30%;
      padding: 15px;
      border: 1px solid #ddd;
      margin-bottom: 20px;
    }

    .trade-form {
      display: flex;
      flex-direction: column;
      margin-bottom: 10px;
    }

    .trade-form label {
      margin-bottom: 5px;
    }

    .trade-form input,
    .trade-form select {
      margin-bottom: 10px;
      padding: 8px;
    }

    .trade-history ul {
      list-style: none;
      padding: 0;
    }

    .trade-history li {
      margin-bottom: 5px;
    }

    .chart-section {
      height: 300px;
    }
    </style>
