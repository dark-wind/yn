<template>
  <div class="repository-switch">
    <div class="current" v-if="current">仓库：{{current}}</div>
    <div class="current" v-else>未选择仓库</div>
    <ul class="list">
      <li v-for="(path, name) in repositories" :key="name" :title="path" @click="choose(name)">{{name}}</li>
    </ul>
  </div>
</template>

<script>
import file from '../file'

export default {
  name: 'repository-switch',
  components: {},
  props: {
  },
  data () {
    return {
      current: null,
      repositories: {}
    }
  },
  created () {
    this.choose(this.getRepo())
    this.fetchRepositories()
  },
  beforeDestroy () {
  },
  methods: {
    fetchRepositories () {
      file.fetchRepositories(data => {
        this.repositories = data
        const keys = Object.keys(data)

        if (keys.length > 0 && !keys.includes(this.current)) {
          this.choose(keys[0])
        }
      })
    },
    choose (name) {
      this.current = name
      this.storeRepo(this.current)
    },
    storeRepo (name) {
      window.localStorage['repository'] = name
      window.localStorage['repository_path'] = this.repositories[name]
    },
    getRepo () {
      return window.localStorage['repository'] || null
    }
  },
  watch: {
    current (val) {
      if (val) {
        this.$bus.emit('switch-repository', val)
        this.storeRepo(this.current)
      }
    },
    repositories () {
      this.storeRepo(this.current)
    }
  },
  computed: {
  }
}
</script>

<style scoped>
.repository-switch {
  width: 100px;
  cursor: pointer;
  user-select: none;
}

.current {
  padding: 0 .3em;
  white-space: nowrap;
  text-overflow: ellipsis;
  overflow: hidden;
}

.repository-switch:hover {
  background: #333030;
}

.repository-switch:hover .list {
  display: block;
}

.list {
  width: 100px;
  margin: 0;
  list-style: none;
  background: #333030;
  padding: 4px 0;
  display: none;
  position: fixed;
  bottom: 20px;
  box-sizing: border-box;
}

.list li {
  padding: 4px .6em;
  white-space: nowrap;
  text-overflow: ellipsis;
  overflow: hidden;
}

.list li:hover {
  background: #252525;
}
</style>
