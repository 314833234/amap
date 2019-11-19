<template>
  <div>
    <div id="amap-container"></div>
    <div class="amap-info-window" :id="amapInfoWindowId">
      <div class="amap-info-window-warp">
        <h3>{{infoWindowData.storeName || ''}}</h3>
        <p>{{infoWindowData.address}}</p>
      </div>
    </div>
  </div>
</template>

<script>
import mapIcon from '@/assets/images/icon.svg'

export default {
  data() {
    return {
      amapKey: '5ce7f61e3c3508f8978ed45a99116613',  // amap key
      amapObj: null,	// amap对象
      amapId: 'amap-container',	// 地图容器
      amapIcon: null,
      amapInfoWindowId: '', // 窗体对象无素id
      amapInfoWindow: null, // 窗体信息
      amapMarkers: [],
      infoWindowData: {}, // 窗体信息数组
    }
  },
  created () {
    let randomNum = (+new Date()) + '_' + parseInt(Math.random() * 10000);
    // 调用组件没有传入id，组件帮自动生成一个id
    this.amapInfoWindowId = 'amap-infowindow-' + randomNum;
  },
  mounted() {
    // 引入amap依赖
    this.insertMap(`//webapi.amap.com/maps?v=1.4.15&key=${this.amapKey}&callback=init`);
    
    // 初始化amap，当 amap 依赖加载完成后会调用 init 方法
    window.init = this.initAmap;
  },
  methods: {
    /**
     * 插入高德地图JS文件
     * @param {String} url	高德地图依赖 Javascript 文件
     */
    insertMap (url) {
      // #amap-script 是高德地图script标签上的ID，遍免多次创建引入
      let myScript, oAmap = document.querySelector('#amap-script');
      if(!oAmap){
        myScript = document.createElement('script');
        myScript.type = 'text/javascript';
        myScript.src = url;
        myScript.defer = true;
        myScript.async = true;
        myScript.id = 'amap-script';
        document.body.appendChild(myScript);
      }
    },
    /**
     * 初始化高德地图
     */
    async initAmap () {
      this.amapObj = await new AMap.Map(this.amapId);

      await this.initMapIcon()

      await this.initInfoWindow()

      await this.autoPostion()

      await this.initMarkers([
        {
          address: "文心五路海岸城购物中心负一层042铺",
          city: "深圳市",
          country: "China",
          county: "南山区",
          hoursTime: "10:00-21:00",
          lat: "22.516761",
          lng: "113.93655",
          phone: "0755-26927034",
          province: "广东省",
          storeName: "顺电深圳海岸城店"
        }
      ])
    },
    /**
     * 初始化地图自定义icon
     */
    initMapIcon () {
      this.amapIcon = new AMap.Icon({
        size: new AMap.Size(28, 36),    // 图标尺寸
        image: mapIcon,  // Icon的图像
        imageOffset: new AMap.Pixel(0, 0),  // 图像相对展示区域的偏移量，适于雪碧图等
        imageSize: new AMap.Size(28, 36)   // 根据所设置的大小拉伸或压缩图片
      })
    },
    /**
     * 窗体信息
     */
    initInfoWindow() {
      this.amapInfoWindow = new AMap.InfoWindow({
        isCustom: true,  //使用自定义窗体
        closeWhenClickMap: true,  // 控制是否在鼠标点击地图后关闭信息窗体，默认false，鼠标点击地图后不关闭信息窗体
        offset: new AMap.Pixel(0, -30)
      });
    },
    /**
     * 实例化marker
     * @param {Array} markers 坐标点数组
     */
    initMarkers (markers) {
      const _this = this

      markers.forEach(marker => {
        let markerObj = new AMap.Marker({
          position: new AMap.LngLat(marker.lng, marker.lat),
          offset: new AMap.Pixel(-10, -10),
          icon: this.amapIcon, // 添加 Icon 实例
          //title: marker.title,
          //zoom: 13
        })

        //鼠标点击marker弹出自定义的信息窗体
        AMap.event.addListener(markerObj, 'click', function (data) {
          _this.infoWindowData = marker

          _this.$nextTick(function() {
            // 窗体内容
            let infoWindowContent = document.querySelector('#' + _this.amapInfoWindowId).innerHTML;

            _this.amapInfoWindow.setContent(infoWindowContent)
            _this.amapInfoWindow.open(_this.amapObj, markerObj.getPosition())
          })
        })

        this.amapMarkers.push(markerObj)
      })

      this.amapObj.add(this.amapMarkers)
    },
    /**
     * 自动定位
     */
    autoPostion () {
      const _this = this;

      AMap.plugin('AMap.Geolocation', function() {
        var geolocation = new AMap.Geolocation({
          enableHighAccuracy: true,//是否使用高精度定位，默认:true
          timeout: 10000,          //超过10秒后停止定位，默认：5s
          buttonPosition:'RB',    //定位按钮的停靠位置
          buttonOffset: new AMap.Pixel(14, 130),//定位按钮与设置的停靠位置的偏移量，默认：Pixel(10, 20)
          zoomToAccuracy: true,   //定位成功后是否自动调整地图视野到定位点
          showMarker: false
        });
        _this.amapObj.addControl(geolocation);
        geolocation.getCurrentPosition(function(status, result){
          // 服务异常
          if(_this.amapServerError(result.info)) {
            _this.$emit('server-error', result.message);
            return false;
          }

          if(status=='complete'){
            _this.$emit('auto-position-success', result);
          }
          else{
            _this.$emit('auto-position-error', result);
          }
        });
      });
    },
    /**
     * 高德地图服务异常
     * 10001  INVALID_USER_KEY            key不正确或过期
     * 10003  DAILY_QUERY_OVER_LIMIT      访问已超出日访问量
     * 10006  INVALID_USER_DOMAIN         绑定的域名无效
     * 10009  USERKEY_PLAT_NOMATCH        请求key与绑定平台不符
     * 10012  INSUFFICIENT_PRIVILEGES     权限不足，请求服务被拒绝
     * 300**  ENGINE_RESPONSE_DATA_ERROR  大于30000时，服务响应失败
     */
    amapServerError(errCode) {
      let flag = false // false: 服务正常    true: 服务异常
      let serverErrorCode = [10001, 10003, 10006, 10009, 10012]
      let reqErrCode = serverErrorCode.some(item => item == errCode); // 服务异常

      if(!errCode) return flag;
      if(reqErrCode) flag = true;

      return flag;
    }
  }
}
</script>

<style lang="scss">
  #amap-container {
    width: 100vw;
    height: 100vh;
  }

  .amap-info-window-warp {
    position: relative;
    padding: 20px;
    width: 280px;
    border-radius: 2px;
    box-shadow: 0 4px 8px 0 rgba(0, 0, 0, 0.2);
    background-color: #ffffff;

    &:before {
      content: '';
      position: absolute;
      bottom: -10px;
      left: 50%;
      transform: translateX(-50%);
      width: 0;
      height: 0;
      display: block;
      border-left: 10px solid transparent;
      border-right: 10px solid transparent;
      border-top: 10px solid #fff;
    }
  }
</style>