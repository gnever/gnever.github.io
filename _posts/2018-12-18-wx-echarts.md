---
layout: post
title: 在微信小程序中使用 ECharts
categories: 微信小程序 Echarts
description: 在微信小程序中使用 ECharts
keywords:  小程序 ECharts
---

## 官方文档
官方已经提供了小程序版的 ECharts  [echarts-for-weixin](https://github.com/ecomfe/echarts-for-weixin) 并提供的了使用示例

使用方式还是熟悉的 ECharts 的配置方案，只要自己 option 或者完整的将浏览器端的 option 迁移到小程序端即可

## 使用方法

### 引入组件

`ec-canvas` 是 ECharts 提供的微信小程序端的组件，在项目中引入即可
```js
import * as echarts from '../../ec-canvas/echarts';
```

>源文件中 `echarts.js` 占用空间较大（2.75MB），建议[自定义构建](http://echarts.baidu.com/builder.html)以减小体积

### 创建图表

- 在项目 ```*.json``` 文件中开启 ```ec-canvas``` 的使用

```js
{
  "usingComponents": {
    "ec-canvas": "../../ec-canvas/ec-canvas"
  }
}
```

- 在 ```.wxml``` 文件中创建 ```ec-canvas``` 组件

```
<view class="container">
  <ec-canvas id="mychart-dom-bar" canvas-id="mychart-bar" ec="{{ ec }}"></ec-canvas>
</view>
````

其中 ```ec``` 是在 ```*.js```中定义的对象，接下来的代码会看到
- 完整的```*.js```文件

```js
import * as echarts from '../../ec-canvas/echarts';

const app = getApp();
const getChartDataUrl = require('../../config').getChartDataUrl;

function setOption(chart, chartData) {
    const option = {
      ......//这里替换成自己的option代码
    };
  chart.setOption(option);
}

Page({
  onShareAppMessage: res => {
    return {
    }
  },

  onLoad: function (e) {
    // 获取组件
    this.ecComponent = this.selectComponent('#mychart-dom-bar');
    this.init(e.scode)//这里是获取url中get方式传递的参数 比如 http://xx.com?scode=11
  },

  data: {
    ec: {  //这里就是上面提到的 ec 对象
      // 将 lazyLoad 设为 true 后，需要手动初始化图表
      lazyLoad: true
    },
    isLoaded: false
  },

  // 点击按钮后初始化图表
  init: function (scode) {
      wx.showLoading({
          title: '加载中',
      })
      const self = this

    this.ecComponent.init((canvas, width, height) => {
      // 获取组件的 canvas、width、height 后的回调函数
      // 在这里初始化图表
      const chart = echarts.init(canvas, null, {
        width: width,
        height: height
      });

        //这个逻辑是异步获取数据然后组装 option 再渲染。若没有需求可以去掉
        wx.request({
            header:{gid:wx.getStorageSync('gid')},
            url: getChartDataUrl,
            data: {scode:scode},
            success(result) {

                if(result.data.code) {
                    wx.hideLoading()
                    wx.showToast({
                        title: result.data.message,
                        icon: 'none',
                        mask: true
                    })
                    return true;
                }

                setOption(chart, result.data.data);

                wx.hideLoading()
                self.setData({
                    list: result.data.data.list,
                    name: result.data.data.name,
                    stock_code: result.data.data.scode
                })
            },

            fail({errMsg}) {
                wx.hideLoading()
                wx.showToast({
                    title: '调用失败',
                    icon: 'none',
                    mask: true
                })
            }
        });



      // 将图表实例绑定到 this 上，可以在其他成员函数（如 dispose）中访问
      this.chart = chart;

      this.setData({
        isLoaded: true
      });

      // 注意这里一定要返回 chart 实例，否则会影响事件处理等
      return chart;
    });
  }

});
```

