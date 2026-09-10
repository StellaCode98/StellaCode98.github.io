---
title: Pinia 与 Vuex 状态管理总结：从单向数据流到组合式 Store
date: 2026-09-10 21:30:00
description: 一篇文章讲透 Vue 两个官方状态管理库：Vuex 的单向数据流与 mutations 存在的理由、它的五个痛点，Pinia 如何用组合式 API 逐个拆掉这些包袱；含 Option/Setup 双写法对照、$patch/$subscribe/$onAction 与插件机制原理、storeToRefs 解构响应性、组件外使用 store、Vuex → Pinia 迁移映射表与高频面试题。
categories:
  - [Vue, 状态管理]
tags:
  - Vue3
  - Vue2
  - Pinia
  - Vuex
  - 状态管理
---

这两个库解决的问题只有一个：**多个组件共享同一份状态**。但它们给出的答案，恰好对应了 Vue2 和 Vue3 两代技术栈的世界观——Vuex 是「中心化的单一仓库 + 单向数据流」，Pinia 是「一组全局的组合式函数」。

> **先说结论**：Vuex 强制把「改状态」拆成 mutations（同步）和 actions（异步），是为了在 Options API 时代让 DevTools 能可靠追踪每一次变更；Vue3 的响应式系统本身就能观测到任何途径的修改，这层强制拆分失去了意义，于是 Pinia 把它删掉了，同时用「天然按 id 拆分的扁平 store」取代了 Vuex 的嵌套 modules。新项目（Vue3）直接用 Pinia，没有第二个选项；Vue2 老项目继续 Vuex，不必强行迁移。

<!-- more -->

## 一、为什么需要状态管理

组件化把页面拆成了树，但状态经常不属于任何一棵子树——登录态、用户信息、购物车、主题配置，都是「很多组件都要读写」的东西。不借助任何方案时，只有两条路：

**1. props 逐层透传（props drilling）**

```vue
<!-- App.vue 知道 user，但真正用它的是 5 层之下的 Avatar.vue -->
<App :user="user" />
  └─ <Layout :user="user" />
       └─ <Sidebar :user="user" />
            └─ <Panel :user="user" />
                 └─ <Avatar :user="user" />  <!-- 中间三层只是搬运工 -->
```

中间组件被迫声明与自己无关的 props，数据流变成了体力劳动；而且兄弟组件之间这条路根本不通，得先把事件冒泡到公共祖先再传下来。

**2. 全局事件或全局变量**

挂 `window`、用 EventBus 面包屑式通知，状态散落在各处，没有统一的「单一数据源」（Single Source of Truth），排查问题时不知道值是谁改的。

状态管理库做的事因此很朴素：**把共享状态提到组件树外的一个全局仓库里，任何组件都能直接读写，同时保留响应式和可追踪性**。

但也要警惕反面：不是所有状态都该进全局仓库。表单草稿、弹窗开关、输入框焦点这类**真正的局部状态**，用 `ref`/`reactive` 留在组件内就好；只是嫌透传麻烦的话，`provide/inject` 或一个模块级 `ref`（见 7.5）可能比引一个库更合适。**先有共享需求，再上状态管理。**

## 二、Vuex：中心化仓库与单向数据流

### 2.1 心智模型

Vuex 的核心是一张经典图：

```text
components ──dispatch──> actions（可异步）──commit──> mutations（必须同步）──> state
    │                                                        │
    └──────────────── commit（直接同步改）───────────────────┘
                                 │
                          getters ──派生──> components
```

规则很明确：

- **state**：唯一数据源，挂在 `store.state` 上；
- **getters**：state 的计算属性（缓存）；
- **mutations**：**唯一合法的修改入口**，必须是同步函数，第一个参数是 state；
- **actions**：提交 mutation，可以异步（发请求、定时器），第一个参数是上下文对象；
- 组件里 `commit('xxx')` 改状态，`dispatch('xxx')` 触发异步流程。

### 2.2 完整示例

```js
// store/index.js
import { createStore } from 'vuex'

const userModule = {
  namespaced: true, // 关键：开启命名空间，否则所有模块的 mutations/getters 都注册在全局
  state: () => ({
    token: '',
    userInfo: null,
  }),
  getters: {
    isLoggedIn: (state) => Boolean(state.token),
  },
  mutations: {
    SET_TOKEN(state, token) {
      state.token = token
    },
    SET_USER_INFO(state, info) {
      state.userInfo = info
    },
  },
  actions: {
    async login({ commit }, { username, password }) {
      const { token, userInfo } = await api.login(username, password)
      commit('SET_TOKEN', token)
      commit('SET_USER_INFO', userInfo)
    },
  },
}

export default createStore({
  modules: { user: userModule },
})
```

```vue
<script>
import { mapState, mapGetters } from 'vuex'

export default {
  computed: {
    // 借助命名空间字符串 + 辅助函数
    ...mapState('user', ['userInfo']),
    ...mapGetters('user', ['isLoggedIn']),
  },
  methods: {
    // 没有辅助函数时，是字符串路径
    handleLogin() {
      this.$store.dispatch('user/login', { username: 'admin', password: 'xxxxxxx' })
    },
  },
}
</script>
```

### 2.3 为什么强制 mutations 必须同步

这是 Vuex 最常被抱怨的设计，但它不是洁癖，是**工程上的必要**：

DevTools 的时间旅行（time travel）依赖「每个 mutation 执行完，拍一次状态快照」。如果 mutation 里有 `setTimeout`，快照拍下的会是**中间状态**，回放时状态对不上，整个调试模型就垮了。所以 Vuex 干脆用约定（严格模式下直接报错）堵死异步 mutation：

```js
// 开发环境开启严格模式后，绕过 mutation 直接改 state 会抛错
store.state.user.token = 'xxx' // ❌ Error: [vuex] do not mutate vuex store state outside mutation handlers
```

理解了这一点就理解了 Vuex 的一半：**mutations/actions 的分裂，是给 DevTools 交的「税」**。

### 2.4 Vuex 的五个痛点

1. **mutations / actions 样板代码**。一个登录要写两个 mutation + 一个 action；同步逻辑被迫三处跳转。明明只是 `state.token = token` 一行的事。
2. **modules 嵌套 + 字符串路径**。`dispatch('user/login')`、`mapState('user', [...])`、子模块里取兄弟模块要 `rootState` / `rootGetters`——类型和重构都不友好，改名只能全局搜字符串。
3. **TypeScript 支持弱**。Vue2 的 `this.$store` 没有类型上下文，需要手写 `declare module 'vuex'` 类型增强和 InjectionKey；`mapState` 一族辅助函数基本放弃推导，getters 的返回类型要逐个手标。
4. **单入口大仓库**。所有模块集中注册进一个 store，代码分割、按需加载都别扭；`store.state.user.xxx` 的深层访问把模块结构焊死。
5. **action 上下文对象心智负担**。`{ state, commit, dispatch, rootState, rootGetters }` 一堆参数，嵌套模块里 `dispatch('other/action', null, { root: true })` 这类调用极易写错。

Vuex4（Vue3 版）只是把内核换成 `reactive()`，这些 API 层面的债原样保留——官方也明确说了 Vuex 进入维护模式，Vue3 的答案是 Pinia。

## 三、Pinia：把 store 写成组合式函数

### 3.1 设计哲学

Pinia（「菠萝」）不再维护一棵模块树，而是：**每个 store 就是一个带响应式的全局对象，靠 `defineStore(id, ...)` 的 id 天然拆分**。它建立在 Vue3 响应式 API 之上，没有引入自己的响应式系统——这是它能把 API 砍薄的根本原因。

### 3.2 两种写法

**Option Store**（长得像 Vuex 模块，迁移友好）：

```js
// stores/user.js
import { defineStore } from 'pinia'

export const useUserStore = defineStore('user', {
  state: () => ({
    token: '',
    userInfo: null,
  }),
  getters: {
    isLoggedIn: (state) => Boolean(state.token),
    // getter 间互访用 this（需要显式标返回类型）
    displayName(state) {
      return this.userInfo?.name ?? '游客'
    },
  },
  actions: {
    // ✅ 同步、异步都在 action 里，改 state 直接改
    async login(username, password) {
      const { token, userInfo } = await api.login(username, password)
      this.token = token
      this.userInfo = userInfo
    },
    logout() {
      this.$reset() // 恢复到 state() 的初始值
    },
  },
})
```

**Setup Store**（组合式 API 写法，推荐新代码使用）：

```js
// stores/user.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useUserStore = defineStore('user', () => {
  // ref()   → state
  const token = ref('')
  const userInfo = ref(null)

  // computed() → getters
  const isLoggedIn = computed(() => Boolean(token.value))

  // function  → actions
  async function login(username, password) {
    const res = await api.login(username, password)
    token.value = res.token
    userInfo.value = res.userInfo
  }

  return { token, userInfo, isLoggedIn, login }
})
```

Setup 写法的额外收益：store 内可以自由使用 `watch`、`provide/inject`、路由实例甚至其他组合式函数，且能按需只暴露部分能力（比如不 return 某个 ref，它就成了 store 的「私有状态」）。一个约定：**state 用 `ref`，getter 用 `computed`，action 用普通函数**。

### 3.3 组件里使用

```vue
<script setup>
import { storeToRefs } from 'pinia'
import { useUserStore } from '@/stores/user'

const userStore = useUserStore()

// ⚠️ 直接解构会丢响应性（见 5.1）
const { token, userInfo } = storeToRefs(userStore)
// action 是普通函数，可以直接解构
const { login } = userStore

// 直接读写
userStore.token = 'new-token'
await login('a', 'b')
</script>
```

注意连 `$store` 都没有了——store 本身就是那个对象，`userStore.token` 直接访问。

### 3.4 为什么删掉 mutations 还能时间旅行

这是 Pinia 最核心的一个「为什么」。

Vuex 时代，追踪变更是靠「你承诺只通过 mutation 改」，DevTools 钩在 mutation 的执行边界上拍快照。Pinia 换了思路：**state 本身就是 `reactive` 对象，任何途径的修改（action 里改、组件里改、`$patch` 批量改）都会触发响应式依赖**。Pinia 通过 `$subscribe` 订阅这层响应式变化，把每次变更连同时间戳记录为事件序列交给 DevTools——不需要你先承诺「从哪个门进」，**门本身消失了，但每个访客仍然被登记**。

所以严格模式也没有存在的必要了：直接 `store.token = 'x'` 不会破坏任何追踪能力，DevTools 照样能看到这次变更。代价只是「变更入口分散、不利于约束团队规范」这类软性问题，由团队规范而不是框架来管。

### 3.5 实例 API：$patch / $reset / $subscribe / $onAction

```js
const userStore = useUserStore()

// $patch：批量修改，触发一次订阅通知（对象式与函数式）
userStore.$patch({ token: 'a', userInfo: { name: 'k' } })
userStore.$patch((state) => {
  state.userInfo.name = 'k'
  state.token = 'a'
})

// $reset：重置为初始 state（⚠️ 仅 Option Store 自带；
// Setup Store 没有 state() 工厂，需自己实现一个 reset action）

// $subscribe：监听 state 变化（持久化的基础）
userStore.$subscribe((mutation, state) => {
  // mutation.type: 'direct' | 'patch object' | 'patch function'
  // mutation.storeId: 'user'
  localStorage.setItem('user', JSON.stringify(state))
}, { detached: true }) // detached: 组件卸载后仍继续订阅

// $onAction：拦截 action 的执行（日志、埋点、错误上报）
const unsubscribe = userStore.$onAction({
  name: 'login',
  after: (result) => console.log(`${name} 成功`, result),
  onError: (err) => console.error(`${name} 失败`, err),
})
```

### 3.6 原理速览

Setup Store 的实现简单到可以口述：`defineStore(id, setup)` 内部用 `effectScope()` 开一个独立作用域，在里面执行你的 setup 函数——`ref` 就是 state，`computed` 就是 getters，返回的对象整体包一层 `reactive`（所以 `userStore.token` 会自动解包 ref）。`effectScope` 保证这些副作用可以被统一追踪和销毁，与任何组件实例**无耦合**——这就是 store 能活在组件外、且支持 HMR（热更新时销毁旧 scope 重建）的原因。相比之下，Vuex3 是把 state 塞进一个隐藏的 `new Vue({ data })` 实例、Vuex4 换成 `reactive()`，历史包袱一目了然。

## 四、逐项对比

| 维度 | Vuex 4 | Pinia |
| --- | --- | --- |
| 心智模型 | 单一大仓库 + 嵌套 modules | 扁平的多 store，按 id 拆分 |
| 修改入口 | 必须经 mutations（同步）/ actions（异步） | action 或任意处直接改，`$patch` 批量 |
| TypeScript | 弱，需手写类型增强 | 全链路自动推导 |
| 组合式 API | 先天不适配（Vuex4 仅兼容运行） | 原生即为组合式设计 |
| 模块间互访 | `rootState` / `rootGetters` / `{ root: true }` | 直接 `useOtherStore()`，或 getter/action 里组合 |
| 代码分割 | 单入口，别扭 | 每个 store 独立文件，天然友好 |
| 体积 | 约数 KB（gzip） | 约 1KB（gzip） |
| DevTools | 时间旅行、时间线 | 同样支持，且 store 树更直观 |
| 插件 | store 插件 | context 更丰富（含 `$subscribe`/`$onAction` 钩子） |
| 严格模式 | 有（dev 下抓非法修改） | 无，也不需要（见 3.4） |
| Vue 版本 | Vue2（Vuex3）/ Vue3（Vuex4） | Vue3；Vue 2.7 也可用 |

几个值得展开的点：

**TypeScript 是体验差距最大的地方。** Pinia 里从 `state` 到 `getters` 到组件里的 `storeToRefs`，类型全程自动流动，改一个接口字段，所有消费处编译器立刻标红。Vuex 里类型在 `commit('user/SET_TOKEN', ...)` 这种字符串处断裂，只能靠自律。

**store 之间可以像普通模块一样组合**，这是 Vuex 做不到的：

```js
// stores/cart.js 里使用 user store
export const useCartStore = defineStore('cart', () => {
  const items = ref([])

  const userStore = useUserStore() // 在 getter/action 执行时调用即可

  const totalPrice = computed(() =>
    userStore.userInfo?.vip
      ? items.value.reduce((s, i) => s + i.price * 0.9, 0)
      : items.value.reduce((s, i) => s + i.price, 0),
  )

  return { items, totalPrice }
})
```

注意：在 setup store 的**顶层**调用其他 store 是允许的（只要那时 pinia 已激活），跨 store 循环依赖时则要放到 getter/action 内部延迟调用。

## 五、实战技巧与易错点

### 5.1 解构丢响应性：必须用 storeToRefs

```js
const { token } = useUserStore() // ❌ token 变成一次性的纯字符串
```

原因和 `props` 解构一样：`store.token` 是对 reactive 对象属性的**读取**，拿到的是值的快照，丢失了与源对象的联系。`storeToRefs` 的实现本质就是对 state 和 getters 逐个 `toRef`：

```js
const { token } = storeToRefs(userStore) // ✅ Ref<string>，保持响应
const { login } = userStore              // ✅ action 是普通函数，随便解构
```

### 5.2 在组件外使用 store

store 依赖已安装的 pinia 实例，而模块顶层的执行时机早于 `app.use(pinia)`：

```js
// ❌ 模块加载时立即调用，此时 pinia 还没激活
const userStore = useUserStore() // getActivePinia was called with no active Pinia

// ✅ 方案一：延迟到运行时调用（路由守卫、axios 拦截器的回调本身就是延迟执行的）
router.beforeEach((to) => {
  const userStore = useUserStore() // 此时 app 已启动，OK
  if (to.meta.auth && !userStore.isLoggedIn) return '/login'
})

// ✅ 方案二：模块顶层就要用，显式传入 pinia 实例
import { pinia } from './pinia' // 单独导出的 pinia 实例
const userStore = useUserStore(pinia)
```

方案二是 SSR 和单元测试（每次 createTestingPinia）场景的标准做法。

### 5.3 持久化：一个 20 行的插件

Pinia 插件是一个函数，作用于每个 store，返回值会被合并进 store（可用于添加 `$reset` 增强等）：

```js
// plugins/persist.js
export function persist({ store }) {
  const saved = localStorage.getItem(store.$id)
  if (saved) store.$patch(JSON.parse(saved))

  store.$subscribe((_mutation, state) => {
    localStorage.setItem(store.$id, JSON.stringify(state))
  })
}

// main.js
pinia.use(persist)
```

生产环境一般直接用 `pinia-plugin-persistedstate`，支持按 store 配置 `persist: { paths: ['token'] }` 做字段级持久化——原理就是上面这段代码加配置化。

### 5.4 Setup Store 的私有状态与 $reset

setup 函数里不 return 的变量就是 store 的私有状态，外界（包括 DevTools）看不到；代价是 `$reset` 失效，需要自己写：

```js
export const useUserStore = defineStore('user', () => {
  const token = ref('')

  function $reset() {
    token.value = ''
  }
  return { token, $reset }
})
```

### 5.5 「轻量替代品」：模块级 ref

如果只是两三个组件共享一个小状态，可以不引库：

```js
// composables/useTheme.js
import { ref } from 'vue'
const theme = ref('light') // 模块作用域，天然全局单例
export function useTheme() {
  const toggle = () => (theme.value = theme.value === 'light' ? 'dark' : 'light')
  return { theme, toggle }
}
```

它的本质就是一个「手写的 setup store」。区别在于没有 DevTools 集成、`$patch`、插件体系和 SSR 安全——状态规模一上去，这些就会从「nice to have」变成刚需，那时再换 Pinia，迁移成本几乎为零（因为写法同构）。

## 六、Vuex → Pinia 迁移指南

两个库可以**共存过渡**（Vuex 照常 `app.use(store)`，Pinia 照常 `app.use(pinia)`），建议按模块逐个迁移、迁移完删除。

映射关系：

| Vuex | Pinia (Option) | 说明 |
| --- | --- | --- |
| `state: () => ({...})` | `state: () => ({...})` | 一致 |
| `getters` | `getters` | 去掉 namespaced 概念 |
| `mutations` + `actions` | `actions` | 合并；`SET_XXX` 大写命名可顺手改成语义化方法 |
| `modules: { user }` | `defineStore('user', ...)` 独立文件 | 嵌套模块拍平 |
| `this.$store.commit('user/login')` | `userStore.login()` | 字符串 → 方法调用 |
| `mapState('user', [...])` | `storeToRefs(useUserStore())` | |
| `store.dispatch('user/login')` | `userStore.login()` | |

迁移前的 user 模块（Vuex）和迁移后（Pinia）对照：

```js
// ── Vuex ──
const userModule = {
  namespaced: true,
  state: () => ({ token: '' }),
  mutations: {
    SET_TOKEN(state, token) { state.token = token },
    CLEAR_TOKEN(state) { state.token = '' },
  },
  actions: {
    async login({ commit }, form) {
      const { token } = await api.login(form)
      commit('SET_TOKEN', token)
    },
    logout({ commit }) {
      commit('CLEAR_TOKEN')
    },
  },
}

// ── Pinia ──
export const useUserStore = defineStore('user', {
  state: () => ({ token: '' }),
  actions: {
    async login(form) {
      this.token = (await api.login(form)).token
    },
    logout() {
      this.$reset()
    },
  },
})
```

代码量减半不是重点，重点是**字符串派发全部变成了可跳转、可推导类型的方法调用**。

## 七、高频面试题速答

**1. 为什么 Pinia 去掉 mutations？**
因为 mutations 存在的唯一理由（同步边界保证 DevTools 快照可靠）在 Vue3 响应式系统下不再必要：state 是 `reactive` 对象，任何途径的修改都可被 `$subscribe` 观测并记录，DevTools 能力无损。

**2. Pinia 为什么不需要 strict 模式？**
同上。Vuex 的严格模式防的是「绕过 mutation 导致 DevTools 看不见」；Pinia 里不存在看不见的修改，直接改 state 只是风格问题，不是正确性问题。

**3. 直接解构 store 为什么丢响应性？**
`const { token } = store` 是对 reactive 属性的一次性读取，拿到原始值。要用 `storeToRefs`（内部 `toRef` 保持引用联系）；action 是普通函数不受影响。

**4. Pinia 和 Vuex 会长期并存吗？**
不会。Pinia 官方定位就是「下一代 Vuex」，Vuex 已进入维护模式（只修 bug 不加特性），Vue 官方文档状态管理章节只推荐 Pinia。

**5. 没有嵌套 modules，大型项目几十个 store 不会乱吗？**
「扁平 + 组合」优于「树形 + 命名空间」：模块间依赖显式化为 `useOtherStore()` 调用，跨域复用靠组合而非 `rootState`；按业务域拆文件（`stores/user.js`、`stores/cart.js`）后，目录即架构。

**6. $patch 和直接赋值有什么区别？**
`$patch` 把多次修改合并为**一次**订阅通知，并且以 `patch object` 类型记录进 DevTools；批量更新时用它可减少订阅回调（如持久化写 localStorage）的触发次数。

## 八、总结

| | 一句话 |
| --- | --- |
| Vuex 的本质 | 用「约定 + 强制」换可追踪性：mutations 是给 DevTools 交的税 |
| Pinia 的本质 | 响应式系统本身可观测，税不用交了；store 即组合式函数，模块化即文件 |
| 选型 | Vue3 新项目 → Pinia（唯一答案）；Vue2 存量 → Vuex4 不必动；简单共享 → 模块级 ref / provide-inject |

真正值得带走的不是 API 对照表，而是这条主线：**状态管理库的一切设计，都是围绕「如何让共享状态的变化可观测、可追踪」展开的**——Vuex 用纪律实现它，Pinia 用代理实现的响应式系统实现它。理解了这一点，下一代的方案再怎么变，你都能在十分钟内看懂它。
