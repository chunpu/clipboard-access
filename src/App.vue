<template>
  <div id="app">
    <div id="content">
    <div id="box">
          <div v-for="(clipboardData, i) in clipboardDataList" v-if="index === i" class="detail">
            <!-- clipboard 级别操作区域 -->
            <div>
              <!-- 只支持 flash -->
              <!-- <el-button :plain="true" type="success" @click="copyToClipboard(clipboardData)">Copy to Clipboard</el-button> -->
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
            <div v-for="(item, i) in clipboardData.items" v-if="i === clipboardData.itemIndex" @paste.stop @copy.stop>
              <el-tabs v-model="item.activeTab">
                <el-tab-pane label="Preview" name="Preview">
                  <div class="preview-box preview-box-border resize">
                    <div class="text-center" v-if="/image/i.test(item.type)" style="height: 100%"><img :src="item.data" style="max-width: 100%; max-height: 100%"></div>
                    <div v-else-if="item.type === 'text/html'" v-html="item.data"></div>
                    <div v-else v-text="item.data"></div>
                  </div>
                </el-tab-pane>

                <el-tab-pane label="Raw" name="Raw">
                  <div class="preview-box preview-box-border resize" v-if="item.activeTab == 'Raw'">
                    <div v-text="item.data"></div>
                  </div>
                </el-tab-pane>

                <el-tab-pane label="Edit" name="Edit">
                  <div class="preview-box resize full-textarea">
                    <el-input
                      v-if="item.activeTab == 'Edit'"
                      type="textarea"
                      resize="none"
                      class="full"
                      placeholder="Input Clipboard Content"
                      v-model="item.data">
                    </el-input>
                  </div>
                </el-tab-pane>

              </el-tabs>
            </div>

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

            <el-input
              class="pretty-textarea margin-vertical"
              type="textarea"
              :autosize="{minRows: 2, maxRows: 2}"
              v-model="hint"
              @paste.native.prevent
              >
            </el-input>

          </div>
    </div>
    </div>

    <!-- 历史必须是方形的 -->
    <div class="history">
      <div style="text-align: center" class="margin-vertical">
        <el-button @click="addClipboard()">Add Clipboard</el-button>
      </div>
      <div
        v-for="(clipboardData, i) in clipboardDataList"
        :class="{'history-item': true, 'margin-vertical': true, 'history-item-current': index === i}"
        @click="index = i"
        >
        <div
          v-for="(item, i) in clipboardData.items"
          v-if="clipboardData.items.length !== 0 && i === 0"
          style="height: 100%; margin: 1rem; overflow: hidden; text-overflow: ellipsis;">
          <div class="text-center" v-if="/image/i.test(item.type)" style="height: 100%"><img :src="item.data" style="max-width: 100%; max-height: 100%"></div>
          <div v-else-if="item.type === 'text/html'" v-html="item.data"></div>
          <div v-else v-text="item.data"></div>
        </div>
      </div>
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
        isEditMode: false, // 是否是编辑模式
        timestamp: _.now()
      }
    },
    createItem (type) {
      var item = {
        type: type,
        data: '',
        score: this.getScore(type),
        // showRaw: !this.isTypeCanPreview(type),
        activeTab: 'Preview'
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
      this.clipboardDataList.unshift(clipboardData)
      this.index = 0
      return clipboardData
    }
  }
}
</script>

<style>
body {
  /*background: #f5f5f5;*/
}
#box {
  max-width: 600px;
  margin: 15vh auto;
}
html {
  /*min-height: 101%;*/
}
.types {
  margin: 20px 0;
}
.text-center {
  text-align: center;
}
.resize {
  resize: both;
}
.preview-box {
  padding: 10px;
  word-wrap: break-word;
  overflow: auto;
  height: 10rem;
}
.preview-box-border {
  border: 1px solid #ddd;
  border-radius: 5px;
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
.history {
  position: absolute;
  top: 0;
  left: 0;
  height: 100vh;
  overflow-y: auto;
  box-sizing: border-box;
}
#content {
  margin-left: 200px;
}
.history-item {
  box-sizing: border-box;
  width: 150px;
  height: 150px;
  font-size: 12px;
  overflow: hidden;
  margin: 0rem 1.5rem 1.5rem 1.5rem;
  /*padding: 1rem;*/
  background: #fff;
  cursor: pointer;
  /*border-radius: 5px;*/
  box-shadow: 1px 2px 8px 1px rgba(0, 0, 0, 0.25);
  /*transition: all .3s;*/
}
.history-item:hover, .history-item-current {
  /*box-shadow: 1px 2px 8px 1px rgba(0, 0, 0, 0.25);*/
  outline: 2px solid #20a0ff;
}
.detail {
  background: #fff;
  padding: 10px;
}
.full {
  height: 100%;
  width: 100%;
}
.full-textarea textarea {
  height: 100%;
  width: 100%;
}

/* 窄屏幕隐藏右侧元素 */
@media(max-width: 620px){
  .history {
    display: none;
  }
  #content {
    margin-left: 0;
  }
}
</style>
