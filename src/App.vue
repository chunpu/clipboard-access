<template>
  <div id="app">
    <div id="box">
      <el-tabs>
        <el-tab-pane label="Show Clipboard">

          <el-input
            type="textarea"
            :autosize="{minRows: 6, maxRows: 6}"
            autofocus
            @paste.native.prevent="onPaste"
            placeholder="Ctrl/Command + v"
            v-model="paste.currentData">
          </el-input>

          <div class="types">
            <el-radio v-for="item in paste.types" v-model="paste.currentType" :label="item"></el-radio>
          </div>

          <div id="preview">
            <!-- 预览 -->
            <div v-if="paste.currentType == 'text/html'" v-html="paste.currentData"></div>
            <div class="text-center" v-if="/image/i.test(paste.currentType)"><img :src="paste.currentData"></div>
          </div>

        </el-tab-pane>
        <el-tab-pane label="Set Clipboard">Coming Soon!</el-tab-pane>
      </el-tabs>
    </div>
    <!-- <router-view></router-view> -->
  </div>
</template>

<script>

import * as _ from 'lodash'

// const share = {}

export default {
  name: 'app',
  data () {
    return {
      textarea: '',
      paste: {
        types: [],
        currentData: null,
        currentType: null,
        items: []
      }
    }
  },
  watch: {
    'paste.currentType' () {
      var currentItem = this.paste.items.find(item => {
        // 因为是粘贴, 所以不可能重复, drag 会
        return item.type === this.paste.currentType
      })
      if (currentItem) {
        this.paste.currentData = currentItem.data
      }
    }
  },
  mounted () {
    window.app = this
  },
  methods: {
    onPaste (e) {
      var items = _.map(e.clipboardData.items, item => {
        var ret = {
          kind: item.kind,
          type: item.type
        }
        if (item.kind === 'file') {
          var file = item.getAsFile()
          var url = URL.createObjectURL(file)
          ret.data = url
        } else {
          ret.data = e.clipboardData.getData(item.type)
        }
        ret.score = this.getScore(item.type)
        return ret
      })

      // 自动优先富文本
      items.sort((a, b) => {
        return b.score - a.score
      })

      console.log(items)
      this.paste.types = _.map(items, item => item.type)
      this.paste.currentType = this.paste.types[0]
      this.paste.items = items
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
</style>
