<template>
  <el-row>
    <el-col :span="20">
      <!-- Previewer -->
      <Container
        group-name="1"
        @drop="onPreviewerDrop($event)"
        lock-axis="y"
        class="previewer-wrap"
        :should-animate-drop="shouldAnimateDrop"
      >
        <Draggable
          v-for="template in previewer.children"
          :key="template.id"
          :style="template.props.style"
        >
          <div>
            <span v-if="template.type !== 'column'">
              {{template.data}}
            </span>
          </div>
          <!-- Layout -->
          <Container
            orientation="horizontal"
            drag-handle-selector=".no-drag"
          >
            <Draggable
              v-for="(col, index) in template.children"
              :key="col.id"
              :style="col.props.style"
            >
              <!-- {{ col.data }}{{ index }} -->
              <!-- Elements -->
              <Container
                group-name="2"
                @drop-ready="onDropReady(col.id)"
                @drop="onElementsDrop(col.id, index, $event)"
                :get-child-payload="getElementPayload(col.id)"
                :should-animate-drop="shouldAnimateDrop"
                class="guide"
                >
                <Draggable
                  v-for="el in col.children"
                  :key="el.id"
                  >
                <div :style="el.props.style">
                  {{ el.data }}
                </div>
                </Draggable>
              </Container>
            </Draggable>
          </Container>
        </Draggable>
      </Container>
    </el-col>
    <el-col :span="4">
      <el-row>
        <Container
          class="draggable-items"
          behaviour="copy"
          group-name="2"
          :get-child-payload="getSourcePayload"
        >
          <Draggable
            v-for="item in items"
            :key="item.id"
          >
            <div class="draggable-item">
              {{item.data}}
            </div>
          </Draggable>
        </Container>
      </el-row>
      <el-row>
        <Container
          class="draggable-items"
          behaviour="copy"
          group-name="1"
          :get-child-payload="getSourceLayoutPayload"
        >
          <Draggable
            v-for="item in colItems"
            :key="item.id"
          >
            <div class="draggable-item">
              {{item.data}}
            </div>
          </Draggable>
        </Container>
      </el-row>
    </el-col>
  </el-row>
</template>

<script>
import { Container, Draggable } from "vue-smooth-dnd"
import { applyDrag, generateItems } from "@/utils/helpers"
import { v4 } from "uuid"
export default {
  components: { Container, Draggable },
  data() {
    return {
      // Source components
      items: [
        { type: "pic", data: "单图" },
        { type: "txt", data: "文本" },
        { type: "test", data: "test" }
      ],
      // Source Layout
      colItems: [
        { type: "column", colNum: 1, data: "one column" },
        { type: "column", colNum: 2, data: "two columns" },
        { type: "column", colNum: 3, data: "three columns" },
        { type: "column", colNum: 4, data: "four columns" }
      ],
      // previewer
      previewer: {
        type: "container",
        children: []
      },
      styles: {
        previewerStyle: {
          display: "flex",
          justifyContent: "center",
          alignItems: "center",
          width: "800px",
          height: "200px"
        },
        layoutStyle: {}
      }
    }
  },
  mounted () {
    let params = [
      {name: 'previewer', size: 1, lineStyle: 'dashed', color: '#842'},
      {name: 'layout', size: 1, lineStyle: 'dashed', color: '#995'}
    ]
    this.initBorderStyle(params)
    // 初始化
    const init = this.getSourceLayoutPayload(0)
    this.previewer.children.push(init)
  },
  methods: {
    // init
    initBorderStyle (arr) {
      arr.map(v => {
        const styles = this.styles[`${v.name}Style`]
        const border = `${v.size}px ${v.lineStyle} ${v.color}`
        let newStyle = {
          borderTop: border,
          borderBottom: border,
          borderLeft: border,
          borderRight: border,
          boxSizing: 'border-box',
          ...styles
        }
        this.styles[`${v.name}Style`] = newStyle
      })
    },

    // 拖拽源组件
    getSourcePayload (index) {
      let uuid = v4().replace(/-/g, "") // 随机生成 id
      let item = _.cloneDeep(this.items[index])
      item.id = uuid
      item.props = {
        style: {
          // width: '200px',
          // height: '150px',
          // backgroundColor: '#ccc'
        }
      }
      return item
    },

    // 拖拽源布局组件
    getSourceLayoutPayload (index) {
      let uuid = v4().replace(/-/g, "") // 随机生成 id
      let item = _.cloneDeep(this.colItems[index])
      let arr = []
      const previewerWith = this.styles.previewerStyle.width
      const previewerHeight = this.styles.previewerStyle.height
      const onlyNumReg = /[^\d.-]/g

      let layoutWidth =
        (previewerWith.replace(onlyNumReg, "") - 20) / item.colNum
      let layoutHeight = previewerHeight.replace(onlyNumReg, "") - 20
      for (let i = 0, l = item.colNum; i < l; i++) {
        let uid = v4().replace(/-/g, "") // 随机生成 id
        let obj = {
          id: uid,
          props: {
            style: {
              width: `${layoutWidth}px`,
              height: `${layoutHeight}px`,
              ...this.styles.layoutStyle
            }
          },
          data: "hello world "
        }
        arr.push(obj)
      }

      item = {
        id: uuid,
        props: {
          style: this.styles.previewerStyle
        },
        children: arr,
        ...item
      }

      return item
    },

    // 拖拽已在视图上生成的元素
    getElementPayload (columnId) {
      return index => {
        let res = {}
        this.previewer.children.map(v => {
          let obj = v.children.filter(x => x.id === columnId)[0]
          obj !== undefined && (res = obj.children[index])
        })
        return { ...res }
      }
    },

    // 在画布上放置布局组件
    onPreviewerDrop (dropResult) {
      const previewer = Object.assign({}, this.previewer)
      previewer.children = applyDrag(previewer.children, dropResult)
      this.previewer = previewer
    },

    onDropReady (columnId) {
    },

    // 在布局内放置元素组件
    onElementsDrop (columnId, columnIndex, dropResult) {
      const { addedIndex, removedIndex, payload } = dropResult
      let result = _.cloneDeep(dropResult)
      if (Array.isArray(payload)) {
        result.payload = { ...payload[0] }
      }
      if (addedIndex !== null || removedIndex !== null) {
        const previewer = Object.assign({}, this.previewer)
        let column = {}
        let preivewerIndex = 0

        /* 此处为暴力枚举，需设法优化 */
        previewer.children.map((v, i) => {
          let res = v.children.filter(x => x.id === columnId)[0]
          if (res !== undefined) {
            column = res
            preivewerIndex = i
          }
        })
        const newColumn = Object.assign({}, column)
        // 初始化 children
        newColumn.children === undefined && (newColumn.children = [])
        newColumn.children = applyDrag(newColumn.children, result)

        previewer.children[preivewerIndex].children.splice(
          columnIndex,
          1,
          newColumn
        )
        this.previewer = previewer
      }
    },

    // 控制 drop 时的动画效果
    shouldAnimateDrop (sourceContainerOptions, payload) {
      return false
    }
  }
}
</script>
<style scoped  lang="less">
& /deep/ .el-col {
  border: 1px solid #000;
  box-sizing: border-box;
  min-height: calc(100vh - 82px);
  .el-row {
    border: 1px dashed #f06;
    box-sizing: border-box;
  }
}
.previewer-wrap {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  .guide {
    &::before {
      content: '';
      position: absolute;
      left: 0;
      width: 100%;
      border-top: 3px solid #990;
      box-sizing: border-box;
      visibility: hidden;
    }
    &:hover {
      &::before {
        visibility: visible;
      }
    }
  }
}
</style>
