<template>
  <div id="app">
    <div id="box">
      <el-tabs>
        <el-tab-pane label="Tab" v-for="clipboardData in clipboardDataList">
          <div>

            <div class="types">
              <el-radio-group v-model="clipboardData.itemIndex">
                <el-radio-button v-for="(item, i) in clipboardData.items" :label="i">{{item.type}}</el-radio-button>
              </el-radio-group>
            </div>

            <div v-for="(item, i) in clipboardData.items" v-if="i === clipboardData.itemIndex">
<!--               <el-input
                type="textarea"
                :autosize="{minRows: 6, maxRows: 6}"
                placeholder="Ctrl/Command + v"
                v-model="item.data">
              </el-input> -->
              <div class="preview">
                <div v-if="item.showRaw" v-text="item.data" class="height-limit"></div>
                <div class="text-center" v-else-if="/image/i.test(item.type)"><img :src="item.data" style="max-width: 100%"></div>
                <div v-else-if="item.type === 'text/html'" v-html="item.data" class="height-limit"></div>
              </div>
              <el-checkbox v-if="isTypeCanPreview(item.type)" style="margin: 15px 0;" v-model="item.showRaw">Raw</el-checkbox>
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
      clipboardDataList: [this.createClipboardData()],
      index: 0
    }
  },
  watch: {
  },
  mounted () {
    window.app = this
    document.addEventListener('paste', this.onPaste)
  },
  methods: {
    createClipboardData () {
      return {
        items: [],
        itemIndex: 0
      }
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
        clipboardData.items = items
        clipboardData.itemIndex = 0
      }).then(() => {
        this.$message({
          message: 'Paste Success',
          type: 'success'
        })
      })
    },
    readItem (item) {
      return new Promise((resolve, reject) => {
        var ret = {
          kind: item.kind,
          type: item.type,
          showRaw: !this.isTypeCanPreview(item.type)
        }
        ret.score = this.getScore(item.type)
        if (item.kind === 'file') {
          var file = item.getAsFile()
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
.preview {
  border: 1px solid #ddd;
  border-radius: 5px;
  resize: both;
  padding: 10px;
  word-wrap: break-word;
  overflow: auto;
}
.preview .height-limit {
  max-height: 350px;
}
.el-tabs__content {
  overflow: visible;
}
</style>
