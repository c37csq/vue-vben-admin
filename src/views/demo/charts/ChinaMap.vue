<template>
  <div class="wrapper">
    <div id="container"></div>
    <div id="chinaMap">
      <div class="chinaMapButton">
        <a-button @click="handleBack">返回</a-button>
      </div>
      <div class="chinaMapChart" ref="chinaMapRef"></div>
    </div>
  </div>
</template>

<script lang="ts">
  import { defineComponent, onMounted, ref, Ref } from 'vue';
  import { registerMap } from 'echarts';
  import { useECharts } from '/@/hooks/web/useECharts';

  export default defineComponent({
    name: 'ChinaMap',
    setup() {
      const chinaMapRef = ref<HTMLDivElement | null>(null);
      const { getInstance, setOptions } = useECharts(chinaMapRef as Ref<HTMLDivElement>);
      const map = ref({}) as Ref<any>;
      const opts = ref({}) as Ref<{
        subdistrict: number; //返回下一级行政区
        showbiz: boolean; //最后一级返回街道信息
      }>;
      const district = ref({}) as Ref<any>;
      const polygons = ref([]) as Ref<any>;
      const chinaMapData = ref([]) as Ref<any>;
      const cityName = ref('中国');
      const cityCode = ref('100000') as Ref<string>;
      const geoJsonData = ref('') as Ref<any>;
      const echartsClickParamsList = ref([]) as Ref<any>;
      const mapLoading = ref<boolean>(false);

      function getData(data: any, adcode: string) {
        mapLoading.value = true;
        const bounds = data.boundaries;
        // const name = data.name;
        if (bounds) {
          for (let i = 0, l = bounds.length; i < l; i++) {
            // @ts-ignore
            const polygon = new AMap.Polygon({
              map: map.value,
              strokeWeight: 1,
              strokeColor: '#0091ea',
              fillColor: '#80d8ff',
              fillOpacity: 0.2,
              path: bounds[i],
            });
            polygons.value.push(polygon);
          }
          map.value.setFitView();
        }

        const subList = data.districtList;
        if (subList) {
          // if (name === '上海市' || name === '北京市' || name === '天津市' || name === '重庆市') {
          //   console.log('直辖市');
          // } else {
          // }
          const curLevel = subList[0].level;
          if (curLevel === 'street') {
            const mapJsonList = geoJsonData.value.features;
            const mapJson = {} as any;
            for (let i in mapJsonList) {
              if (mapJsonList[i].properties.name === cityName.value) {
                mapJson.type = 'FeatureCollection';
                mapJson.features = [].concat(mapJsonList[i]);
              }
            }
            chinaMapData.value = [];
            chinaMapData.value.push({
              name: cityName.value,
              value: Math.random() * 100,
              level: curLevel,
            });
            loadMap(cityName.value, mapJson);
            return;
          }
          chinaMapData.value = [];
          for (let i = 0, l = subList.length; i < l; i++) {
            const name = subList[i].name;
            const cityCode = subList[i].adcode;
            chinaMapData.value.push({
              name: name,
              value: Math.random() * 100,
              cityCode: cityCode,
              level: curLevel,
            });
          }
          loadMapData(adcode);
        }
      }

      function loadMap(mapName: string, data: any) {
        if (data) {
          registerMap(mapName, data);
          setOptions({
            visualMap: [
              {
                min: 0,
                max: 100,
                left: 'left',
                top: 'bottom',
                text: ['高', '低'],
                calculable: false,
                orient: 'horizontal',
                inRange: {
                  color: ['#e0ffff', '#006edd'],
                  symbolSize: [30, 100],
                },
              },
            ],
            tooltip: {
              trigger: 'item',
              backgroundColor: 'rgba(0, 0, 0, .6)',
              textStyle: {
                color: '#fff',
                fontSize: 12,
              },
            },
            series: [
              {
                name: '总数',
                type: 'map',
                map: cityName.value,
                selectedMode: 'single',
                itemStyle: {
                  areaColor: '#2f82ce',
                  borderColor: '#0DAAC1',
                },
                zoom: 1.2,
                data: chinaMapData.value,
              },
            ],
          });
        }
        mapLoading.value = false;
      }

      function loadMapData(areaCode) {
        // @ts-ignore
        AMapUI.loadUI(['geo/DistrictExplorer'], (DistrictExplorer) => {
          //创建一个实例
          // @ts-ignore
          const districtExplorer = (window.districtExplorer = new DistrictExplorer({
            eventSupport: true, //打开事件支持
            map: map.value,
          }));

          districtExplorer.loadAreaNode(areaCode, (error, areaNode) => {
            if (error) {
              console.error(error);
              return;
            }
            const mapJson = {} as any;
            mapJson.type = 'FeatureCollection';
            mapJson.features = areaNode.getSubFeatures();
            loadMap(cityName.value, mapJson);
            geoJsonData.value = mapJson;
          });
        });
      }

      function handleBack() {
        if (mapLoading.value) return;
        if (echartsClickParamsList.value.length === 0) {
          return;
        } else {
          echartsClickParamsList.value.pop();
          const params = echartsClickParamsList.value.pop();
          if (!params) {
            showChinaMap();
          } else {
            echartsMapClick(params);
          }
        }
      }

      function echartsMapClick(params: any) {
        if (
          (echartsClickParamsList.value[0] &&
            (echartsClickParamsList.value[0].name === '上海市' ||
              echartsClickParamsList.value[0].name === '北京市' ||
              echartsClickParamsList.value[0].name === '重庆市' ||
              echartsClickParamsList.value[0].name === '天津市')) ||
          mapLoading.value ||
          params.data.level === 'street'
        ) {
          return;
        }
        //清除地图上所有覆盖物
        for (let i = 0, l = polygons.value.length; i < l; i++) {
          polygons.value[i].setMap(null);
        }
        echartsClickParamsList.value.push(params);
        cityName.value = params.data.name;
        cityCode.value = params.data.cityCode;
        district.value.setLevel(params.data.level); //行政区级别
        district.value.setExtensions('all');

        //行政区查询
        //按照adcode进行查询可以保证数据返回的唯一性
        district.value.search(cityCode.value, (status, result) => {
          console.log(result);
          if (status === 'complete') {
            getData(result.districtList[0], cityCode.value);
          }
        });
      }

      function showChinaMap() {
        // @ts-ignore
        map.value = new AMap.Map('container', {
          resizeEnable: true,
          center: [116.30946, 39.937629],
          zoom: 3,
        });
        opts.value = {
          subdistrict: 1, //返回下一级行政区
          showbiz: false, //最后一级返回街道信息
        };
        // @ts-ignore
        district.value = new AMap.DistrictSearch(opts.value);
        district.value.search('中国', (status, result) => {
          if (status == 'complete') {
            getData(result.districtList[0], '100000');
          }
        });
      }

      onMounted(() => {
        showChinaMap();
        getInstance()!.on('click', (e) => {
          echartsMapClick(e);
        });
      });

      return {
        chinaMapRef,

        handleBack,
      };
    },
  });
</script>

<style scoped lang="less">
  .wrapper {
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: space-between;

    #container {
      width: 50%;
      height: 100%;
    }

    #chinaMap {
      width: 50%;
      height: 100%;
      display: flex;
      flex-direction: column;
      justify-content: flex-start;

      .chinaMapButton {
        display: flex;
        justify-content: flex-end;
        width: 100%;
        padding: 12px 20px 12px 0;
      }

      .chinaMapChart {
        height: 100%;
        width: 100%;
      }
    }
  }
</style>
