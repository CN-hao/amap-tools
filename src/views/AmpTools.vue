<script setup>
import MyMap from '@/components/MyMap/index.vue';
import { ElMessageBox } from 'element-plus';
import { onMounted ,ref,reactive, watch } from 'vue';

let waypointsMarkers=reactive([]); // 存储关键点的marker

const showPointData = ref('');
const polylineInfo=reactive({});

// watch(showPointData, (newValue) => {
//   if(showPointData.value === '') {
//     polylineInfo.length=0;
//   }else{
//     polylineInfo.length=polyline.getLength();
//   }
// }, { immediate: true });

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
polyline.on('click', () => {
        const length= polyline.getLength();
        ElMessageBox.alert(`路线总长${length}米`, '提示', {
            confirmButtonText: '确  定',
            type: 'info'
        });
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
  marker.on('dragging', function(e) {
      const position = marker.getPosition();
      polyline.setPath(polyline.getPath().map(item => {
            return item.toString() === lnglat.toString() ? position : item;
      }));
      lnglat[0] = position.lng;
      lnglat[1] = position.lat;
      // 强制更新waypointsMarkers引用 否则不触发watch监听事件
      waypointsMarkers.push({});
      waypointsMarkers.pop();
  });
    waypointsMarkers.push(reactive(marker));
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

const showPoint=(e)=>{
  if (waypointsMarkers.length === 0) {
    ElMessageBox.alert('请先添加关键点', '提示', {
      confirmButtonText: '确  定',
      type: 'warning'
    });
    return;
  }
  updateShowPointData();
}

const resetPoint = () => {
  ElMessageBox.confirm('确认要重置所有关键点吗？', '提示', {
    confirmButtonText: '确  定',
    cancelButtonText: '取  消',
    type: 'warning'
  }).then(() => {
    mapClear();
  });
};
const mapClear=()=>{
  waypointsMarkers.forEach(marker => map.remove(marker));
    waypointsMarkers.length = 0; // 清空数组
    try{
      polyline.setPath([]); // 清空折线
      polyline.hide(); // 隐藏折线
    }catch {
    }
    showPointData.value = '';
}

watch(waypointsMarkers, (newMarkers) => {
  updateShowPointData();
  polylineInfo.length = polyline.getLength();
}, { deep: true });

const updateShowPointData = () => {
  showPointData.value = waypointsMarkers.map(marker => {
    const position = marker.getPosition();
    return `${position.lng},${position.lat};`;
  }).join('');
};

const drawPoint = () => {
  if (showPointData.value === '') {
    ElMessageBox.alert('请先填入经纬度', '提示', {
      confirmButtonText: '确  定',
      type: 'warning'
    });
    return;
  }
  const points = showPointData.value.split(';').filter(item => item.trim() !== '');
  mapClear();
  points.forEach(point => {
    const [lng, lat] = point.split(',').map(Number);
    if (!isNaN(lng) && !isNaN(lat)) {
      addWayPoint({ lnglat: { lng, lat } });
    }
  });
};

onMounted(()=>{
})

</script>

<template>
      <MyMap class="main-container"
        @init="initMap"
        autoComplatePlaceholder="搜索关键词添加关键点"
        :autoComplete="true"
        @select="handleGetMapLocation"
        :resize="false"
      >
          <el-row class="inner ">
              <el-row  style="width: 100%;" :gutter="10">
                <el-col :span="16">
                  <el-input 
                    v-model="showPointData"
                    placeholder="请先选择关键点或填入经纬度" 
                    />
                </el-col>
                <el-col :span="4">
                    <el-button @click="drawPoint">绘制</el-button>
                </el-col>
                <el-col :span="4">
                    <el-button @click="resetPoint">重置</el-button>
                </el-col>
              </el-row>
              <el-row  style="width: 100%; margin-top: 10px;">
                  <el-card style="width: 100%;">
                      <template #header>
                          路线信息
                      </template>
                      总长度：{{ polylineInfo.length||0 }} 米
                  </el-card>
              </el-row>
          </el-row>
    </MyMap>
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
    width: 400px;

}

::v-deep(.el-textarea__inner) {
  white-space: pre-wrap;
  word-break: break-word;
}

</style>
