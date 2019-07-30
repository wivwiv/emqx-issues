<template>
  <div class="main">
    <div class="header">
      <h3>
        issues 面板
        <span class="item-label time-label">更新时间降序</span>
      </h3>
      <button @click="loadData">
        {{ loading ? '加载中' : '刷新' }}
      </button>
      <span class="item-label time-label">{{ reloadDate }}</span>
      <span class="item-label time-label dark">
        自动刷新 {{ exp }}

        <span class="item-label">
          <input v-model="inv" name="d" type="radio" :value="60" /> 1 分钟
          <input v-model="inv" name="d" type="radio" :value="180" /> 3 分钟
        </span>

        <button @click="checkNotice" style="float:right">
          启用通知
        </button>
      </span>
    </div>
    <ul>
      <li v-for="item in list" :key="item.id" class="item-list" :class="{ is_new: item.not_reply }">
        <div class="user-info">
          <img class="user-avatar" :src="item.user.avatar_url" :alt="item.user.login">
          <span> {{ item.user.login }} </span>
        </div>

        <div class="item" :title="item.body || '暂无'">
          <a :href="item.html_url" target="_blank">
            {{ item.title }}
          </a>

          <div class="item-footer">
            <span v-if="item.is_new" class="item-label is-new">
              新增
            </span>
            <span v-if="item.not_reply" class="item-label is-new">
              未回复
            </span>
            <span class="item-label">{{ item.created_at | date }}</span>
            <span class="btn item-label">详情</span>
            <a :href="item.html_url" target="_blank" class="btn item-label">
              查看
            </a>
            <span v-if="item.not_reply" class="btn item-label" @click="delApply(item)">
              忽略
            </span>
          </div>
        </div>

      </li>
    </ul>
  </div>
</template>

<script>
const ISSUES_API = 'https://api.github.com/repos/emqx/emqx/issues?sort=updated&';

export default {
  name: 'Main',
  data() {
    return {
      loading: false,
      reloadDate: '',
      list: [],
      inv: 60,
      timer: 0,
      exp: 60,
      expTimer: 0,
    }
  },
  watch: {
    inv(val) {
      clearInterval(this.timer)
      this.timer = setInterval(this.loadData, val * 1000)
    }
  },
  methods: {
    __exp() {
      clearInterval(this.expTimer)
      this.expTimer = clearInterval(() => {
        this.exp -= 1
      }, 1000)
    },
    checkNotice() {
      this.notice({})
    },
    notice({ title = '通知已启用', body = '浏览器的通知已启用成功' }) {
      Notification.requestPermission(function (status) {
        var n = new Notification(title, { body }); // 显示通知
      });
    },
    setS(obj) {
      localStorage.setItem('s', JSON.stringify(obj))
    },
    getS() {
      return JSON.parse(
        localStorage.getItem('s') || '{}'
      )
    },
    async getData() {
      let list = []
      for (let i = 1; i <= 3; i++) {
        const resp = await fetch(ISSUES_API + `page=${i}`)
        let data = await resp.json()
        list = list.concat(data)
      }
      return list
    },
    delApply(item) {
      const map = this.getS()
      map[item.id] = map[item.id] || {}
      map[item.id].delApply = true
      this.setS(map)
      item.not_reply = false
    },
    async loadData() {
      this.loading = true
      try {
        let list = await this.getData()
        const oldIdMap = this.getS()
        const idMap = {}
        let sum = 0
        let sum1 = 0
        list = list.map(item => {
          const oldItem = oldIdMap[item.id] || {}

          idMap[item.id] = {
            ...oldItem,
            comments: item.comments,
          }
          item.is_new = item.comments > oldItem.comments
          if (item.is_new) {
            sum1 += 1
          }
          item.not_reply = item.comments === 0 && oldItem.delApply !== true
          if (item.comments === 0 && oldItem.delApply !== true) {
            sum += 1
          }
          return item
        })
        this.list = list
        this.setS(idMap)
        this.__exp()
        if (sum === 0 && sum1 === 0) {
          return
        }
        this.notice({ title: 'EMQ X issues 提醒', body: `你有 ${sum} 条未回复的 issues, ${sum1} 条新动态` })
      } catch (e) {
        console.log(e)
        this.loading = false
      }
      this.loading = false
      this.reloadDate = this.getNow()
    },
    getNow() {
      const d = new Date()
      return `${d.toLocaleTimeString()}`
    }
  },
  filters: {
    date(date) {
      if (!date) {
        return 'N/A'
      }
      const d = new Date(date)
      return `${d.toLocaleDateString()} ${d.toLocaleTimeString()}`
    }
  },
  created() {
    Notification.requestPermission(function (status) {
    });
    this.loadData()
    clearInterval(this.timer)
    this.timer = setInterval(this.loadData, this.inv * 1000)
  }
}
</script>

<style>
.time-label {
  margin-left: 12px;
}
.header {
  color: #333;
  padding: 42px;
}
a {
  text-decoration: none;
  color: #444;
}
a:hover {
  text-decoration: solid;
}
.item {
  margin-left: 20px;
}
.item-list {
  list-style: none;
  display: flex;
  align-items: center;
  margin: 4px auto;
  padding: 4px 6px;
  transition: all 0.3s;
  border: 1px dashed transparent;
}
.item-list.is_new {
  border-color: red;
}
.item-list:hover {
  border-color: #d8d8d8;
}
.user-info {
  font-size: 12px;
  display: flex;
  align-items: center;
  width: 120px;
  color: #999;
}
.user-avatar {
  width: 28px;
  height: 28px;
  border-radius: 100%;
  margin-right: 10px;
}
.item-label {
  font-size: 12px;
  color: #999;
}
.item-label.is-new {
  color: red;
  font-weight: bolder;
  font-size: 13px;
}
.btn {
  cursor: pointer;
  margin-left: 10px;
  visibility: hidden;
  transition: all 0.15s;
}
.item-list:hover .btn {
  visibility: visible;
}
.item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
}
.item-footer {
  float: right;
}
.dark {
  color: #333;
}
</style>
