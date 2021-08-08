<template>
  <!-- <img alt="Vue logo" src="./assets/logo.png" /> -->
  <div class="hero is-light">
    <div class="hero-body">
      <div class="container">
        <h1 class="title">ArConnect Statistics</h1>
      </div>
    </div>
  </div>
  <div class="hero is-medium">
    <div class="hero-body">
      <div class="container">
        <div class="columns">
          <div class="column">
            <div class="box">
              <h2 class="subtitle">TOTAL AR</h2>
              <h1 class="title">{{ totalARDist }}</h1>
            </div>
          </div>
          <div class="column">
            <div class="box">
              <h2 class="subtitle">TOTAL TIPS</h2>
              <h1 class="title">{{ totalTips }}</h1>
            </div>
          </div>
          <div class="column">
            <div class="box">
              <h2 class="subtitle">TOTAL RECIPIENTS</h2>
              <h1 class="title">{{ totalARRecipients.length }}</h1>
            </div>
          </div>
        </div>
        <div class="box">
          <vue3-chart-js ref="chartRef" v-bind="{ ...barChart }" />
        </div>
      </div>
    </div>
  </div>
</template>

<script>

import Vue3ChartJs from '@j-t-mcc/vue3-chartjs';
import { all } from "ar-gql";
import * as dayjs from "dayjs";
import { ref, watch } from "vue";

export default {
  name: "App",
  components: {
    Vue3ChartJs
  },
  data() {
    return {
      totalTips: 0,
      totalARDist: 0,
      totalARRecipients: [],
      errorString: ""
    }
  },
  props: ["chartData"],
  setup(props) {
    const chartRef = ref(null);

    const barChart = {
      type: "bar",
      options: {
        min: 0,
        max: 1,
        responsive: true,
        plugins: {
          legend: {
            position: "top"
          }
        },
        scales: {
          y: {
            min: 0,
            max: 1,
            ticks: {
              callback: function(value) {
                return `${value}hi`;
              }
            }
          }
        }
      },
      data: props.chartData
    };

    watch(() => props.chartData, () => {
      console.log("Chart data has changed bruh");
      chartRef.value.update(250);
    });

    return {
      barChart,
      chartRef
    }
  },
  async mounted() {
    await this.fetchTipData();
  },
  methods: {
    async fetchTipData() {
      const queryString = `query($cursor: String) { transactions( tags: [ { name: "App-Name", values: "ArConnect" } { name: "Type", values: "Fee-Transaction" } ] after: $cursor ) { pageInfo { hasNextPage } edges { cursor node { recipient quantity { ar } block { timestamp } } } } }`;
      let queryResult;
      try {
        queryResult = await all(queryString);
        console.log("Done querying");
      } catch (err) {
        this.errorString = err.toString();
      }
      
      this.totalTips = queryResult.length;
      this.calcTotalARDistributed(queryResult);
      this.calcTotalARRecipients(queryResult);
      this.prepareTipChartData(queryResult);
    },

    calcTotalARDistributed(response) {
      let total = 0;
      for (let i = 0; i < response.length; i++) {
        total += parseFloat(response[i].node.quantity.ar);
      }
      this.totalARDist = total;
    },

    calcTotalARRecipients(response) {
      let recipientsArray = [];
      for (let i = 0; i < response.length; i++) {
        const currentRecipient = response[i].node.recipient;
        if (!recipientsArray.includes(currentRecipient)) {
          recipientsArray.push(currentRecipient);
        }
      }
      this.totalARRecipients = recipientsArray;
    },

    prepareTipChartData(response) {
      const timeMap = [];
      const amountMap = [];
      for (let i = 0; i < response.length; i++) {
        const currentTime = dayjs.unix(response[i].node.block.timestamp).format("MM/DD/YYYY");
        let foundTime = false;
        for (let x = 0; x < timeMap.length; x++) {
          if (timeMap.length && !foundTime) {
            if (timeMap[x] === currentTime && !foundTime) {
              amountMap[x] += parseFloat(response[i].node.quantity.ar);
              foundTime = true;
            }
          }
        }
        if (!foundTime) {
          timeMap.push(currentTime);
          amountMap.push(parseFloat(response[i].node.quantity.ar));
        }
      }
      this.props.chartData.labels = timeMap;
      this.props.chartData.datasets.data = amountMap;
      console.log(this.barChart.data.labels);
    }
  }
}

</script>

<style lang="scss">
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
</style>
