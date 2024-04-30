<template>
  <div class="offline-amap-container">
    <div class="search-container">
      <input placeholder="请输入XYZ瓦片地址，例如：https://webst01.is.autonavi.com/appmaptile?style=6&x={x}&y={y}&z={z}"
        type="input" v-model="url" style="width: 688px;" />
      <button @click="initMap">加载瓦片</button>
      <button @click="drawRect">划范围</button>

      <button @click="isShow = true" v-if="rect">下载</button>
    </div>

    <div class="bottom-info" ref="info">
      <span>点击“划范围”在地图上绘制矩形</span>
      当前缩放级别： {{ zoom }},中心经纬度:{{ lng }},{{ lat }}
      <span v-if="rect">选中范围:{{ rectLngLat }}</span>
    </div>

    <div class="left-tool" v-if="rect">
      <button type="primary" @click="clearRect()">清空范围</button>
      <button type="primary" @click="onEditRectStart()">开始编辑范围</button>
      <button type="primary" @click="onEditRectEnd()">结束编辑范围</button>
    </div>
    <div id="canvas" class="canvas" ref="canvas"></div>

    <el-dialog v-model="isShow" title="下载地图瓦片" width="60%" append-to-body>
      <el-form label-width="100px">
        <el-form-item label="选中范围">
          <el-input :value="rectLngLat" readonly> </el-input>
        </el-form-item>
        <el-form-item label="中心点">
          <el-input :value="centerLnglat" readonly> </el-input>
        </el-form-item>
        <el-form-item label="路径规则">
          <el-input :value="rule" readonly> </el-input>
        </el-form-item>
      </el-form>
      <el-table v-if="rect && tableData" :data="tableData" height="400">
        <el-table-column prop="level" label="缩放级别"></el-table-column>
        <el-table-column prop="num" label="瓦片数量"></el-table-column>
        <el-table-column label="选中">
          <template #default="scope">
            <input type="checkbox" v-if="scope.row" v-model="zoomMap[scope.row.level]" />
          </template>
        </el-table-column>
      </el-table>
      <template #footer>
        <div style="text-align: right">
          <el-button type="primary" @click="download">下载</el-button>
        </div>
      </template>
    </el-dialog>
    <div class="loading" v-show="isLoading">下载进度：{{ process }}%</div>
  </div>
</template>
<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue';
import JSZip from 'jszip';
import { Map, View } from 'ol';
import TileLayer from 'ol/layer/WebGLTile';
import VectorLayer from 'ol/layer/Vector';
import { XYZ, Vector } from 'ol/source';
import { fromLonLat, toLonLat } from 'ol/proj';
import { MousePosition, OverviewMap, ScaleLine, ZoomToExtent } from 'ol/control';
import { Draw, Modify } from 'ol/interaction'
import { Polygon } from 'ol/geom';
import { Feature } from 'ol';
import Style from 'ol/style/Style';
import Stroke from 'ol/style/Stroke';
import Fill from 'ol/style/Fill';
import { ElMessage, ElMessageBox } from 'element-plus';

let map = null;
const url = ref('https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}');
const isShow = ref(false);
const isLoading = ref(false);
const process = ref(0);
const rect = ref(null);
const rule = ref('tiles/[z]/[y]/[x].png');
const zoomMap = ref({});
const lat = ref(39.90923);
const lng = ref(116.397428);
const zoom = ref(12);

const tableData = computed(() => {
  if (rect) {
    let list = [];
    const zooms = [...Array(20).keys()].slice(0, 19);
    zooms.forEach((z) => {
      list.push({ level: z, num: getTileCount(z) });
    });
    return list;
  }
  return [];
});

function getTileCount(z) {
  const latlngMin = toLonLat([rect.value[0], rect.value[3]]);
  const latlngMax = toLonLat([rect.value[2], rect.value[1]]);
  const xMin = lon2tile(latlngMin[0], z);
  const yMin = lat2tile(latlngMin[1], z);
  const xMax = lon2tile(latlngMax[0], z);
  const yMax = lat2tile(latlngMax[1], z);
  return (xMax - xMin + 1) * (yMax - yMin + 1);
}

const centerLnglat = computed(() => {
  if (rect) {
    return toLonLat([(rect.value[0] + rect.value[2]) / 2, (rect.value[1] + rect.value[3]) / 2]).toString();
  }
  return '';
});

const rectLngLat = computed(() => {
  if (rect) {
    return `左上角: ${toLonLat([rect.value[0], rect.value[3]]).toString()} 右下角: ${toLonLat([rect.value[2], rect.value[1]]).toString()}`;
  }
  return null;
});

onMounted(() => {
  initMap();
});

onUnmounted(() => {
  map = null;
});

function lon2tile(lon, zoom) {
  return (Math.floor((lon + 180) / 360 * Math.pow(2, zoom)));
}

function lat2tile(lat, zoom) {
  return (Math.floor((1 - Math.log(Math.tan(lat * Math.PI / 180) + 1 / Math.cos(lat * Math.PI / 180)) / Math.PI) / 2 * Math.pow(2, zoom)));
}

function downloadTile(x, y, z) {
  return new Promise((resolve, reject) => {
    fetch(url.value.replace('{x}', x).replace('{y}', y).replace('{z}', z))
      .then((res) => res.blob())
      .then((blob) => {
        resolve(blob);
      })
      .catch((err) => {
        reject(err);
      });
  });
}

function initMap() {
  if (map) {
    map.setTarget(null);
  }
  map = new Map({
    target: 'canvas',
    layers: [
      new TileLayer({
        source: new XYZ({
          url: url.value,
        }),
        projection: 'EPSG:3857'
      }),
    ],
    view: new View({
      center: fromLonLat([lng.value, lat.value]),
      zoom: zoom.value,
    })
  });

  map.addControl(new ZoomToExtent({
    extent: map.getView().calculateExtent(map.getSize())
  }))
  map.addControl(new MousePosition({
    coordinateFormat: (coordinate) => {
      return `经度: ${coordinate[0].toFixed(8)} 纬度: ${coordinate[1].toFixed(8)}`
    },
    projection: 'EPSG:4326',
  }))
  map.addControl(new OverviewMap({
    collapsed: false,
    layers: [
      new TileLayer({
        source: new XYZ({
          url: 'https://webrd02.is.autonavi.com/appmaptile?lang=zh_cn&size=1&scale=1&style=8&x={x}&y={y}&z={z}'
        })
      })
    ]
  }))
  map.addControl(new ScaleLine())
}

let draw
let graphicsLayer = new VectorLayer({
  title: "绘画图层",
  source: new Vector()
})
let rectLayer = new VectorLayer({
  title: "矩形图层",
  source: new Vector(),
  style: new Style({
    stroke: new Stroke({
      color: 'red',
      width: 2
    }),
    fill: new Fill({
      color: 'rgba(255, 0, 0, 0.1)'
    })
  })
})
function drawRect() {
  map.addLayer(graphicsLayer)
  map.addLayer(rectLayer)
  draw = new Draw({
    source: graphicsLayer.getSource(),
    type: 'Polygon'
  });

  map.addInteraction(draw);

  draw.on('drawend', (e) => {
    rect.value = e.feature.getGeometry().getExtent();
    map.removeInteraction(draw);
    graphicsLayer.getSource().removeFeature(e.feature);
    rectLayer.getSource().addFeature(new Feature({
      geometry: new Polygon([[
        [rect.value[0], rect.value[1]],
        [rect.value[0], rect.value[3]],
        [rect.value[2], rect.value[3]],
        [rect.value[2], rect.value[1]],
        [rect.value[0], rect.value[1]]
      ]]),
    }));
  });
}

function clearRect() {
  rect.value = null;
  rectLayer.getSource().clear();
  graphicsLayer.getSource().clear();
  map.removeLayer(graphicsLayer);
  map.removeLayer(rectLayer);
}

let modify
function onEditRectStart() {
  modify = new Modify({
    source: graphicsLayer.getSource()
  });
  map.addInteraction(modify);

  modify.on('modifyend', (e) => {
    rect.value = e.features.getArray()[0].getGeometry().getExtent();
    rectLayer.getSource().clear();
    rectLayer.getSource().addFeature(new Feature({
      geometry: new Polygon([[
        [rect.value[0], rect.value[1]],
        [rect.value[0], rect.value[3]],
        [rect.value[2], rect.value[3]],
        [rect.value[2], rect.value[1]],
        [rect.value[0], rect.value[1]]
      ]]),
    }));
  });
}

function onEditRectEnd() {
  map.removeInteraction(modify);
}

function download() {
  const latlngMin = toLonLat([rect.value[0], rect.value[3]]);
  const latlngMax = toLonLat([rect.value[2], rect.value[1]]);

  const list = [];
  for (let z = 0; z < 20; z++) {
    const xMin = lon2tile(latlngMin[0], z);
    const yMin = lat2tile(latlngMin[1], z);
    const xMax = lon2tile(latlngMax[0], z);
    const yMax = lat2tile(latlngMax[1], z);

    if (zoomMap.value[z]) {
      for (let x = xMin; x <= xMax; x++) {
        for (let y = yMin; y <= yMax; y++) {
          list.push({ x, y, z });
        }
      }
    }
  }

  ElMessageBox.confirm(`确定下载选中的瓦片吗？共${list.length}个瓦片 
  大概需要${(list.length * 0.1 / 6).toFixed(2)}秒`, '提示', {
    confirmButtonText: '确定',
    cancelButtonText: '取消',
    type: 'warning'
  }).then(() => {
    downloadTiles(list);
  }).catch(() => {
    ElMessage.info('已取消下载');
  });
}

async function downloadTiles(list) {
  isLoading.value = true;
  const total = list.length;
  let count = 0;
  let zip = new JSZip();
  for (let i = 0; i < list.length; i += 6) {
    let promises = [];
    if (i + 6 > list.length) {
      promises = list.slice(i, list.length).map(async (item) => {
        const blob = await downloadTile(item.x, item.y, item.z)
        zip.file(`${item.z}/${item.y}/${item.x}.png`, blob);
        count++;
        process.value = ((count / total) * 100).toFixed(2);
      });
    } else {
      promises = list.slice(i, i + 6).map(async (item) => {
        const blob = await downloadTile(item.x, item.y, item.z)
        zip.file(`${item.z}/${item.y}/${item.x}.png`, blob);
        count++;
        process.value = ((count / total) * 100).toFixed(2);
      });
    }
    await Promise.all(promises);

    if (count === total) {
      isLoading.value = false;
      zip.generateAsync({ type: "blob" }).then((content) => {
        const url = URL.createObjectURL(content);
        const a = document.createElement('a');
        a.href = url;
        a.download = 'tiles.zip';
        a.click();
        URL.revokeObjectURL(url);
        isLoading.value = false;
      });
    }
  }
}

</script>
<style lang="scss" scoped>
.loading {
  position: fixed;
  z-index: 99999;
  background-color: rgba(0, 0, 0, 0.5);
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 18px;
  height: 100%;
  width: 100%;
  top: 0px;
  left: 0px;
}

.offline-amap-container {
  position: absolute;
  height: 100%;
  width: 100%;
  color: black;

  button {
    border: none;
    background-color: dodgerblue;
    min-width: 100px;
    color: white;
    height: 40px;

    &:active {
      filter: brightness(1.3);
    }
  }

  .left-tool {
    position: fixed;
    top: 40%;
    left: 0px;
    z-index: 3;
    display: inline-flex;

    flex-direction: column;
    width: 100px;

    >button {
      width: 100%;
      margin: 0px;
    }

    >button:not(:last-child) {
      margin-bottom: 10px;
    }
  }

  .bottom-info {
    position: fixed;
    bottom: 0px;
    display: block;
    width: 100%;
    height: 32px;
    line-height: 32px;
    font-size: 14px;
    color: white;
    text-align: center;
    z-index: 2;
    background-color: black;
  }

  .search-container {
    position: fixed;
    top: 100px;
    width: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 2;
    height: 40px;

    button {
      margin: 0 4px;
    }

    input,
    select {
      background-color: white;
      border: solid #1e90ff 1px;
      height: 40px;
      padding: 0 8px;
      line-height: 1;
      outline: none;
      font-size: 16px;

      &::placeholder,
      &::-webkit-input-placeholder,
      &::-moz-placeholder {
        color: gray;
      }
    }
  }

  .canvas {
    height: 100%;
    width: 100%;
  }
}
</style>
