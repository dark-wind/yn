<template>
  <aside class="side" @dblclick="init()" title="双击此处刷新目录树">
    <div class="loading" v-if="tree === null"> 加载中 </div>
    <template v-else>
      <TreeNode
        v-for="item in tree"
        :slected-file="file"
        :item="item"
        :key="item.path"
        @move="onMove"
        @change="change"
        @select="handleSelect"
        @delete="onDelete" />
        <transition name="fade">
          <div v-if="showFilter" class="filter-wrapper" @click="showFilter = false">
            <XFilter @choose-file="showFilter" :repo="repo" :files="files" />
          </div>
        </transition>
    </template>
  </aside>
</template>

<script>
import File from '../file'
import TreeNode from './TreeNode'
import XFilter from './Filter'

export default {
  name: 'tree',
  components: { TreeNode, XFilter },
  data () {
    return {
      repo: null,
      tree: null,
      file: null,
      showFilter: false,
      editorReady: false
    }
  },
  created () {
    window.addEventListener('keydown', this.keydownHandler, true)
    this.$bus.on('switch-repository', this.init)
    this.$bus.on('editor-ready', this.handleReady)
    this.$bus.on('tree-refresh', this.change)
  },
  beforeDestroy () {
    window.removeEventListener('keydown', this.keydownHandler)
    this.$bus.off('switch-repository', this.init)
    this.$bus.off('editor-ready', this.handleReady)
    this.$bus.off('tree-refresh', this.change)
  },
  methods: {
    closeCurrentFile () {
      this.file = null
    },
    handleSelect (file) {
      if (this.editorReady) {
        this.file = file
        this.updateRecentOpenTime(this.file)
      }
    },
    handleReady () {
      this.editorReady = true

      // 第一次打开此程序
      if (!Object.keys(window.localStorage).find(x => x.endsWith('_open_time'))) {
        this.$bus.$emit('toggle-readme')
      }
    },
    init (repo = null, path = null) {
      if (repo) {
        this.repo = repo
        this.tree = null
        this.file = null
      }

      File.tree(this.repo, tree => {
        this.tree = tree

        if (path) {
          this.$nextTick(() => {
            this.$bus.emit('choose-file', { path })
          })
        }
      })
    },
    onDelete (path) {
      // 删除了正在编辑的文件或者其父目录
      if (this.file && this.file.path.startsWith(path)) {
        this.file = null
      }

      this.init()
    },
    onMove ({ oldPath, newPath }) {
      // 移动了正在编辑的文件或者其父目录
      if (this.file && this.file.path.startsWith(oldPath)) {
        this.file = null
        this.init(null, newPath)
      } else {
        this.init()
      }
    },
    change (path) {
      this.init(null, path)
    },
    keydownHandler (e) {
      if (e.key === 'i' && e.ctrlKey && e.altKey) {
        this.showFilter = f => {
          if (this.file) {
            const relativePath = f.path.replace(this.file.path.substr(0, this.file.path.lastIndexOf('/')), '.')
            this.$bus.emit('editor-insert-value', `[${f.name.replace(/\.[^.]$/, '')}](${encodeURI(relativePath)})`)
          }
          this.showFilter = false
        }
        e.preventDefault()
        e.stopPropagation()
      } else if (e.key === 'p' && e.ctrlKey) {
        this.showFilter = f => {
          this.$bus.emit('choose-file', f)
          this.showFilter = false
        }
        e.preventDefault()
        e.stopPropagation()
      } else if (e.key === 'Escape' && this.showFilter) {
        this.showFilter = false
        e.preventDefault()
        e.stopPropagation()
      }
    },
    travelFiles (tree) {
      let tmp = []

      tree.forEach(node => {
        if (node.type === 'file' && node.path.endsWith('.md')) {
          tmp.push(node)
        }

        if (Array.isArray(node.children)) {
          tmp = tmp.concat(this.travelFiles(node.children))
        }
      })

      return tmp
    },
    updateRecentOpenTime (file) {
      const json = window.localStorage[`${file.repo}_open_time`] || '{}'

      let map = {}
      try {
        map = JSON.parse(json)
      } catch (error) {
      }

      map[file.path] = new Date().getTime()

      window.localStorage[`${file.repo}_open_time`] = JSON.stringify(map)
    }
  },
  watch: {
    file (f) {
      this.$emit('input', f)
    }
  },
  computed: {
    files () {
      return this.travelFiles(this.tree)
    }
  }
}
</script>

<style scoped>
.side {
  color: #ddd;
  background: #282a2b;
}

.filter-wrapper {
  position: fixed;
  top: 0;
  width: 100%;
  height: 100%;
  background: rgba(255, 255, 255, .2);
  z-index: 99999;
  padding-top: 4em;
}

.fade-enter-active, .fade-leave-active {
  transition: opacity .5s;
}

.fade-enter, .fade-leave-to {
  opacity: 0;
}

.loading {
  font-size: 32px;
  text-align: center;
  padding-top: 50%;
  color: #848181;
}
</style>
