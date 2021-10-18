<template>
  <div
    class="selectArea"
    ref="selectArea"
    :style="{
      left: `${left}px`,
      top: `${top}px`,
      width: `${width}px`,
      height: `${height}px`,
    }"
    v-show="showDrag"
  ></div>
</template>

<script lang="ts">
import { ref, reactive, defineComponent, computed, watch } from "vue";
export default defineComponent({
  name: "dragTool",
  props: {
    boxClassName: {
      type: String,
      default: "box",
    },
    scrollDom: Object,
    containerDom: Object,
  },
  setup: (props, context) => {
    const boxClassName = ref(props.boxClassName);
    const scrollDom = computed({
      get: () => props.scrollDom as HTMLElement,
      set: (val) => context.emit("update:scrollDom", val),
    });
    const containerDom = computed({
      get: () => props.containerDom as HTMLElement,
      set: (val) => context.emit("update:containerDom", val),
    });
    type BoxList = {
      index: number;
      dom: HTMLElement;
      position: number[];
      cache: boolean;
    };
    type ActiveList = {
      arr: Array<number>;
    };
    watch(scrollDom, (nv, ov) => {
      nv && scrollDom.value.addEventListener("mousedown", dragStart);
    });
    let activeList: ActiveList = reactive({ arr: [] });
    let showDrag = ref(false);
    let left = ref(0),
      top = ref(0),
      width = ref(0),
      height = ref(0);
    let dragStart = (e: MouseEvent) => {
      let isMove = false;
      let moveTimer = 0;

      let baseL = e.clientX,
        baseT = e.clientY,
        baseScroll = scrollDom.value.scrollTop,
        boxList: BoxList[] = [],
        moveL = 0,
        moveT = 0,
        xOffset = 0,
        yOffset = 0,
        scrollT = 0;

      moveTimer = setTimeout(() => {
        isMove = true;
      }, 150);
      if (containerDom.value.children.length != 0) {
        // 筛选需要判断位置的盒子
        for (let i = 0, index = 0; i < scrollDom.value.children.length; i++) {
          let child = scrollDom.value.children[i] as HTMLElement;
          if (child.classList.value.indexOf(boxClassName.value) != -1) {
            let obj = {
              index: index,
              dom: child,
              position: [],
              cache: false,
            };
            boxList.push(obj);
            index++;
          }
        }
      }
      activeList.arr = []; // clean select
      context.emit("setActiveList", activeList.arr);

      let addSelect = (
        left: number,
        top: number,
        width: number,
        height: number
      ) => {
        let arr = new Set();
        for (let i = 0; i < boxList.length; i++) {
          if (boxList[i].cache) {
            let p = boxList[i].position;
            if (
              (left <= p[0] && left + width > p[0]) ||
              (left > p[0] && left < p[0] + p[2] && width > 0)
            ) {
              if (
                (top <= p[1] && top + height > p[1]) ||
                (top > p[1] && top < p[1] + p[3] && height > 0)
              ) {
                arr.add(boxList[i].index);
              } else {
                arr.delete(boxList[i].index);
              }
            } else {
              arr.delete(boxList[i].index);
            }
          } else {
            let domData = boxList[i].dom.getBoundingClientRect();
            let p = [
              domData.x,
              domData.y + baseScroll,
              domData.width,
              domData.height,
            ];
            boxList[i].position = p;
            boxList[i].cache = true;
            if (
              (left <= p[0] && left + width > p[0]) ||
              (left > p[0] && left < p[0] + p[2] && width > 0)
            ) {
              if (
                (top <= p[1] && top + height > p[1]) ||
                (top > p[1] && top < p[1] + p[3] && height > 0)
              ) {
                arr.add(boxList[i].index);
              } else {
                arr.delete(boxList[i].index);
              }
            } else {
              arr.delete(boxList[i].index);
            }
          }
        }
        activeList.arr = Array.from(arr) as Array<number>;
        context.emit("setActiveList", activeList.arr);
      };
      let draw = () => {
        scrollT = scrollDom.value.scrollTop - baseScroll;
        if (xOffset <= 0) {
          left.value = moveL;
          width.value = -xOffset;
        } else {
          left.value = baseL - 1;
          width.value = xOffset - 1;
        }
        if (yOffset + scrollT <= 0) {
          top.value = moveT + baseScroll + scrollT;
          height.value = -yOffset - scrollT;
        } else {
          top.value = baseT + baseScroll - 1;
          height.value = yOffset - 1 + scrollT;
        }
        addSelect(left.value, top.value, width.value, height.value);
      };

      document.onmousemove = (ev: MouseEvent) => {
        if (!isMove) return;
        showDrag.value = true;
        (moveL = ev.clientX),
          (moveT = ev.clientY),
          (xOffset = moveL - baseL),
          (yOffset = moveT - baseT),
          draw();
      };

      document.onmouseup = (ev: MouseEvent) => {
        clearTimeout(moveTimer);
        showDrag.value = false;
        document.onmousemove = null;
        document.onmouseup = null;
        scrollDom.value.removeEventListener("scroll", draw);
      };
      scrollDom.value.addEventListener("scroll", draw);
    };
    return { showDrag, left, top, width, height, dragStart };
  },
});
</script>

<style scoped>
.selectArea {
  position: absolute;
  top: 0;
  left: 0;
  width: 100px;
  height: 100px;
  border: 1px solid #66ccff;
  background-color: rgba(111, 189, 255, 0.2);
  pointer-events: none;
}
.dragtool {
  position: absolute;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
}
</style>
