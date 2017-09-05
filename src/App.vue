<template>
  <div id="app">
    <div id="box">
      <el-button @click="addClipboard()" class="margin-vertical">Add Clipboard</el-button>
      <!-- TODO 改成列表, 可预览 -->
      <el-tabs v-model="index">
        <el-tab-pane label="Tab" v-for="clipboardData in clipboardDataList">
          <div>
            <!-- clipboard 级别操作区域 -->
            <div>
              <el-button type="text" :disabled="clipboardData.isEditMode" @click="clipboardData.isEditMode = true">Edit Clipboard</el-button>
              <!-- 只支持 flash -->
              <!-- <el-button :plain="true" type="success" @click="copyToClipboard(clipboardData)">Copy to Clipboard</el-button> -->
            </div>

            <el-input
              class="pretty-textarea margin-vertical"
              type="textarea"
              :autosize="{minRows: 2, maxRows: 2}"
              v-model="hint"

              @paste.native.prevent
              >
            </el-input>
            <div class="types" v-if="clipboardData.items.length !== 0">
              <el-tag
                :key="i"
                v-for="(item, i) in clipboardData.items"
                :type="clipboardData.itemIndex == i ? 'primary' : 'gray'"
                :closable="clipboardData.isEditMode"
                :close-transition="true"
                @click.native="clipboardData.itemIndex = i"
                @close="clipboardData.items.splice(i, 1)"
              >{{item.type}}
              </el-tag>
              <el-input
                v-if="clipboardData.isEditMode"
                placeholder="Input Type"
                icon="plus"
                v-model="clipboardData.newItemType"
                size="small"
                class="new-item-type"
                :on-icon-click="handleAddItem"
                @keyup.enter.native="handleAddItem"
                >
              </el-input>
            </div>

<!--             <div v-if="clipboardData.items.length === 0">
              <el-alert
                class="margin-vertical"
                title="Press Ctrl / Command + V"
                type="info"
                :closable="false"
                show-icon>
              </el-alert>
            </div> -->
            <div v-for="(item, i) in clipboardData.items" v-if="i === clipboardData.itemIndex">
              <div v-if="clipboardData.isEditMode">
                <el-input
                  type="textarea"
                  :autosize="{minRows: 3, maxRows: 12}"
                  placeholder="Input Clipboard Content"
                  v-model="item.data">
                </el-input>
              </div>
              <div class="preview" v-else>
                <div class="preview-box">
                  <div v-if="!isTypeCanPreview(item.type)" v-text="item.data" class="height-limit"></div>
                  <div class="text-center" v-else-if="/image/i.test(item.type)"><img :src="item.data" style="max-width: 100%"></div>
                  <div v-else-if="item.type === 'text/html'" v-html="item.data" class="height-limit"></div>

                </div>

                <el-checkbox v-if="isTypeCanPreview(item.type)" class="margin-vertical" v-model="item.showRaw">Raw</el-checkbox>

                <div class="preview-box" v-if="item.showRaw && isTypeCanPreview(item.type)">
                  <div v-text="item.data" class="height-limit"></div>
                </div>
              </div>
            </div>
          </div>
        </el-tab-pane>
      </el-tabs>
    </div>
    <!-- <router-view></router-view> -->
  </div>
</template>

<script>

import * as _ from 'lodash'

// 相比其他剪贴板工具的特色是, 可以手动生成剪贴板, 可以查看 html 具体内容
// 数据结构是这样的
// clipboardDataList => 可以保存一连串的 clipboardData, 这个可以后面做, 先始终只改第一个
// 可以自己新建 clipboardData
// clipboardData.itemList 包含各种格式
// item => {type(mime), data}
// clipboardData 可以被修改, 然后继续复制
// 一开始, clipboardData 是空的, 然后被 Ctrl + V 填充
// 可以新建 clipboardData, 然后自己设置
// clipboardData 中可以新建 item

export default {
  name: 'app',
  data () {
    return {
      hint: 'Press Ctrl / Command + V to Paste\nPress Ctrl / Command + C to Copy',
      clipboardDataList: [],
      index: null
    }
  },
  watch: {
  },
  mounted () {
    window.app = this
    this.addClipboard()
    document.addEventListener('paste', this.onPaste)
    document.addEventListener('copy', this.onCopy)
  },
  methods: {
    createClipboardData () {
      return {
        items: [], // item list
        itemIndex: 0, // 当前显示的 item
        name: this.clipboardDataList.length + '', // 名称, 用来显示 tab 的, 需要是字符串
        newItemType: '', // 新建的 item 类型
        textarea: '', // 鼠标操作区只能是 textarea
        isEditMode: false // 是否是编辑模式
      }
    },
    createItem (type) {
      var item = {
        type: type,
        data: '',
        score: this.getScore(type),
        showRaw: !this.isTypeCanPreview(type)
      }
      return item
    },
    handleAddItem () {
      var clipboardData = this.clipboardDataList[this.index]
      if (clipboardData) {
        clipboardData.items.push(this.createItem(clipboardData.newItemType))
      }
      clipboardData.itemIndex = clipboardData.items.length - 1
      clipboardData.newItemType = ''
    },
    isTypeCanPreview(type) {
      // 不是图片也不是 html
      if (/image/i.test(type) || /html/i.test(type)) {
        return true
      }
      return false
    },
    copyToClipboard (clipboardData) {
      var promises = _.map(clipboardData.items, item => {
        return new Promise()
      })
      return Promise.all()
    },
    onPaste (e) {
      var items = _.map(e.clipboardData.items, item => this.readItem(item))
      return Promise.all(items).then(items => {
        // 自动优先富文本
        items.sort((a, b) => {
          return b.score - a.score
        })
        console.log('paste', items)
        var clipboardData = this.clipboardDataList[this.index]
        if (clipboardData.items.length !== 0) {
          clipboardData = this.addClipboard()
        }
        clipboardData.items = items
        clipboardData.itemIndex = 0
      }).then(() => {
        this.$message({
          message: 'Paste Success',
          type: 'success'
        })
      }).catch(e => {
        this.$message({
          message: 'Paste Fail',
          type: 'error'
        })
      })
    },
    onCopy (e) {
      var dataTransfer = e.clipboardData
      dataTransfer.clearData()
      var clipboardData = this.clipboardDataList[this.index]
      if (clipboardData) {
        _.each(clipboardData.items, item => {
          dataTransfer.setData(item.type, item.data)
        })
        this.$message({
          message: 'Copy Success',
          type: 'success'
        })
      }
      e.preventDefault() // 默认是复制选中区域, 但不能 return false
    },
    readItem (item) {
      return new Promise((resolve, reject) => {
        var ret = this.createItem(item.type)
        ret.kind = item.kind
        if (item.kind === 'file') {
          var file = item.getAsFile()
          if (!file) {
            return reject(new Error('read item fail'))
          }
          var reader = new FileReader()
          _.extend(ret, {
            name: file.name,
            lastModified: file.lastModified,
            size: file.size
          })
          reader.onload = function () {
            var base64 = reader.result // base64
            ret.data = base64
            resolve(ret)
          }
          reader.onerror = _.noop
          reader.readAsDataURL(file)
        } else {
          item.getAsString(str => {
            ret.data = str
            resolve(ret)
          })
        }
      })
    },
    getScore (type) {
      if (/image/i.test(type)) {
        return 100
      } else if (/html/i.test(type)) {
        return 50
      }
      return 0
    },
    addClipboard () {
      var clipboardData = this.createClipboardData()
      this.clipboardDataList.push(clipboardData)
      this.index = (this.clipboardDataList.length - 1) + ''
      return clipboardData
    }
  }
}
</script>

<style>
#box {
    width: 600px;
    margin: 100px auto;
}
html {
    min-height: 101%;
}
.types {
  margin: 20px 0;
}
.text-center {
  text-align: center;
}
.preview-box {
  border: 1px solid #ddd;
  border-radius: 5px;
  resize: both;
  padding: 10px;
  word-wrap: break-word;
  overflow: auto;
}
.preview-box .height-limit {
  max-height: 350px;
}
.el-tabs__content {
  overflow: visible;
}
.pretty-textarea textarea {
  border: 2px dashed #ddd;
  color: #ddd;
}
.margin-vertical {
  margin-top: 15px;
  margin-bottom: 15px;
}
.el-tag+.el-tag {
    margin-left: 10px;
}
.el-tag {
  cursor: pointer;
}
.new-item-type {
  margin: 0 10px;
  width: 115px;
  height: 24px;
  line-height: 22px;
  padding-top: 0;
  padding-bottom: 0;
}
.new-item-type input {
  height: 100% !important;
}
</style>
