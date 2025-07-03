<script setup>
import MyMap from '@/components/MyMap/index.vue';
import { ElMessageBox } from 'element-plus';
import { onMounted ,ref,reactive, watch} from 'vue';

let waypointsMarkers=reactive([]); // 存储关键点的marker

const showData = ref('');

const polyline = new AMap.Polyline({
  isOutline: true,
  outlineColor: '#ffeeee',
  borderWeight: 1,
  strokeColor: '#3366FF', // 线颜色
  strokeOpacity: 1,
  strokeWeight: 3, // 线宽
  strokeStyle: 'solid',
  lineJoin: 'round' // 折线拐点连接处样式
});


let map = null
const initMap = (mapInstance) => {
    map = mapInstance;
    map.on('click', addWayPoint);
    //map.on('dblclick', showPoint);
    map.on('rightclick',showPoint);
    map.add(polyline);
};

const addWayPoint = e => {
    // 不存在 marker：创建新
    const lnglat = [e.lnglat.lng, e.lnglat.lat]; // 获取点击坐标marker
    let marker = new AMap.Marker({
        position: lnglat,
        map: map,
        draggable: true // 可拖动
    });
    marker.setLabel({
        direction:'top',
        offset: new AMap.Pixel(10, 0),  //设置文本标注偏移量
        content: `<div>关键点</div>`, //设置文本标注内容
    });
    map.setFitView()
    marker.on('click', async() => {
        await ElMessageBox.confirm('确认要删除该关键点吗？', '提示', {
            confirmButtonText: '确  定',
            cancelButtonText: '取  消',
            type: 'warning'
        });
        map.remove(marker)
        polyline.setPath(polyline.getPath().filter(item => item.toString() !== lnglat.toString())); // 从折线中移除
        waypointsMarkers.splice(waypointsMarkers.indexOf(marker), 1)
    });
    marker.on('dragend', function(e) {
      const position = marker.getPosition();
      polyline.setPath(polyline.getPath().map(item => {
            return item.toString() === lnglat.toString() ? position : item;
      })); // 更新折线中的点 
      lnglat[0] = position.lng;
      lnglat[1] = position.lat;
  });
    waypointsMarkers.push(marker);
    polyline.setPath([...(polyline.getPath() || []), lnglat]);
    polyline.show(); // 显示折线
}
const handleGetMapLocation = info => {
    const { location, name, address } = info;
    const lnglat = [location.lng, location.lat]; // 获取经纬度
    addWayPoint({
        lnglat: location,
        name: name,
        address: address
    });
}

const showPoint=()=>{
  if (waypointsMarkers.length === 0) {
    ElMessageBox.alert('请先添加关键点', '提示', {
      confirmButtonText: '确  定',
      type: 'warning'
    });
    return;
  }
  showData.value= waypointsMarkers.map(marker => {
    const position = marker.getPosition();
    return `${position.lng},${position.lat};`;
  }).join('\n');
}

const resetPoint = () => {
  ElMessageBox.confirm('确认要重置所有关键点吗？', '提示', {
    confirmButtonText: '确  定',
    cancelButtonText: '取  消',
    type: 'warning'
  }).then(() => {
    waypointsMarkers.forEach(marker => map.remove(marker));
    waypointsMarkers.length = 0; // 清空数组
    try{
      polyline.setPath([]); // 清空折线
      polyline.hide(); // 隐藏折线
    }catch {
    }
    showData.value = '';
  });
};

watch(waypointsMarkers, (newMarkers) => {
  showData.value = newMarkers.map(marker => {
    const position = marker.getPosition();
    return `${position.lng},${position.lat};`;
  }).join('\n');
});

onMounted(()=>{
})

function handSelect(item) {
  console.log(item);
  // 空函数
}
</script>

<template>
    <div >
      <MyMap class="main-container"
        @init="initMap"
        autoComplatePlaceholder="搜索关键词添加关键点"
        :autoComplete="true"
        @select="handleGetMapLocation"
        :resize="false"
      >
          <div class="inner">
              <div>
                  <el-input 
                :rows="5"
                :autosize="{ minRows: 5, maxRows: 10 }"
                v-model="showData"
                :show-word-limit="true" 
                type="textarea"
                placeholder="请先选择关键点" 
                readonly/>
              </div>
              <div style="margin-top: 10px;">
                  <el-button @click="resetPoint">重置</el-button>
              </div>
          </div>
    </MyMap>
    </div>
</template>

<style scoped>
html,
body,
.main-container{
    height: 95vh;
}
.inner {
    position: absolute;
    left: 20px;
    top: 20px;
    width: 300px;

}

::v-deep(.el-textarea__inner) {
  white-space: pre-wrap;
  word-break: break-word;
}

</style>
