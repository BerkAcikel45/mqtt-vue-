<template>
  <div class="main">
     <div v-show="loader" class="loader">Loading...</div>   
      <div class="container" v-show="!loader">
          <div class="row">
              <div class="col-md-6">
                <div class="card" style="width: 100%;">
                    <div class="card-body" style="height: 500px">
                        <h5 class="card-title">Client Status</h5>
                        <div id="piechart" style="height: 100%"></div>
                    </div>
                </div>
              </div>
              <div class="col-md-6">
                <div class="card" style="width: 100%;">
                    <div class="card-body" style="height: 500px">
                        <h5 class="card-title">Messages</h5>
                        <div id="messages" style="height: 100%"></div>
                    </div>
                </div>
              </div>
                <div class="col-md-12"  style="margin-top:50px">
                    <div class="card" style="width: 100%;">
                        <div class="card-body" style="height: 500px">
                            <h5 class="card-title">Bytes</h5>
                            <div id="bytes" style="height: 100%"></div>
                        </div>
                    </div>
              </div>
                <div class="col-md-12"  style="margin-top:50px">
                    <div class="card" style="width: 100%;">
                        <div class="card-body" style="height: 500px">
                            <h5 class="card-title">Packets</h5>
                            <div id="packets" style="height: 100%"></div>
                        </div>
                </div>
                <div class="col-md-12" style="margin-top:50px">
                    <div class="card" style="width: 100%">
                        <div class="card-body" style="height: 500px">
                            <h5 class="card-title">Subscriptions</h5>
                            <div id="subs" style="height:100%"></div>
                        </div>
                    </div>
                </div>
              </div>
          </div>
      </div>

  </div>
</template>

<script>
import mqtt from 'mqtt'
import * as am4core from "@amcharts/amcharts4/core";
import * as am4charts from "@amcharts/amcharts4/charts";

export default {
  name: 'main',
  data() {
    return {
      connection: {
        host: 'mqtttest.connio.cloud',
        port: 8083,
        endpoint: '/mqtt',
        protocol: 'mqtts', 
        protocolId: 'MQIsdp',
        clean: true, 
        connectTimeout: 4000, 
        reconnectPeriod: 4000,
        clientId: 'digiterra-coding-task-1',
        username: '',
        password: '',
      },
      subscription: {
        topic: '$SYS',
        qos: 0,
      },
      receiveNews: '',
      qosList: [
        { label: 0, value: 0 },
        { label: 1, value: 1 },
        { label: 2, value: 2 },
      ],
      client: {
        connected: false,
      },
      subscribeSuccess: false,
      formatted_data: {},
      loader: true,

    }
  },

    methods: {

    },

    created() {
      const { host, port, endpoint, ...options } = this.connection
      const connectUrl = `ws://${host}:${port}${endpoint}`
      const { topic, qos } = this.subscription
      try {
        this.client = mqtt.connect(connectUrl, options)
      } catch (error) {
        console.log('mqtt.connect error', error)
      }
      this.client.on('connect', () => {
        console.log('Connection succeeded!')
      })
      this.client.on('error', error => {
        console.log('Connection failed', error)
      })
      this.client.on('message', (topic, message) => {
        this.receiveNews = this.receiveNews.concat(message)
        //console.log(`Received message ${message} from topic ${topic}`)
      })

        this.client.subscribe(topic, { qos }, (error, res) => {
        if (error) {
            console.log('Subscribe to topics error', error)
            return
        }
            this.subscribeSuccess = true
            console.log('Subscribe to topics res', res)
        })
    },

    watch:{
        receiveNews:function(){
            this.loader = false;

            let received = this.receiveNews;
            //delete {, }, " chars
            received = received.replace('}', '');
            received = received.replace('{', '');
            received = received.replace(/"/g, '');

            let formatted = received.split(",");
            // make objects 
            for(let i = 1; i < formatted.length; i++ ){
                let split = formatted[i].split(":");
                
                this.formatted_data[split[0]] = split[1]
                
            }   
            console.log(this.formatted_data);

            // create piechart
            let chart = am4core.create("piechart", am4charts.PieChart);

            // Add data
            chart.data = [
            {
                "status": 'maxConnected',
                "data": this.formatted_data['maxConnected']
            },
            {
                "status": 'disconnected',
                "data": this.formatted_data['disconnected']

            }
            ];

            // Add and configure Series
            let pieSeries = chart.series.push(new am4charts.PieSeries());
            pieSeries.dataFields.value = "data";
            pieSeries.dataFields.category = "status";
            pieSeries.ticks.template.disabled = true;
            pieSeries.alignLabels = false;
            pieSeries.labels.template.text = "";


            // create message chart
            let messagesChart = am4core.create("messages", am4charts.XYChart);
            messagesChart.data = [
            {
                "status": 'Sent',
                "data": this.formatted_data['messageSent']
            },
            {
                "status": 'Received',
                "data": this.formatted_data['messageReceived']
            },
            {
                "status": 'Dropped',
                "data": this.formatted_data['messageDropped']

            }
            ];

            let categoryAxis = messagesChart.xAxes.push(new am4charts.CategoryAxis());
                    categoryAxis.renderer.grid.template.location = 0;
                    categoryAxis.dataFields.category = "status";
                    categoryAxis.renderer.minGridDistance = 60;
                    categoryAxis.renderer.inversed = true;
                    categoryAxis.renderer.grid.template.disabled = true;
                    categoryAxis.renderer.hideOversized  = true;
                    categoryAxis.renderer.labels.template.wrap = true;
                    categoryAxis.renderer.labels.template.maxWidth = 300;

            let valueAxis = messagesChart.yAxes.push(new am4charts.ValueAxis());
                    valueAxis.min = 0;
                    valueAxis.extraMax = 0.1;

            let series = messagesChart.series.push(new am4charts.ColumnSeries());


            series.dataFields.categoryX = "status";
            series.dataFields.valueY = "data";


            // create message chart
            let bytesChart = am4core.create("bytes", am4charts.XYChart);
            bytesChart.data = [
            {
                "status": 'Sent',
                "data": this.formatted_data['messageBytesSent']
            },
            {
                "status": 'Received',
                "data": this.formatted_data['messageBytesReceived']
            },
            ];

            let categoryAxis_bytesChart = bytesChart.xAxes.push(new am4charts.CategoryAxis());
                    categoryAxis_bytesChart.renderer.grid.template.location = 0;
                    categoryAxis_bytesChart.dataFields.category = "status";
                    categoryAxis_bytesChart.renderer.minGridDistance = 60;
                    categoryAxis_bytesChart.renderer.inversed = true;
                    categoryAxis_bytesChart.renderer.grid.template.disabled = true;
                    categoryAxis_bytesChart.renderer.hideOversized  = true;
                    categoryAxis_bytesChart.renderer.labels.template.wrap = true;
                    categoryAxis_bytesChart.renderer.labels.template.maxWidth = 300;

            let valueAxis_bytesChart= bytesChart.yAxes.push(new am4charts.ValueAxis());
                    valueAxis_bytesChart.min = 0;
                    valueAxis_bytesChart.extraMax = 0.1;

            let series_bytesChart = bytesChart.series.push(new am4charts.ColumnSeries());
            series_bytesChart.dataFields.categoryX = "status";
            series_bytesChart.dataFields.valueY = "data";
            series_bytesChart.tooltipText = "{valueY.data}"
            series_bytesChart.columns.template.strokeOpacity = 0;
            series_bytesChart.columns.template.column.cornerRadiusTopRight = 10;
            series_bytesChart.columns.template.column.cornerRadiusTopLeft = 10;

            // create message packets
            let packetsChart = am4core.create("packets", am4charts.XYChart);
            packetsChart.data = [
            {
                "status": 'Sent',
                "data": this.formatted_data['messageBytesSent']
            },
            {
                "status": 'Received',
                "data": this.formatted_data['messageBytesReceived']
            },
            ];

            let categoryAxis_packetsChart = packetsChart.xAxes.push(new am4charts.CategoryAxis());
                    categoryAxis_packetsChart.renderer.grid.template.location = 0;
                    categoryAxis_packetsChart.dataFields.category = "status";
                    categoryAxis_packetsChart.renderer.minGridDistance = 60;
                    categoryAxis_packetsChart.renderer.inversed = true;
                    categoryAxis_packetsChart.renderer.grid.template.disabled = true;
                    categoryAxis_packetsChart.renderer.grid.template.location = 0;
                    categoryAxis_packetsChart.renderer.hideOversized  = true;
                    categoryAxis_packetsChart.renderer.labels.template.wrap = true;
                    categoryAxis_packetsChart.renderer.labels.template.maxWidth = 300;

            let valueAxis_packetsChart= packetsChart.yAxes.push(new am4charts.ValueAxis());
                    valueAxis_packetsChart.min = 0;
                    valueAxis_packetsChart.extraMax = 0.1;

            let series_packetsChart = packetsChart.series.push(new am4charts.ColumnSeries());

            series_packetsChart.dataFields.categoryX = "status";
            series_packetsChart.dataFields.valueY = "data";
            series_packetsChart.columns.template.column.cornerRadiusTopRight = 10;
            series_packetsChart.columns.template.column.cornerRadiusTopLeft = 10;






            // Create chart instance
            let subsChart = am4core.create("subs", am4charts.XYChart);
            subsChart.data = [
            {
                "status": 'Subscribed',
                "data": this.formatted_data['subscribed']
            },
            {
                "status": 'Unsubscribed',
                "data": this.formatted_data['unsubscribed']
            },
            ];

           let categoryAxis_subsChart = subsChart.xAxes.push(new am4charts.CategoryAxis());
                    categoryAxis_subsChart.renderer.grid.template.location = 0;
                    categoryAxis_subsChart.dataFields.category = "status";
                    categoryAxis_subsChart.renderer.minGridDistance = 60;
                    categoryAxis_subsChart.renderer.inversed = true;
                    categoryAxis_subsChart.renderer.grid.template.disabled = true;
                    categoryAxis_subsChart.renderer.grid.template.location = 0;
                    categoryAxis_subsChart.renderer.hideOversized  = true;
                    categoryAxis_subsChart.renderer.labels.template.wrap = true;
                    categoryAxis_subsChart.renderer.labels.template.maxWidth = 300;

            let valueAxis_subsChart= subsChart.yAxes.push(new am4charts.ValueAxis());
                    valueAxis_subsChart.min = 0;
                    valueAxis_subsChart.extraMax = 0.1;

            let series_subsChart = subsChart.series.push(new am4charts.ColumnSeries());

            series_subsChart.dataFields.categoryX = "status";
            series_subsChart.dataFields.valueY = "data";
            series_subsChart.columns.template.column.cornerRadiusTopRight = 10;
            series_subsChart.columns.template.column.cornerRadiusTopLeft = 10;

        }
    }

  }


</script>


