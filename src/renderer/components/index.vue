<template>
    <div class="index">
        <!--<div class="body-move"></div>-->

        <div class="slide-left">
            <div class="body-move"></div>
            <div class="bucket-tl">
                <i></i>
                <span>COS控制台</span>
            </div>

            <div class="bucket-opt">
                <ul>
                    <li><a @click="dialogAddVisible = true"><i class="el-new"></i> <span>新建</span> </a></li>
                    <li><a @click="dialogSettingVisible=true"><i class="el-setting"></i> <span>设置</span> </a></li>
                    <li><a @click="fetchData"><i class="fresh"></i> <span>刷新</span> </a></li>
                </ul>
            </div>

            <div class="bucket-group">
                <div class="bucket-item">
                    <div class="item" v-for="bkt in bucketList"
                         @click="selectBucket(bkt)"
                         @contextmenu="openMenu(bkt)"
                         :key="bkt.Name"
                         :class="{ 'active':bkt.active }"
                         :title="bkt.Name">
                        <a>
                            <i class="point"></i>
                            <span>{{bkt.Name}}</span>
                        </a>
                    </div>
                </div>
            </div>

            <div class="loading" v-if="bloading"><i class="el-icon-loading"></i></div>

            <div class="load-progress" @click="openProgressWindow">
                <i class="icon"></i>
                传输队列
                <scroll-bar v-if="run.upload || run.download"></scroll-bar>
            </div>
        </div>

        <div>
            <router-view @createBucket=" dialogAddVisible = true "></router-view>
        </div>

        <add-bucket :dialogAddVisible="dialogAddVisible" @closeBucket="dialogAddVisible = false"
                    @freshBucket="fetchData"></add-bucket>

        <!--属性管理-->
        <proto-manage :dialogManageVisible="dialogManageVisible" :currentBucket="rightChooseBucket"
                      @freshBucket="fetchData" @closeManage="dialogManageVisible=false"></proto-manage>
        <!--设置-->
        <setting :dialogSettingVisible="dialogSettingVisible" @closeDiolog="dialogSettingVisible = false"></setting>

        <!--跨域访问CORS设置-->
        <file-debris :isShow.sync="dialogCrosVisible" :options="rightChooseBucket"></file-debris>

        <!--权限设置-->
        <file-limit :isShow.sync="dialogFileLimit" :options="{type:'bucket', bucket:rightChooseBucket}"></file-limit>

    </div>
</template>

<script>
  import { mapState } from 'vuex'
  import setting from './modules/setting.vue'
  import addBucket from './modules/add-bucket.vue'
  import protoManage from './modules/property-manager.vue'
  import fileDebris from './modules/file-debris.vue'
  import fileLimit from './modules/file-limit.vue'
  import scrollBar from './modules/scroll-bar.vue'
  import { ipcRenderer, remote } from 'electron'

  const {Menu, MenuItem} = remote

  const bucketMenu = new Menu()

  export default {
    name: 'index-page',
    data () {
      return {
        bloading: false,
        rightChooseBucket: null,
        dialogAddVisible: false,
        dialogManageVisible: false,
        dialogSettingVisible: false,
        dialogCrosVisible: false,
        dialogFileLimit: false,
        run: {
          upload: false,
          download: false
        }
      }
    },

    components: {addBucket, protoManage, setting, fileDebris, fileLimit, scrollBar},

    computed: {
      ...mapState('bucket', ['bucketList']),
      ...mapState('menulist', ['options'])
    },

    watch: {
      options (nv, ov) {
        if (nv.Bucket !== ov.Bucket) {
          this.$store.commit('bucket/bucketActive', nv.Bucket)
        }
      }
    },

    created () {

      ipcRenderer.send('GetUploadTasks')
      ipcRenderer.send('GetDownloadTasks')

      ipcRenderer.on('GetUploadTasks-data', (event, tasks) => {
        this.run.upload = tasks.some(t => t.status === 'run')
      })

      ipcRenderer.on('GetDownloadTasks-data', (event, tasks) => {
        this.run.download = tasks.some(t => t.status === 'run')
      })

      bucketMenu.append(new MenuItem({label: '基本信息', click: this.getProperty}))
      bucketMenu.append(new MenuItem({label: '跨域访问CORS设置', click: this.setCors}))
      bucketMenu.append(new MenuItem({label: '权限管理', click: this.setLimit}))

      if(this.bucketList.length)
        this.selectBucket(this.bucketList[0])
    },

    methods: {
      openProgressWindow () {
        const url = (process.env.NODE_ENV === 'development'
          ? `http://localhost:9080/#/progress` : `file://${__dirname}/index.html#/progress`)

        let id = remote.getCurrentWindow().id

        let windows = remote.BrowserWindow.getAllWindows()
        if (windows.length > 1) {
          windows.forEach(n => {
            if (n.id !== id) {
              n.focus()
            }
          })
          return
        }

        let mainWindow = new remote.BrowserWindow({
          height: 450,
          maxHeight: 450,
          useContentSize: true,
          width: 794,
          maxWidth: 794,
          title: '传输队列',
          transparent: true,
          frame: false
        })
        mainWindow.loadURL(url)
      },

      fetchData () {
        this.bloading = true
        this.$store.dispatch('bucket/getService').then(() => {
          if (this.options.Bucket) this.$store.commit('bucket/bucketActive', this.options.Bucket)
          this.bloading = false
        })
      },

      selectBucket (b) {
        this.$router.push({
          path: '/file/' + b.Name,
          query: {
            Region: b.Location,
            Prefix: '',
            keyWord: ''
          }
        })
      },

      openMenu (item) {
        this.rightChooseBucket = {
          Bucket: item.Name,
          Region: item.Location,
          CreateDate: item.CreateDate
        }
        bucketMenu.popup(remote.getCurrentWindow(), {async: true})
      },

      getProperty () {
        this.dialogManageVisible = true
      },

      setCors () {
        this.dialogCrosVisible = true
      },

      setLimit () {
        this.dialogFileLimit = true
      }
    }
  }
</script>
