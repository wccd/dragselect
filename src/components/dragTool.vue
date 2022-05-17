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
// 拖拽边缘滚动,仅竖直方向
type DomData = {
  top?: number,
  height?: number,
  scrollTop?: number,
  scrollHeight?: number,
  maxScrollTop?: number
}
class DragSelectScroll {
    scrollDom!: HTMLElement
    scrollSpeed = 10
    scrollDomData = {
        top: 0,
        height: 0,
        scrollTop: 0,
        scrollHeight: 0,
        maxScrollTop: 0
    }
    yScrollKey = 0
    yEventFlag = false
    outRange = -1 // 0 上滚，1 下滚 2 左滚 3 右滚
    watchFunc!:any

    constructor(dom: HTMLElement) {
        this.scrollDom = dom;
    }

    bindWatch() {
        this.watchFunc = this.watchMove.bind(this);
        document.addEventListener('mousemove', this.watchFunc)
        this.resetScroll();
    }

    removeWatch() {
        document.removeEventListener('mousemove', this.watchFunc)
        this.resetScroll();
    }

    resetScroll() {
        this.outRange = -1
        this.yEventFlag = false;
        cancelAnimationFrame(this.yScrollKey)
    }
    
    watchMove(ev: MouseEvent) {
        let moveT = ev.clientY;
        if(moveT < this.scrollDomData.top) {
            this.getSpeed(this.scrollDomData.top - moveT);
            this.outRange = 0;
        } else if(moveT > this.scrollDomData.top + this.scrollDomData.height) {
            this.getSpeed(moveT - this.scrollDomData.top - this.scrollDomData.height);
            this.outRange = 1;
        } else {
            this.resetScroll();
            return
        }

        if(this.outRange != -1 && !this.yEventFlag) {
            this.startScroll();
        }
    }

    getSpeed(offset: number) {
        const baseSpeed = 50;
        offset = offset > 200 ? 200 : offset;
        this.scrollSpeed = baseSpeed * Math.sin(Math.PI / 2 * offset / 200);
    }

    updateScrollDomData(data:DomData) {
        let keys = Object.keys(data);
        for(let item of keys) {
            this.scrollDomData[item as keyof DomData] = data[item as keyof DomData] || 0
        }
    }

    scrollTop() {
        let height = this.scrollDomData.scrollTop - this.scrollSpeed;
        height = height <= 0 ? 0 : height;
        this.scrollDom.scrollTo(0, height);
        this.updateScrollDomData({
            scrollTop: height,
            scrollHeight: this.scrollDom.scrollHeight, // 懒加载容器滚动高度会变化
        })
        this.yScrollKey = requestAnimationFrame(this.scrollTop.bind(this))
    }

    scrollBottom() {
        this.scrollDomData.maxScrollTop = this.scrollDomData.scrollHeight - this.scrollDomData.height;
        let height = this.scrollDomData.scrollTop + this.scrollSpeed;
        height = height > this.scrollDomData.maxScrollTop ? this.scrollDomData.maxScrollTop : height;
        this.scrollDom.scrollTo(0, height);
        this.updateScrollDomData({
            scrollTop: height,
            scrollHeight: this.scrollDom.scrollHeight, // 懒加载容器滚动高度会变化
        })
        this.yScrollKey = requestAnimationFrame(this.scrollBottom.bind(this))
    }

    startScroll() {
        if(this.outRange == 0) {
            this.yEventFlag = true;
            this.yScrollKey = requestAnimationFrame(this.scrollTop.bind(this))
        } else if(this.outRange == 1) {
            this.yEventFlag = true;
            this.yScrollKey = requestAnimationFrame(this.scrollBottom.bind(this))
        }
    }
}
export default defineComponent({
  name: "dragTool",
  props: {
    boxClassName: {
      type: String,
      default: "box",
    },
    scrollDom: Object,
    containerDom: Object,
    preventDragSelect: Boolean,
    isLazy: Boolean
  },
  setup: (props, context) => {
    const boxClassName = ref(props.boxClassName);
    const preventDragSelect = ref(props.preventDragSelect);
    const isLazy = ref(props.isLazy); // 宿主懒加载数据，导致滚动区域内容更新的需要刷新数据
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
    };
    type ActiveList = {
      arr: Array<number>;
    };
    let scroller:any
    watch(scrollDom, (nv, ov) => {
      nv && scrollDom.value.addEventListener("mousedown", dragStart);
      scroller = new DragSelectScroll(scrollDom.value)
    });
    let activeList: ActiveList = reactive({ arr: [] });
    let showDrag = ref(false);
    let left = ref(0),
      top = ref(0),
      width = ref(0),
      height = ref(0);
    let dragStart = (e: MouseEvent) => {
      if(e.buttons != 1 || preventDragSelect.value) return
      scroller.bindWatch();
      let isMove = false;
      let moveTimer = 0;

      let baseL = e.clientX,
        baseT = e.clientY,
        baseScroll = scrollDom.value.scrollTop,
        boxList: BoxList[] = [],
        dragDom = scrollDom.value,
        dragDomL = dragDom.getBoundingClientRect()['x'],
        dragDomT = dragDom.getBoundingClientRect()['y'],
        moveL = 0,
        moveT = 0,
        xOffset = 0,
        yOffset = 0,
        scrollT = 0;

      moveTimer = setTimeout(() => {
        isMove = true;
      }, 150);
      scroller.updateScrollDomData({
          top: dragDom.getBoundingClientRect()['y'],
          height: dragDom.getBoundingClientRect()['height'],
          scrollTop: dragDom.scrollTop,
          scrollHeight: dragDom.scrollHeight,   
      })

      let lastChildNum = containerDom.value.children.length;
      let maxHeight =  scrollDom.value.scrollHeight;
      if(lastChildNum != 0) { // 筛选需要判断位置的盒子
          for(let i = 0,index = 0; i < lastChildNum; i++) {
            let child = containerDom.value.children[i] as HTMLElement;
            if(child.className.indexOf(boxClassName.value) != -1) {
                let domData = child.getBoundingClientRect();
                let p = [domData['x'], domData['y']+baseScroll, domData.width, domData.height];
                let obj = {
                    index: index,
                    dom: child,
                    position: p,
                }
                boxList.push(obj);
                index++
            }
          }
      }
      let lazyUpdateData = () => {
          maxHeight =  scrollDom.value.scrollHeight;
          if(lastChildNum != containerDom.value.children.length) {
              for(let i = lastChildNum,index = boxList[boxList.length-1].index+1; i < containerDom.value.children.length; i++) {
                  let child = containerDom.value.children[i] as HTMLElement;
                  if(child.className.indexOf(boxClassName.value) != -1) {
                      let domData = child.getBoundingClientRect();
                      let p = [domData['x'], domData['y']+baseScroll, domData.width, domData.height];
                      let obj = {
                          index: index,
                          dom: child,
                          position: p,
                      }
                      boxList.push(obj);
                      index++
                  }
              }
              lastChildNum = containerDom.value.children.length;
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
        }
        activeList.arr = Array.from(arr) as Array<number>;
        context.emit("setActiveList", activeList.arr);
      };
      let draw = () => {
        isLazy.value && lazyUpdateData();
        scrollT = scrollDom.value.scrollTop - baseScroll;
        if (xOffset <= 0) {
          left.value = moveL - dragDomL;
          width.value = -xOffset;
        } else {
          left.value = baseL - dragDomL - 1;
          width.value = xOffset - 1;
        }
        if (yOffset + scrollT <= 0) {
          top.value = moveT - dragDomT + baseScroll + scrollT;
          height.value = -yOffset - scrollT;
        } else {
          top.value = baseT - dragDomT + baseScroll - 1;
          height.value = yOffset - 1 + scrollT;
        }
        height.value = height.value > maxHeight - top.value ? maxHeight - top.value : height.value;
        addSelect(left.value + dragDomL, top.value + dragDomT, width.value, height.value);
      };
      let updateScrollTop = () => { // 滚动后更新滚动高度
          scroller.updateScrollDomData({
              scrollTop: dragDom.scrollTop,
          })
      }

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
        scrollDom.value.removeEventListener('scroll', updateScrollTop)
      };
      scrollDom.value.addEventListener("scroll", draw);
      scrollDom.value.addEventListener('scroll', updateScrollTop)
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
