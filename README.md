# beacon-indoor-position-app
# beacon-indoor-position-app

## 项目简介

`beacon-indoor-position-app` 是一个基于 iBeacon 技术的室内定位应用。该应用通过监听 iBeacon 设备的信号强度 (RSSI)，结合预先设定的坐标信息，计算出设备在室内的具体位置，并以图表的形式展示信号强度的变化趋势和定位结果。

## 功能特性

- **iBeacon 设备监听**：实时监听并获取周围 iBeacon 设备的信号强度。
- **室内定位计算**：基于多点信号强度计算设备的具体位置。
- **数据可视化**：使用折线图和散点图展示信号强度变化趋势和定位结果。
- **历史数据管理**：记录并管理信号强度的历史数据。

## 主要技术栈

- **前端框架**：Vue.js
- **数据处理**：mathjs
- **图表绘制**：uCharts
- **前端框架**：uniapp

## 项目结构

- `App.vue`：应用的主组件。
- `main.js`：应用的入口文件。
- `pages/index/index.vue`：主要的页面组件，包含定位和图表展示功能。
- `uni.promisify.adaptor.js`：Promise 适配器，用于处理异步操作。
- `uni.scss`：全局样式文件。
- `manifest.json`：应用的配置文件。
- `pages.json`：页面配置文件。

## 安装与运行

1. uniapp 需要安装Hbuilder x：


## 许可协议

该项目采用 MIT 许可协议，详情请参阅 [LICENSE](LICENSE) 文件。