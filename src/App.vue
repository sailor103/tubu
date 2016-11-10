<template lang="pug">
  .content-wrapper
    .start-wrapper
      button.start(@click="startLog",class="{{cssStart}}") {{{ logBtnText }}} 
    .view-wrapper
      button.view(@click="viewPos") 查看结果
    .setting
      button.set(@click="setParam") 应用设置
    .console {{consoleInfo}}
  Confirm(:show.sync="confirmObj.show",
    :title="confirmObj.title", 
    :ok-text="confirmObj.btnOk",
    :no-text="confirmObj.btnNo",
    :no-event="confirmObj.noEvt",
    :ok-event="confirmObj.okEvt") {{confirmObj.text}} 
  .mapbox(v-show="mapObj.show")
    #map-container
    button.mapclose(@click="closeMap") 关闭地图
  .set-wrapper(v-show="setObj.show")
    .set-panel
      button.delData(@click="clearData") {{setObj.text}} 
      p 采样间隔:
      input.interValue(placeholder="设置采样间隔", v-model="interVal")
      button.subSet(@click="subChange") 提交更改
</template>

<script>
import Confirm from './components/Confirm';
import BMap from 'BMap';
// import './js/convertor.js';

export default {
  data() {
    return {
      logBtnText: '开始记录',
      cssStart: 'green',
      /* 0.未开始 1.记录中 */
      status: 0,
      confirmObj: {
        show: false,
        title: '',
        text: '',
        btnOk: '确定',
        btnNo: '取消',
        okEvt() {},
        noEvt() {},
      },
      posObj: {
        watchid: 0,
        options: {
          enableHighAccuracy: true,
          timeout: 30000,
          maximumAge: 0,
        },
      },
      interVal: 10,
      consoleInfo: '',
      db: {},
      mapObj: {
        show: false,
        mapStore: {},
      },
      setObj: {
        show: false,
        text: '清空数据',
      },
    };
  },
  ready() {
    this.mapObj.mapStore = new BMap.Map('map-container');
  },
  methods: {
    /**
     * 开始记录按钮响应
    */
    startLog() {
      if (this.status === 0) {
        this.logBtnText = '记录中..<br><span class="subtitle">(点击停止)</span>';
        this.cssStart = 'red';
        this.status = 1;
        this.onStartLoc();
      } else {
        this.confirmObj.text = '是不是误触了？真的要停止记录？';
        this.confirmObj.btnOk = '停止记录';
        this.confirmObj.btnNo = '点错了';
        this.confirmObj.title = '温馨提示';
        this.confirmObj.okEvt = () => {
          this.logBtnText = '开始记录';
          this.cssStart = 'green';
          this.status = 0;
          this.confirmObj.show = false;
          this.onEndLoc();
        };
        this.confirmObj.show = true;
      }
    },
    /**
     * 封装定位函数
    */
    onGeoNow() {
      navigator.geolocation.getCurrentPosition(
        this.onLocSuc, this.onLocErr, this.posObj.options
      );
    },
    /**
     * 定位成功回调
    */
    onLocSuc(position) {
      const coord = {
        time: position.timestamp,
        latitude: position.coords.latitude,
        longitude: position.coords.longitude,
        accuracy: position.coords.accuracy,
        heading: position.coords.heading,
        speed: position.coords.speed,
      };
      const store = this.db.transaction(['locations'], 'readwrite').objectStore('locations');
      store.add(coord);
      this.consoleInfo += ` ${coord.time}`;
    },
    /**
     * 定位失败回调
    */
    onLocErr(err) {
      console.log(err);
    },
    /**
     * 打开数据库同时开始定位
    */
    onStartLoc() {
      const request = window.indexedDB.open('myLocation', '1');
      request.onsuccess = (openData) => {
        this.db = openData.target.result;
        this.onGeoNow();
        this.posObj.watchid = setInterval(this.onGeoNow, this.interVal * 1000);
      };
      request.onupgradeneeded = (upData) => {
        this.db = upData.target.result;
        if (!this.db.objectStoreNames.contains('locations')) {
          this.db.createObjectStore('locations', {
            autoIncrement: true,
          });
        }
      };
    },
    /**
     * 结束定位函数
    */
    onEndLoc() {
      clearInterval(this.posObj.watchid);
    },
    /**
     * 转换坐标并显示
     */
    transDis(data) {
      if (data.status === 0) {
        const newPoints = [];
        for (let i = 0; i < data.points.length; i++) {
          newPoints.push(data.points[i]);
        }
        const polyline = new BMap.Polyline(newPoints, {
          strokeColor: 'blue',
          strokeWeight: 6,
          strokeOpacity: 0.5,
        });
        this.mapObj.mapStore.addOverlay(polyline);
        this.mapObj.mapStore.centerAndZoom(newPoints[0], 15);
      }
    },
    /**
     * 查看地图
    */
    viewPos() {
      this.mapObj.show = true;
      // this.mapObj.mapStore.clearOverlays();
      const request = window.indexedDB.open('myLocation', '1');
      request.onsuccess = (openData) => {
        const pointArr = [];
        this.db = openData.target.result;
        const objectStore = this.db.transaction('locations').objectStore('locations');
        objectStore.openCursor().onsuccess = (event) => {
          const cursor = event.target.result;
          if (cursor) {
            const tmpPoint = new BMap.Point(cursor.value.longitude, cursor.value.latitude);
            pointArr.push(tmpPoint);
            cursor.continue();
          } else {
            // 重采样
            const minPoints = [];
            for (let i = 0; i < pointArr.length; i++) {
              if (i % 9 === 0) {
                minPoints.push(pointArr[i]);
              }
            }
            // 添加地图工具条
            const tln = new BMap.NavigationControl({
              anchor: window.BMAP_ANCHOR_TOP_LEFT,
            });
            this.mapObj.mapStore.clearOverlays();
            this.mapObj.mapStore.addControl(tln);
            this.mapObj.mapStore.centerAndZoom(minPoints[0], 15);
            // 批量转换坐标
            const convertor = new BMap.Convertor();
            let tParr = [];
            for (let i = 0; i < minPoints.length; i++) {
              tParr.push(minPoints[i]);
              if (tParr.length % 9 === 0) {
                convertor.translate(tParr, 1, 5, this.transDis);
                tParr = [];
                tParr.push(minPoints[i]);
              }
            }
            convertor.translate(tParr, 1, 5, this.transDis);
          }
        };
      };
    },
    /**
     * 关闭地图
    */
    closeMap() {
      this.mapObj.show = false;
    },
    /**
     * 应用设置按钮响应
     */
    setParam() {
      this.setObj.show = true;
    },
    /**
     * 提交更改响应
     */
    subChange() {
      this.setObj.show = false;
    },
    /**
     * 清空数据库响应
     */
    clearData() {
      this.confirmObj.text = '数据清了就什么都没了！！！！';
      this.confirmObj.btnOk = '果断删除';
      this.confirmObj.btnNo = '我再想想';
      this.confirmObj.title = '温馨提示';
      this.confirmObj.okEvt = () => {
        const del = window.indexedDB.deleteDatabase('myLocation');
        del.onsuccess = () => {
          this.setObj.text = '已经清空啦';
        };
      };
      this.confirmObj.show = true;
    },
  },
  components: {
    Confirm,
  },
};
</script>

<style lang="stylus">
*
  margin    0
  padding   0 

html,body 
  -webkit-user-select    none 
  font-family            'PingFang SC', 'Comic Sans MS', "幼圆", "黑体", sans-serif 
  background             #F3F3F3 
  padding-bottom         50px  
 
.left 
  float left 
 
.right 
  float right 
 
.center 
  text-align  center 
 
.clear 
  clear both 
 
.green
  background-color #04be02

.red
  background-color #ef4f4f

.subtitle
  font-size 18px

.alertBoxBtn button
  color #04be02

.content-wrapper
  max-width 750px
  margin 0 auto

  .start-wrapper
    width 100%
    height 320px
    position relative

    .start
      width 180px
      height 180px
      border-radius 50%
      color #ffffff
      position absolute
      top 0
      bottom 0
      left 0
      right 0
      margin auto
      border 0
      outline 0
      font-size 30px

  .view-wrapper
    width 100%
    text-align center
    .view
      width 180px 
      border 0
      outline 0
      height 45px
      border-radius 5px
      font-size 15px
      color #ffffff
      background-color #1677D2

  .setting
    width 100%
    text-align center
    margin-top 30px
    .set
      width 180px 
      border 0
      outline 0
      height 45px
      border-radius 5px
      font-size 15px
      color #ffffff
      background-color #38ae67
.mapbox
  position fixed
  top 0
  left 0
  width 100%
  height 100%
  background #ffffff
  #map-container
    width 100%
    height 100%
  .mapclose
    width 200px 
    height 40px
    outline none
    border none
    border-radius 5px
    position absolute
    bottom 10px 
    left 0
    right 0
    margin auto
    background #1677D2
    color #ffffff

.set-wrapper
  background-color rgba(0,0,0,0.7)
  position fixed
  width 100%
  height 100%
  top 0
  left 0
  z-index 2
  .set-panel
    width 240px
    height 200px
    background #FFFFFF
    position absolute
    top 0
    left 0
    bottom 0
    right 0
    margin auto
    border-radius 5px
    padding 15px
    box-sizing border-box
    -webkit-box-sizing border-box
    -moz-box-sizing border-box
    text-align center
    p
      font-size 14px
      width 90%
      display block
      box-sizing border-box
      -webkit-box-sizing border-box
      -moz-box-sizing border-box
      text-align left
      margin 0 auto
      margin-top 15px
    button
    input
      width 90%
      height 36px
      border none
      outline none
      border-radius 5px
      margin-top 15px
      box-sizing border-box
      -webkit-box-sizing border-box
      -moz-box-sizing border-box
    .delData
      background red
      color #fff
      margin-top 5px
    .interValue
      border 1px solid #1677D2
      padding-left 5px
      margin-top 5px
    .subSet
      background #38ae67
      color #FFF 

</style>
