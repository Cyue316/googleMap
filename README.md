# googleMap
googleMap
<template>
  <div style="margin-top:10px;">
    <div ref="map" style="box-sizing:border-box; width:100%; height:500px; border-radius:6px; border: 1px solid black; margin-top:10px;">
    </div>
  </div>
</template>

<script>

  import util from 'assets/js/util.js'
  import axios from 'assets/js/axios'
  import qs from 'qs'

  export default {

    props: {
      source:{default:"10001"},
      customerId: {default: 0},
    },

    data() {
      return {
        map : null,
        infoWindow : null,
        geoCoder : null,

        gpsList : [],
        totalGpsCount : 0,
        positionList : [],
        totalPositionCount : 0,
      }
    },

    mounted() {
      this.loadGpsInfo();
    },

    methods: {

      loadGpsInfo(){
        axios({
          method: 'get',
          headers: {'Accept-Language': localStorage.getItem('lang')},
          url: '/examine-platform/customerInfo/queryGPS',
          params: {
            customerId: this.customerId,
            currentPage: 1,
            pageSize: 10000,
          }
        }).then(res => {
          if (res.data.errcode) {
            this.$alert(res.data.msg, {
              type: 'error'
            });
            return;}
          this.convertGpsInfoToList(res.data.data)
          this.caculatePositionList();
          this.initGoogleMapApi();
        });
      },

      convertGpsInfoToList(data){
        this.totalGpsCount = data.totalCount;
        let customerGPSList = data.customerGPSList;
        let gpsList = [];
        for(let i in customerGPSList){
          let customerGPS = customerGPSList[i];
          gpsList.push({
            "lat": customerGPS["lat"],
            "lng": customerGPS["lon"],
            "time":customerGPS["time"],
          });
        }
        this.gpsList = gpsList;
      },

      caculatePositionList(){
        let gpsList = this.gpsList;
        let positionList = [];
        out: for(let i in gpsList){
          let rawGps = gpsList[i];
          for(let j in positionList){
            let gps = positionList[j];
            if(rawGps.lat == gps.lat && rawGps.lng == gps.lng){
              gps.timeList.push(rawGps.time);
              continue out;
            }
          }
          positionList.push({"lat":rawGps.lat, "lng":rawGps.lng, "timeList":[rawGps.time]});
        }
        this.positionList =  positionList;
        this.totalPositionCount = positionList.length;
      },

      initGoogleMapApi(){
        let language = localStorage.getItem('lang');
//        let key = "AIzaSyCBj-oybLhR7XsaetOESDM3HQf9vM6tLhk"; //original
        let key = "AIzaSyA81nVdAlkgHAGF0xTzasQlXb0lMckTVoY"; //from YuanJun which has payed
        let scriptSrc = "https://maps.googleapis.com/maps/api/js?key="+key+"&language="+language;

        let head = document.getElementsByTagName('head')[0];
        let script = document.createElement('script');
        script.setAttribute('async', 'async');
        script.src= scriptSrc;
        head.appendChild(script);
        script.onerror = function(){
          this.$alert("Load Google Map Failed.", {
            type: 'error'
          })
        };
        let that = this;
        script.onload = function(){
          that.initMap();
          that.fitBounds();
          that.drawMarkers();
        };
      },
      //初始化地图
      initMap(){
        this.infoWindow = new google.maps.InfoWindow();
        this.geoCoder = new google.maps.Geocoder();
        this.map = new google.maps.Map( this.$refs['map'], {zoom: 4, center: {lat: -25.363, lng: 131.044},});
      },
       //计算边界使其包含所有的marker
      fitBounds(){
        let north = -90;
        let south = 90;
        let west = 180;
        let east = -180;
        for(let i in this.gpsList){
          let gps = this.gpsList[i];
          north = Math.max(north, gps.lat);
          south = Math.min(south, gps.lat);
          west  = Math.min(west, gps.lng);
          east  = Math.max(east, gps.lng);
        }

        this.map.fitBounds(new google.maps.LatLngBounds(new google.maps.LatLng(north, west), new google.maps.LatLng(south, east)), 75);
      },
      /标记marker
      drawMarkers(){
        for(let i in this.positionList){
          let position = this.positionList[i];
          let marker = new google.maps.Marker({
            "position" : { "lat": + position.lat, "lng": + position.lng},
            "map": this.map,
            "timeList": position.timeList,
          });
          //给marker添加点击事件
          marker.addListener('click', ()=>{
            this.infoWindow.setContent("");
            this.openInfoWindow(marker);
          });
        }
      },
      //点击marker打开地址信息窗口
      openInfoWindow(marker){
        let lat = marker.position.lat();
        let lng = marker.position.lng();
        //根据经纬度反转码算出位置
        this.geoCoder.geocode(
          {'location': {lat: lat, lng: lng}},
          (response, status) => {
            if (status == 'OK' && response != null && response[1] != null) {
              let content = "<p>" + response[1].formatted_address + "</p>";
//              content += "<p>(" + lat.toFixed(5) +", " + lng.toFixed(5) + ")</p>";
              for(let i in marker.timeList){
                let time = marker.timeList[i];
                content += "<p>" + util.UTCtimestampToLocaleTime(time, +7) + "</p>";
              }
              this.infoWindow.setContent(content);
            }
          })
        this.infoWindow.open(this.map, marker)
      },

      closeInfoWindow(){
        this.infoWindow.setContent("");
        this.infoWindow.close();
      },

    },
  }
</script>
