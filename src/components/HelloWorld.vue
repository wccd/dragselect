<template>
  <div class="container">
    <div class="dragbox" ref="scrollDom">
      <dragTool
        :boxClassName="'box'"
        :scrollDom="scrollDom"
        :containerDom="scrollDom"
        @setActiveList="setActiveList"
      >
      </dragTool>
      <div
        class="box"
        :class="{ active: activeList.arr.includes(item.id) }"
        v-for="item in list"
        :key="item.id"
      >
        {{ item.label }}
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { ref, reactive, defineComponent } from "vue";
import dragTool from "./dragTool.vue";
export default defineComponent({
  name: "HelloWorld",
  components: {
    dragTool,
  },
  setup: () => {
    type ActiveList = {
      arr: Array<number>;
    };

    let list: Array<Object> = reactive([]);
    let activeList: ActiveList = reactive({ arr: [] });
    let scrollDom = ref({} as HTMLElement);
    for (let i = 0; i < 1000; i++) {
      let obj = {
        id: i,
        label: `block-${i}`,
      };
      list.push(obj);
    }
    let setActiveList = (list: Array<number>) => {
      activeList.arr = list;
    };
    return { list, activeList, scrollDom, setActiveList };
  },
});
</script>

<style scoped>
div {
  padding: 0;
  margin: 0;
  box-sizing: border-box;
}
.container {
  display: flex;
  width: 100%;
  height: 100%;
}
.dragbox {
  position: relative;
  display: flex;
  flex-wrap: wrap;
  height: 100%;
  overflow: auto;
  padding: 50px;
  box-shadow: 0 8px 20px 0 rgba(0, 0, 0, 0.2);
}
.box {
  width: 130px;
  height: 80px;
  margin: 0 20px 20px 0;
  background: #fff;
  border-radius: 3px;
  text-align: center;
  line-height: 80px;
  font-size: 14px;
  border: 1px solid transparent;
  box-shadow: 0 8px 20px 0 rgba(0, 0, 0, 0.2);
  user-select: none;
}
.active {
  border-color: #66ccff;
}
</style>
