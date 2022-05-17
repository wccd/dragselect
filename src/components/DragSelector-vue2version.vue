<style lang='stylus'>
    .selectArea
        position absolute
        top 0
        left 0
        width 100px
        height 100px
        border 1px solid #66ccff
        background-color rgba(111, 189, 255, 0.2)
        pointer-events none
        z-index 9999
        
    .dragtool
        position absolute
        left 0
        top 0
        width 100%
        height 100%
        
</style>

<template> 
    <div class="selectArea" ref="selectArea" :style="{left: `${left}px`,top:`${top}px`,width:`${width}px`,height:`${height}px`}" v-show="showDrag"></div>
</template>

<script lang='ts'>
import { Component, Prop, Vue, Watch } from 'vue-property-decorator';

@Component({
    components:{
        
    }
})
export default class DragSelector extends Vue {
    @Prop()
    boxClassName
    @Prop()
    scrollDom
    @Prop()
    containerDom
    @Prop()
    preventDragSelect
    @Prop()
    isLazy
    
    showDrag: boolean = false;
    left: number = 0;
    top: number = 0;
    width: number = 0;
    height: number = 0;

    scroller: any

    dragStart(e: MouseEvent) {
        if(e.buttons != 1 || this.preventDragSelect) return
        this.scroller.bindWatch();
        let isMove = false;
        let moveTimer = 0;
        let baseL = e.clientX,
            baseT = e.clientY,
            baseScroll = this.scrollDom.scrollTop,
            dragDom = this.scrollDom,
            dragDomL = dragDom.getBoundingClientRect()['x'],
            dragDomT = dragDom.getBoundingClientRect()['y'],
            // move
            moveL = 0,
            moveT = 0,
            xOffset = 0,
            yOffset = 0,
            scrollT = 0,
            boxList: {
                index: number,
                dom: HTMLElement,
                position: number[]
            }[] = [],
            activeList: Array<number> = [];
        moveTimer = setTimeout(() => {
            isMove = true;
        }, 150);
        this.scroller.updateScrollDomData({
            top: dragDom.getBoundingClientRect()['y'],
            height: dragDom.getBoundingClientRect()['height'],
            scrollTop: dragDom.scrollTop,
            scrollHeight: dragDom.scrollHeight,
        })
        
        let lastChildNum = this.containerDom.children.length;
        let maxHeight =  this.scrollDom.scrollHeight; 
        if(lastChildNum != 0) { // 筛选需要判断位置的盒子
            for(let i = 0,index = 0; i < lastChildNum; i++) {
                let child = this.containerDom.children[i] as HTMLElement;
                if(child.className.indexOf(this.boxClassName) != -1) {
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
            maxHeight =  this.scrollDom.scrollHeight;
            if(lastChildNum != this.containerDom.children.length) {
                for(let i = lastChildNum,index = boxList[boxList.length-1].index+1; i < this.containerDom.children.length; i++) {
                    let child = this.containerDom.children[i] as HTMLElement;
                    if(child.className.indexOf(this.boxClassName) != -1) {
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
                lastChildNum = this.containerDom.children.length;
            }
        }

        isMove && this.$emit("dragMouseDown");

        let addSelect = (left: number, top: number, width: number, height: number) => {
            let arr = new Set();
            for(let i = 0; i < boxList.length; i++) {
                let p = boxList[i].position;
                if(left <= p[0] && left + width > p[0] || (left > p[0] && left < p[0] + p[2] && width > 0)) {
                    if(top <= p[1] && top + height > p[1] || (top > p[1] && top < p[1] + p[3] && height > 0)) {
                        arr.add(boxList[i].index)
                    } else {
                        arr.delete(boxList[i].index)
                    }
                }else {
                    arr.delete(boxList[i].index)
                }
            }
            activeList = Array.from(arr) as Array<number>;
            this.$emit("setActiveList", activeList)
        }
        let draw = () => {
            this.isLazy && lazyUpdateData();
            scrollT = this.scrollDom.scrollTop - baseScroll;
            if(xOffset <= 0){
                this.left = moveL - dragDomL;
                this.width = -xOffset;
            } else {
                this.left = baseL - dragDomL - 1;
                this.width = xOffset - 1;
            }
            if(yOffset + scrollT <= 0){
                this.top = moveT - dragDomT + baseScroll + scrollT;
                this.height = -yOffset - scrollT;
            } else {
                this.top = baseT - dragDomT + baseScroll - 1;
                this.height = yOffset - 1 + scrollT;
            }
            this.height = this.height > maxHeight - this.top ? maxHeight - this.top : this.height;
            addSelect(this.left + dragDomL, this.top + dragDomT, this.width, this.height);
        }
        let updateScrollTop = () => {
            this.scroller.updateScrollDomData({
                scrollTop: dragDom.scrollTop,
            })
        }
        document.onmousemove = (ev:MouseEvent) => {
            if(!isMove) return
            this.$emit("dragMouseMove");
            this.showDrag = true;
            moveL = ev.clientX;
            moveT = ev.clientY;
            xOffset = moveL - baseL;
            yOffset = moveT - baseT;
            draw();
        }

        document.onmouseup = (ev:MouseEvent) => {
            clearTimeout(moveTimer);
            let clickFile = (ev.target as HTMLElement).classList && (ev.target as HTMLElement).classList.value.indexOf('files-item') != -1;
            this.$emit("dragMouseUp", isMove, clickFile);
            this.showDrag = false;
            document.onmousemove = null;
            document.onmouseup = null;
            this.scrollDom.removeEventListener('scroll', draw)
            this.scrollDom.removeEventListener('scroll', updateScrollTop)
            this.scroller.removeWatch();
        }
        this.scrollDom.addEventListener('scroll', draw)
        this.scrollDom.addEventListener('scroll', updateScrollTop)
    }

    @Watch('scrollDom')
    watchdata(n, o) {
        if(n && this.scrollDom) {
            this.scrollDom.addEventListener('mousedown', this.dragStart);
            this.scroller = new DragSelectScroll(this.scrollDom)
        }
    }
}
// 拖拽边缘滚动,仅竖直方向
class DragSelectScroll {
    scrollDom = null
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
    watchFunc = null

    constructor(dom) {
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
    
    watchMove(ev) {
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

    getSpeed(offset) {
        const baseSpeed = 50;
        offset = offset > 200 ? 200 : offset;
        this.scrollSpeed = baseSpeed * Math.sin(Math.PI / 2 * offset / 200);
    }

    updateScrollDomData(data) {
        let keys = Object.keys(data);
        for(let item of keys) {
            this.scrollDomData[item] = data[item]
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
</script>