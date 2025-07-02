<template>
    <div class="map-container" id="map-container">
         <div id="myMap"></div>
         <!-- 添加可拖动图标 -->
         <div
            v-if="resize"
            class="resize-icon"
            @mousedown="startDrag"
            @touchstart="startDrag"

         >
            <svg t="1749453106330" class="icon" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" p-id="990" width="200" height="200"><path d="M389.20009955 708.01898549L170.96683732 926.25224651H356.65540741v103.56306254H-5.81530905V667.34459259h103.56306254v185.68857009l218.23326102-218.2591526 73.21908504 73.27086577z m-65.0893843-310.68918518L97.74775349 170.96683732V356.65540741H-5.81530905V-5.81530905h362.47071646v103.56306254h-185.68857009l226.36296299 226.36296176-73.21908506 73.21908506z m284.79842016-55.45801956L853.03316268 97.74775349H667.34459259V-5.81530905h362.47071646v362.47071646h-103.56306254v-185.68857009l-244.12402726 244.14991884-73.21908384-73.27086699z m81.34878459 275.14116384L926.25224651 853.03316268V667.34459259h103.56306254v362.47071646H667.34459259v-103.56306254h185.68857009l-236.02021809-235.99432651 73.24497541-73.24497541z" fill="#1296db" p-id="991"></path></svg>
        </div>

        <el-autocomplete
            v-if="autoComplete"
            v-model="tipinput"
            :fetch-suggestions="querySearch"
            :placeholder="autoComplatePlaceholder"
            :disabled="autoCompleteDisabled"
            popper-append-to-body="false"
            popper-class="map-search-autocomplete"
            @select="handleSelect"
        >
            <template #default="{ item }">
                <div class="name">{{ item.name }}</div>
                <div class="address">{{ item.address }}</div>
            </template>
        </el-autocomplete>

        <slot></slot>
    </div>
 </template>

 <script lang="ts" setup>
import { useProp } from 'element-plus';
import { computed, watch, onMounted, ref } from 'vue';

const props = withDefaults(defineProps<{
   autoComplete?: boolean;
   resize?: boolean;
   autoCompleteDisabled?: boolean;
   autoComplatePlaceholder?: string;
}>(), {
    autoComplete: false,
    resize: true,
    autoCompleteDisabled: false,
    autoComplatePlaceholder: '搜索地点...'
});
 // 定义AMap的类型
 interface AMapInstance {
   resize(): void;
 }
const tipinput = ref('');
 let map: AMapInstance | null = null;
 const isDragging = ref(false);
 const initialX = ref(0);
 const initialY = ref(0);
 const initialleft = ref(0);
 const initialBottom = ref(0);
 const initHeight = ref(0);
 const autoComplete = ref(null);
 onMounted(() => {
    // 假设AMap是全局变量，实际使用时需要正确导入
    AMap.plugin(['AMap.MoveAnimation', 'AMap.MouseTool', 'AMap.PlaceSearch','AMap.AutoComplete', 'AMap.CircleEditor'], function(){
        map = new (window as any).AMap.Map('myMap', {
            center: [103.797477, 30.08692], // 中心点经纬度
            zoom: 13, // 缩放级别
        });
        var placeSearch = new AMap.PlaceSearch({
        // 构造地点查询类
        pageSize: 5, // 单页显示结果条数
        pageIndex: 1, // 页码
        citylimit: true, // 是否强制限制在设置的城市内搜索
        map: map, // 展现结果的地图实例
        panel: "panel", // 结果列表将在此容器中进行展示。
        autoFitView: true, // 是否自动调整地图视野使绘制的 Marker点都处于视口的可见范围
    });


        if(props.autoComplete) {
            // 如果需要自动完成
            autoComplete.value = new AMap.AutoComplete({
                city: '全国', // 可以限定城市
                type: '地点' // 搜索类型
            });
        }



        // 使用类型断言告诉TypeScript我们确定这个emit会传递正确的参数
        emit('init', map as unknown as object);

        const mapContainer = document.getElementById('map-container');
        if (mapContainer) {
            initialleft.value = mapContainer.offsetLeft;
            const rect = mapContainer.getBoundingClientRect();
            initHeight.value = mapContainer.offsetHeight
        }

    });

});

const querySearch = (queryString: string, cb) => {
    if (!queryString) {
        cb([]);
        return;
    }
    autoComplete.value.search(queryString, (status: string, result: any) => {
        if (status === 'complete' && result.tips) {
            const items = result.tips.map((tip: any) => ({
                ...tip,
                value: tip.name,
                name: tip.name,
                address: tip.district + tip.address,
                location: tip.location,

                id: tip.id
            }));
            console.log(items)
            cb(items.filter(({ location }) => location));
        } else {
            cb([]);
        }
    });
}

const handleSelect = (item: Record<string, any>) => {
    emit('select', item);
    tipinput.value = '';
}

 const startDrag = (e: MouseEvent | TouchEvent) => {
     isDragging.value = true;
     initialX.value = ('touches' in e ? e.touches[0].clientX : e.clientX);
     initialY.value = ('touches' in e ? e.touches[0].clientY : e.clientY);
     document.addEventListener('mousemove', drag);
     document.addEventListener('mouseup', stopDrag);
 };

 const drag = (e: MouseEvent | TouchEvent) => {
     if (!isDragging.value) return;

     const currentX = ('touches' in e ? e.touches[0].clientX : e.clientX);
     const currentY = ('touches' in e ? e.touches[0].clientY : e.clientY);
     const deltaX = currentX - initialX.value;
     const deltaY = currentY - initialY.value;

     const mapContainer = document.getElementById('map-container');
     if (mapContainer) {
         // 修改宽度和高度计算方式，确保最小尺寸为100px
         const newLeft = Math.max(initialleft.value + deltaX, 0);
         const newBottom = Math.min(initialBottom.value - deltaY, initHeight.value);

         mapContainer.style.left = newLeft + 'px';
         mapContainer.style.bottom = newBottom + 'px';

         // 调整地图大小
       //  map && map.resize();
     }
 };

 const stopDrag = () => {
    isDragging.value = false;
    const mapContainer = document.getElementById('map-container')

    var parent = mapContainer.offsetParent; // 获取最近的定位父元素
    var parentRect = parent.getBoundingClientRect();
    var elementRect = mapContainer.getBoundingClientRect();
    var bottomValue = parentRect.bottom - elementRect.bottom; // 相对于父元素的底部位置



    initialleft.value = mapContainer?.offsetLeft || 0;
    document.removeEventListener('mousemove', drag);
    document.removeEventListener('mouseup', stopDrag);
    initialBottom.value = 0;
    mapContainer.style.bottom = 0

    emit('resize', {
        left: initialleft.value,
        bottom: initialBottom.value,
        height: elementRect.height,
    });
   //

 };

 const clearInput = () =>{
    tipinput.value = '';
   // autoComplete.value.clear();
 }


 const emit = defineEmits<{
     (e: 'resize', val: object): void;
     (e: 'init', val: object): void;
     (e: 'select', item: Record<string, any>): void;
 }>();

 defineExpose({
     map,
     querySearch,
     handleSelect,
     clearInput
 });
 </script>

 <style lang="scss" scoped>
 .map-container {
    resize: both;
     position: relative;
     width: 100%;
     z-index:999;
 }

 #myMap {
     width: 100%;
     height: 100%;
 }

 .resize-icon {
     position: absolute;
     left: 0;
     bottom: 0;
     padding: 5px;
     width: 20px;
     height: 20px;
     cursor: nwse-resize;
     background: #fff;
     z-index: 1000;
     svg {
         width: 100%;
         height: 100%;
     }
 }
 .name {}
 .address {
    margin-top: -10px;
    font-size: 12px;
    color: #999;

 }
 ::v-deep(.el-autocomplete){
    position: absolute !important;
    right: 20px;
    top: 20px;
    width: 300px;
 }
 </style>
