<template>
    <div class="drag-tree-table">
        <div class="drag-tree-table-header">
          <column
            v-for="(item, index) in data.columns"
            :width="item.width"
            :key="index" >
            {{item.title}}
          </column>
        </div>
        <div class="drag-tree-table-body" @dragover="draging" @dragend="drop">
          <row depth="0" :columns="data.columns"
            :model="item" v-for="(item, index) in data.lists" :key="index">
        </row>
        </div>

    </div>
</template>

<script>
  import row from './row.vue'
  import column from './column.vue'
  import space from './space.vue'
  document.body.ondrop = function (event) {
    event.preventDefault();
    event.stopPropagation();
  }
  export default {
    name: "dragTreeTable",
    components: {
        row,
        column,
        space
    },
    props: {
      data: Object,
      onDrag: Function
    },
    data() {
      return {
        treeData: [],
        dragX: 0,
        dragY: 0,
        dragId: '',
        targetId: '',
        whereInsert: ''
      }
    },
    methods: {
      // 拖动时目标元素的页面偏移量，x轴
      getElementLeft(element) {
        var actualLeft = element.offsetLeft;
        var current = element.offsetParent;
        // 页面偏移：要知道某个元素在页面上的偏移量，将这个元素的offsetLeft和offsetTop与其offsetParent的相同属性相加，并加上offsetParent的相应方向的边框，如此循环直到根元素，就可以得到元素到页面的偏移量
        // 1.元素自身有fixed定位，父元素不存在定位，则offsetParent的结果为null（firefox中为：body，其他浏览器返回为null）
        // 2.元素自身无fixed定位，且父元素也不存在定位，offsetParent为<body>元素
        // 3.元素自身无fixed定位，且父元素存在定位，offsetParent为离自身最近且经过定位的父元素
        // 4.<body>元素的offsetParent是null
        while (current !== null){
          actualLeft += current.offsetLeft;
          current = current.offsetParent;
        }
        return actualLeft
      },
      // 拖动时目标元素的页面偏移量，y轴
      getElementTop(element) {
        var actualTop = element.offsetTop;
        var current = element.offsetParent;
        while (current !== null) {
          actualTop += current.offsetTop;
          current = current.offsetParent;
        }
        return actualTop
      },
      //事件主体是目标元素，在被拖放在某元素内移动时触发。
      draging(e) {
        if (e.pageX == this.dragX && e.pageY == this.dragY) return  // 拖动时记录开始位置，当位置一样时不执行后面与语句，主要提高性能(不过感觉视乎没多大作用，拖动时后很小几率起作用)
        this.dragX = e.pageX
        this.dragY = e.pageY
        this.filter(e.pageX, e.pageY)
      },
      drop(event) {
        this.clearHoverStatus()
        this.resetTreeData()
      },
      filter(x,y) {
        var rows = document.querySelectorAll('.tree-row')
        this.targetId = undefined
        const dragOriginElementTop = this.getElementTop(dragParentNode)
        const dragOriginElementLeft = this.getElementLeft(dragParentNode)
        const dragW = dragOriginElementLeft + dragParentNode.clientWidth
        const dragH = dragOriginElementTop + dragParentNode.clientHeight
        if (x >= dragOriginElementLeft && x <= dragW && y >= dragOriginElementTop && y <= dragH) {
          // 当前正在拖拽原始块不允许插入,还在原始区域时
          return
        }
        for(let i=0; i < rows.length; i++) {
          const row = rows[i]
          const rx = this.getElementLeft(row);
          const ry = this.getElementTop(row);
          const rw = row.clientWidth;
          const rh = row.clientHeight;
          if (x > rx && x < (rx + rw) && y > ry && y < (ry + rh)) { // 在允许拖放的范围内
            const diffY = y - ry
            const hoverBlock = row.children[row.children.length - 1]
            hoverBlock.style.display = 'block' // 拖动到每个目标区域时，目标区域所呈现的灰色
            let targetId = row.getAttribute('tree-id')
            this.targetId = targetId
            let whereInsert = ''
            var rowHeight = document.getElementsByClassName('tree-row')[0].clientHeight
            if (diffY/rowHeight > 3/4) {
              if (hoverBlock.children[2].style.opacity !== '0.5') {
                this.clearHoverStatus()
                hoverBlock.children[2].style.opacity = 0.5  // 拖动到目标区域时在底部，把目标区域顶部透明，呈现出被拖拽的元素的背景色,此时的hoverBlock.style.display为none好奇怪
              }
              whereInsert = 'bottom'
            } else if (diffY/rowHeight > 1/4) {
              if (hoverBlock.children[1].style.opacity !== '0.5') {
                this.clearHoverStatus()
                hoverBlock.children[1].style.opacity = 0.5
              }
              whereInsert = 'center'
            } else {
              if (hoverBlock.children[0].style.opacity !== '0.5') {
                this.clearHoverStatus()
                hoverBlock.children[0].style.opacity = 0.5
              }
              whereInsert = 'top'
            }
            this.whereInsert = whereInsert
          }
        }
      },
      clearHoverStatus() {
        var rows = document.querySelectorAll('.tree-row')
        for(let i=0; i < rows.length; i++) {
          const row = rows[i]
          const hoverBlock = row.children[row.children.length - 1]
          hoverBlock.style.display = 'none'
          hoverBlock.children[0].style.opacity = 0.1
          hoverBlock.children[1].style.opacity = 0.1
          hoverBlock.children[2].style.opacity = 0.1
        }
      },
      // 重置列表数据
      resetTreeData() {
        if (this.targetId === undefined) return
        const newList = []
        const curList = this.data.lists
        const _this = this
        function pushData(curList, needPushList) {  // 遍历原数组,
          // let order = 0    // order好像没作用
          for( let i = 0; i < curList.length; i++) {
            const item = curList[i]
            var obj = _this.deepClone(item)
            obj.lists = []
            if (_this.targetId == item.id) {
              const curDragItem = _this.getCurDragItem(_this.data.lists, window.dragId)  // 得到当前被拖拽的item
              if (_this.whereInsert === 'top') {
                curDragItem.parent_id = item.parent_id
                // curDragItem.order = order
                needPushList.push(curDragItem)
                // order = order + 1
                // obj.order = order
                needPushList.push(obj)
              } else if (_this.whereInsert === 'center'){
                curDragItem.parent_id = item.id
                // curDragItem.order = 0
                obj.lists.push(curDragItem)  //
                needPushList.push(obj)
              } else {
                // order = order + 1
                // obj.order = order
                curDragItem.parent_id = item.parent_id
                needPushList.push(obj)
                needPushList.push(curDragItem)
              }
            } else {
              if (window.dragId != item.id){
                // obj.order = order
                needPushList.push(obj)
              }
            }
            // order = needPushList.length
            if (item.lists && item.lists.length) {
              pushData(item.lists, obj.lists)
            }
          }
        }
        pushData(curList, newList)
        this.onDrag(newList)
      },
      deepClone (aObject) {
        if (!aObject) {
          return aObject;
        }
        var bObject, v, k;
        bObject = Array.isArray(aObject) ? [] : {};
        for (k in aObject) {
          v = aObject[k];
          bObject[k] = (typeof v === "object") ? this.deepClone(v) : v;
        }
        return bObject;
      },
      getCurDragItem(lists, id) {
        var curItem = null
        var _this = this
        function getchild(curList) {
          for( let i = 0; i < curList.length; i++) {
            var item = curList[i]
            if (item.id == id) {
              curItem = JSON.parse(JSON.stringify(item)) // 克隆的效果
              break
            } else if (item.lists && item.lists.length) {
              getchild(item.lists)
            }
          }
        }
        getchild(lists)
        return curItem;
      }
    }
  }
</script>

<style lang="scss">
.drag-tree-table{
  margin: 20px 0;
  color: #606266;
  font-size: 12px;
}
.drag-tree-table-header{
  display: flex;
  padding: 15px 10px;
  background: #f5f7fa;
  height: 66px;
  line-height: 36px;
  box-sizing: border-box;
  font-weight: 600;
}
.tree-icon-hidden{
  visibility: hidden;
}
</style>
