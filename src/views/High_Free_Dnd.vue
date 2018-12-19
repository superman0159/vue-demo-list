<template>
  <el-row>
    <el-col :span="20">
      <!-- Previewer -->
      <Container
        group-name="1"
        @drop="onPreviewerDrop($event)"
        lock-axis="y"
        class="previewer-wrap"
        >
        <Draggable
          v-for="template in previewer.children"
          :key="template.id"
          :style="template.props.style"
          >
          <div>
            <span v-show="template.type !== 'column'">
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
              {{ col.data }}{{ index }}
              <!-- Elements -->
              <Container
                group-name="2"
                @drop="onElementsDrop(col.id, index, $event)"
                :get-child-payload="getElementPayload(col.id)"
                >
                <Draggable
                  v-for="el in col.children"
                  :key="el.id"
                  >
                  {{ el.data }}
                </Draggable>
              </Container>
            </Draggable>
          </Container>
        </Draggable>
      </Container>
    </el-col>
    <el-col :span="4">
      <el-row>
        <Container class="draggable-items" behaviour="copy" group-name="2" :get-child-payload="getSourcePayload">
        <Draggable v-for="item in items" :key="item.id">
          <div class="draggable-item">
            {{item.data}}
          </div>
        </Draggable>
        </Container>
      </el-row>
      <el-row>
        <Container class="draggable-items" behaviour="copy" group-name="1" :get-child-payload="getSourceLayoutPayload">
        <Draggable v-for="item in colItems" :key="item.id">
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
import { v4 } from 'uuid'
export default {
  components: {Container, Draggable},
  data() {
    return {
      // Source components
      items: [
        { type: 'pic', data: '单图'},
        { type: 'txt', data: '文本' },
        { type: 'test', data: 'test' }
      ],
      // Source Layout
      colItems: [
        { type: 'column', colNum: 1, data: 'one column' },
        { type: 'column', colNum: 2, data: 'two columns' },
        { type: 'column', colNum: 3, data: 'three columns' },
        { type: 'column', colNum: 4, data: 'four columns' }
      ],
      // previewer
      previewer: {
        type: 'container',
        children: []
      },
      styles: {
        previewerStyle: {
          display: 'flex',
          justifyContent: 'center',
          alignItems: 'center',
          width: '800px',
          height: '200px',
          border: '1px dotted #059',
          boxSizing: 'border-box'
        },
        layoutStyle: {
          border: '1px dashed #066',
          boxSizing: 'border-box'
        }
      }
    }
  },
  mounted () {
    // 初始化
    const init = this.getSourceLayoutPayload(0)
    this.previewer.children.push(init)
  },
  methods: {

    // 拖拽源组件
    getSourcePayload (index) {
      let uuid = v4().replace(/-/g, '') // 随机生成 id
      let item = _.cloneDeep(this.items[index]) 
      item.id = uuid
      item.props = {
        style: this.styles.previewerStyle
      }
      // console.log('source', item)
      return item
    },

    // 拖拽源布局组件
    getSourceLayoutPayload (index) {
      let uuid = v4().replace(/-/g, '') // 随机生成 id
      let item = _.cloneDeep(this.colItems[index]) 
      let arr = []
      const previewerWith = this.styles.previewerStyle.width
      const previewerHeight = this.styles.previewerStyle.height
      const onlyNumReg = /[^\d.-]/g

      let layoutWidth = (previewerWith.replace(onlyNumReg, '') - 20) / item.colNum
      let layoutHeight = previewerHeight.replace(onlyNumReg, '') - 20
      for (let i = 0; i < item.colNum; i++) {
        let uid = v4().replace(/-/g, '') // 随机生成 id
        let obj = {
          id: uid,
          props: {
            style: {
              width: `${layoutWidth}px`,
              height: `${layoutHeight}px`,
              ...this.styles.layoutStyle
            }
          },
          data: 'hello world '
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
      // console.log(item)
      
      return item
    },

    onPreviewerDrop (dropResult) {
      const previewer = Object.assign({}, this.previewer)
      previewer.children = applyDrag(previewer.children, dropResult)
      this.previewer = previewer
    },

    onElementsDrop (columnId, columnIndex, dropResult) {
      const { addedIndex, removedIndex, payload } = dropResult
      let result = _.cloneDeep(dropResult)
      if (Array.isArray(payload)) {
        result.payload = {...payload[0]}
      }
      // console.log('paylaod', result.payload)
      if (addedIndex !== null || removedIndex !== null) {
        const previewer = Object.assign({}, this.previewer)
        let column = {}
        let preivewerIndex = 0

        /* 此处为暴力枚举，需设法优化 */
        previewer.children.map ((v, i) => {
          let res = v.children.filter(x => 
            x.id === columnId
          )[0]
          if (res !== undefined) {
            column = res
            preivewerIndex = i
          }
        })
        const newColumn = Object.assign({}, column)
        // 初始化 children
        newColumn.children === undefined && (newColumn.children = [])
        newColumn.children = applyDrag(newColumn.children, result)

        previewer.children[preivewerIndex].children.splice(columnIndex, 1, newColumn)
        this.previewer = previewer
      }
    },

    getElementPayload (columnId) {
      return index => {
        let res = {}
        this.previewer.children.map (v => {
          let obj = v.children.filter (x => 
            x.id === columnId
          )[0]
          obj !== undefined && (res = obj.children[index])
        })
        return {...res}
      }
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
}
</style>