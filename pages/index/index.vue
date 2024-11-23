<template>
    <view class="container">
        <view class="button-container">
            <button @click="startBeaconDiscovery">开始监听</button>
            <button @click="stopBeaconDiscovery">停止定位</button>
        </view>
      <!--  <view class="beacon-list">
            <text class="title">当前搜索到的 iBeacon 设备列表：</text>
            <view v-for="(beacon, index) in beacons" :key="index" class="beacon-item">
                <text>UUID: {{ beacon.uuid }} | RSSI: {{ beacon.rssi }}</text>
            </view>
        </view> -->
        <view class="position-result">
            <text class="title">定位坐标：</text>
            <text>{{ calculatedPosition }}</text>
        </view>
        <!-- 散点图 -->
		   <text class="title">定位地图：</text>
        <canvas canvas-id="scatterChart" id="scatterChart" class="charts" @touchend="tapScatter" />
		<text class="title">beacon rssi趋势：</text>
        <!-- 折线图 -->
        <canvas canvas-id="lineChart" id="lineChart" class="charts" @touchend="tapLine" />
    </view>
</template>

<script>
import * as math from 'mathjs';
import uCharts from '@qiun/ucharts';

var uChartsInstance = {};
export default {
    data() {
        return {
            beacons: [],
            calculatedPosition: "",
            beaconCoordinates: {
                "FDA50693-A4E2-4FB1-AFCF-C6EB07647824": { x: 0, y: 3.2 },
                "FDA50693-A4E2-4FB1-AFCF-C6EB07647821": { x: 0.51, y: 6.06 },
                "FDA50693-A4E2-4FB1-AFCF-C6EB07647823": { x: 3.27, y: 4.1 },
                "FDA50693-A4E2-4FB1-AFCF-C6EB07647822": { x: 2.65, y: 0 },
            },
            chartData: [], // 散点图数据
            rssiHistory: { // RSSI 历史数据
                "FDA50693-A4E2-4FB1-AFCF-C6EB07647824": [],
                "FDA50693-A4E2-4FB1-AFCF-C6EB07647821": [],
                "FDA50693-A4E2-4FB1-AFCF-C6EB07647823": [],
                "FDA50693-A4E2-4FB1-AFCF-C6EB07647822": [],
            },
            timeStamps: [], // 时间戳，用于折线图的 X 轴
            cWidth: 690,
            cHeight: 600,
        };
    },
    onReady() {
        this.cWidth = uni.upx2px(750);
        this.cHeight = uni.upx2px(500);
        this.initScatterChart();
        this.initLineChart();
    },
    methods: {
        startBeaconDiscovery() {
            uni.startBeaconDiscovery({
                uuids: Object.keys(this.beaconCoordinates),
                success: (res) => {
                    console.log("开始搜索成功", res);
                    this.listenToBeaconUpdates();
                },
                fail: (err) => {
                    console.error("开始搜索失败", err);
                },
            });
        },
        stopBeaconDiscovery() {
            uni.stopBeaconDiscovery({
                success: (res) => {
                    console.log("停止搜索成功", res);
                    this.chartData = [];
                    this.timeStamps = [];
                    for (let key in this.rssiHistory) {
                        this.rssiHistory[key] = [];
                    }
                    this.updateScatterChart();
                    this.updateLineChart();
                },
                fail: (err) => {
                    console.error("停止搜索失败", err);
                },
            });
        },
        listenToBeaconUpdates() {
            uni.onBeaconUpdate((res) => {
                console.log("设备更新", res);
                this.beacons = res.beacons;

                const timestamp = new Date().toLocaleTimeString([], {  second: '2-digit' });
                // 获取当前时分秒，格式 mm:ss
					
				
                this.timeStamps.push( this.timeStamps.length);

                // 更新 RSSI 历史数据
                res.beacons.forEach((beacon) => {
                    const uuid = beacon.uuid.toUpperCase();
                    if (this.rssiHistory[uuid]) {
                        this.rssiHistory[uuid].push(beacon.rssi);
                    }
                });

                // 限制历史数据长度，避免过长
                if (this.timeStamps.length > 10) {
                    this.timeStamps.shift();
                    for (let key in this.rssiHistory) {
                        if (this.rssiHistory[key].length > 10) {
                            this.rssiHistory[key].shift();
                        }
                    }
                }

                this.calculatePosition(res.beacons);
                this.updateLineChart();
            });
        },
        calculatePosition(beacons) {
            if (beacons.length >= 3) {
                try {
                    let coordinates = beacons.map((beacon) => {
                        let coord = this.beaconCoordinates[beacon.uuid.toUpperCase()];
                        return { x: coord.x, y: coord.y, rssi: beacon.rssi };
                    });

                    const pathLossExponent = 2; // 信号衰减因子
                    let distances = coordinates.map(beacon =>
                        Math.pow(10, (Math.abs(beacon.rssi) - 60) / (10 * pathLossExponent))
                    );

                    let A = [],
                        b = [];
                    for (let i = 1; i < beacons.length; i++) {
                        let { x, y } = coordinates[i];
                        A.push([2 * (x - coordinates[0].x), 2 * (y - coordinates[0].y)]);
                        b.push(
                            Math.pow(distances[0], 2) - Math.pow(distances[i], 2) -
                            Math.pow(coordinates[0].x, 2) + Math.pow(x, 2) -
                            Math.pow(coordinates[0].y, 2) + Math.pow(y, 2)
                        );
                    }

                    let At = math.transpose(A);
                    let AtA = math.multiply(At, A);
                    let Atb = math.multiply(At, b);
                    let position = math.lusolve(AtA, Atb);

                    this.calculatedPosition = `x: ${position[0][0].toFixed(2)}, y: ${position[1][0].toFixed(2)}`;
					
                    this.chartData.push([position[0][0], position[1][0]]);
                    this.updateScatterChart();
                } catch (e) {
                    console.error(e);
                }
            } else {
                this.calculatedPosition = "信号不足，无法计算定位结果";
            }
        },
        initScatterChart() {
            const ctx = uni.createCanvasContext("scatterChart", this);
            uChartsInstance["scatterChart"] = new uCharts({
                type: "scatter",
                context: ctx,
                width: this.cWidth,
                height: this.cHeight,
                series: [{ name: "定位结果", data: this.chartData }],
                animation: false,
                background: "#FFFFFF",
				dataLabel: false, // 关闭点上的文字展示
                color: ["#1890FF"],
                xAxis: { disableGrid: false, gridType: "dash" },
                yAxis: { disableGrid: false, gridType: "dash" },
            });
        },
        updateScatterChart() {
            if (uChartsInstance["scatterChart"]) {
                uChartsInstance["scatterChart"].updateData({
                    series: [{ name: "定位结果", data: this.chartData }],
                });
            }
        },
        initLineChart() {
            const ctx = uni.createCanvasContext("lineChart", this);
            uChartsInstance["lineChart"] = new uCharts({
                type: "line",
                context: ctx,
                width: this.cWidth,
                height: this.cHeight,
                categories: this.timeStamps,
                series: Object.keys(this.rssiHistory).map(uuid => ({
                    name: uuid,
                    data: this.rssiHistory[uuid],
                })),
                animation: false,
                background: "#FFFFFF",
                color: ["#1890FF", "#91CB74", "#FAC858", "#EE6666"],
                xAxis: { disableGrid: true },
                yAxis: { gridType: "dash" },
            });
        },
        updateLineChart() {
            if (uChartsInstance["lineChart"]) {
                uChartsInstance["lineChart"].updateData({
                    categories: this.timeStamps,
                    series: Object.keys(this.rssiHistory).map(uuid => ({
                        name: uuid,
                        data: this.rssiHistory[uuid],
                    })),
                });
            }
        },
        tapScatter(e) {
            uChartsInstance["scatterChart"].showToolTip(e);
        },
        tapLine(e) {
            uChartsInstance["lineChart"].showToolTip(e);
        },
    },
};
</script>

<style scoped>
.container {
    padding: 20rpx;
    background-color: #f5f5f5;
}

.button-container {
    display: flex;
    justify-content: space-between;
    margin-bottom: 20rpx;
}

button {
    flex: 1;
    margin: 0 10rpx;
    padding: 20rpx;
    background-color: #007aff;
    color: #fff;
    border: none;
    border-radius: 10rpx;
    font-size: 30rpx;
}

.beacon-list {
    margin-bottom: 20rpx;
}

.title {
    font-size: 32rpx;
    font-weight: bold;
    margin-bottom: 10rpx;
}

.beacon-item {
    background-color: #fff;
    padding: 20rpx;
    margin-bottom: 10rpx;
    border-radius: 10rpx;
    box-shadow: 0 2rpx 5rpx rgba(0, 0, 0, 0.1);
}

.position-result {
    margin-bottom: 20rpx;
}

.charts {
    width: 750rpx;
    height: 500rpx;
    background-color: #fff;
    border-radius: 10rpx;
    box-shadow: 0 2rpx 5rpx rgba(0, 0, 0, 0.1);
}
</style>
