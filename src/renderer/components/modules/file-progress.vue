<template>
    <div class="list-progress">
        <div class="el-side-tab">
            <div class="el-side-head clearfix">
                <ul>
                    <li :class="{'active' : tabList[0].iscur}" @click="tabFn(0)">
                        {{ tabList[0].name }} {{ uploadCount }}
                    </li>
                    <li :class="{'active' : tabList[1].iscur}" @click="tabFn(1)">
                        {{ tabList[1].name }} <span> {{ downloadCount }}</span>
                    </li>
                    <li :class="{'active' : tabList[2].iscur}" @click="tabFn(2)">
                        {{ tabList[2].name }}
                    </li>
                </ul>


            </div>

            <div class="el-side-content">
                <div class="first" v-show=" tabList[0].iscur ">
                    <progress-list :list="uploadList" :type="'upload'"
                                   @speed="val => uploadSpeed = val"></progress-list>
                </div>
                <div class="second" v-show=" tabList[1].iscur ">
                    <progress-list :list="downloadList" :type="'download'"
                                   @speed="val => downloadSpeed = val"></progress-list>
                </div>
                <div class="third" v-show=" tabList[2].iscur ">
                    <div class="title-bar">
                        <div v-if="platform === 'other'" class="full-win-btn">
                            <span @click="windCtrl('mini')" class="btn-p1"><i></i></span>
                            <span @click="windCtrl('maxi')" class="btn-p2"><i></i></span>
                            <span @click="windCtrl('close')" class="btn-p3"><i></i></span>
                        </div>
                        <div class="el-button-group">
                            <el-button size="small" @click="openLog" :plain="true">打开日志列表</el-button>
                        </div>
                    </div>
                    <div class="log" ref="log"></div>
                </div>
            </div>
            <div class="el-speed">
                <p> 上传速度: {{uploadSpeed | bitSpeed}} </p>
                <p> 下载速度: {{downloadSpeed | bitSpeed}} </p>
            </div>
        </div>
    </div>
</template>

<script>
  import progressList from './progress-list.vue'
  import { ipcRenderer, shell ,remote } from 'electron'

  export default {
    name: 'fileProgress-page',
    data () {
      return {
        tabList: [
          {iscur: true, name: '上传队列', id: 0},
          {iscur: false, name: '下载队列', id: 1},
          {iscur: false, name: '错误日志', id: 2}
        ],
        uploadList: [],
        downloadList: [],
        uploadSpeed: 0,
        downloadSpeed: 0
      }
    },
    created () {
      ipcRenderer.send('GetUploadTasks')
      ipcRenderer.on('GetUploadTasks-data', (event, data) => {
        // console.log('GetUploadTasks-data', data.length)
        this.uploadList = data
      })

      ipcRenderer.send('GetDownloadTasks')
      ipcRenderer.on('GetDownloadTasks-data', (event, data) => {
        // console.log('GetDownloadTasks-data', data.length)
        this.downloadList = data
      })
      if (process.platform === 'darwin') {
        this.$store.commit('setPlatform', 'darwin')
      }
      let elem = document.getElementById('fullbtn')
      if (elem && this.platform === 'other') elem.remove()
    },
    mounted () {
      ipcRenderer.on('__ELECTRON_LOG_RENDERER__', (event, level, text) => {
        let el = this.$refs.log
        let node = document.createTextNode(text)
        let dom = document.createElement('div')
        dom.appendChild(node)
        dom.setAttribute('class', level)
        el.appendChild(dom)
        console.log('log', dom, el)
        if (el.childElementCount > 1000) {
          el.removeChild(el.firstChild)
        }
      })
    },
    components: {progressList},
    computed: {
      platform(){
        return this.$store.state.platform
      },
      config () {
        return this.$store.state.config
      },
      uploadCount () {
        if (!this.uploadList.length) return ''
        let c = 0
        for (let item of this.uploadList) {
          if (item.status === 'complete') c++
        }
        return `(${c}/${this.uploadList.length})`
      },
      downloadCount () {
        if (!this.downloadList.length) return ''
        let c = 0
        for (let item of this.downloadList) {
          if (item.status === 'complete') c++
        }
        return `(${c}/${this.downloadList.length})`
      }
    },
    methods: {
      tabFn (index) {
        this.tabList.forEach(tab => { tab.iscur = false })
        this.tabList[index].iscur = true
      },
      openLog () {
        shell.showItemInFolder(ipcRenderer.sendSync('GetLogPath'))
      },
      windCtrl(type){
        let win = remote.getCurrentWindow()
        switch (type) {
          case 'close':
            win.close()
            break
          case 'mini':
            win.minimize()
            break
          case 'maxi':
            if (win.isMaximized())
              win.unmaximize()
            else
              win.maximize()
            break
        }
      }
    }
  }
</script>