---
title: 前端权限控制实战：从 token 到 v-auth 指令，数据权限放在哪一层？
date: 2026-09-09 21:30:00
description: 以一个真实后台项目为例复盘前端权限体系：登录凭证与权限快照的存取、「模块:动作」权限码模型、v-auth / v-menuAuth 双指令与 Tab 级显隐，以及最容易被误解的问题——数据权限到底该由前端还是后端负责。
categories:
  - [Vue 进阶]
tags:
  - Vue3
  - RBAC
  - Pinia
  - Axios
---

> 这是真实项目复盘系列的第二篇。这个 Vue3 + Vite + Pinia + Ant Design Vue 的后台系统里，权限是典型的"隐形基础设施"：平时没人注意它，直到某天测试提了个 bug——"为什么这个角色看不到导出按钮"。本文把这套权限体系完整拆开：**登录凭证怎么存、权限码怎么建模、按钮和菜单怎么按权限显隐、角色权限在哪个界面配出来**，以及一个几乎所有后台项目都会遇到、也最容易做错的问题——**数据权限（数据范围过滤）到底放在前端还是后端**。先说结论：这个项目把数据权限完全交给了后端，前端只做了功能层（菜单/按钮）权限。为什么这样选、边界在哪，第 7 节详细展开。

## 目录

- [0. 前言：前端权限到底防谁](#sec0)
- [1. 登录与凭证：token 从哪来，权限快照存哪去](#sec1)
- [2. 权限数据模型：两张 Map 与「模块:动作」权限码](#sec2)
- [3. 判断核心：hasBtnAuth / hasAuth 双函数](#sec3)
- [4. 双指令：v-auth 与 v-menuAuth](#sec4)
- [5. 三层显隐实战：模块入口 → Tab → 按钮](#sec5)
- [6. 权限的配置面：角色管理与权限树](#sec6)
- [7. 数据权限：这一层前端做了什么、该做什么](#sec7)
- [8. 短板与改进清单](#sec8)
- [9. 写在最后](#sec9)

<a id="sec0"></a>

## 0. 前言：前端权限到底防谁

先把一个立场立住，后面所有设计才立得住：**前端权限从来不是安全机制，而是体验机制**。

用户打开 DevTools 就能改 localStorage、就能直接 curl 你的接口。所以前端权限真正回答的不是"怎么阻止未授权操作"，而是"**怎么让不同角色看到与身份相符的界面，不误操作、不困惑**"。真正的安全边界永远在后端——每一个接口都要自己校验"这个 token 有没有资格调我"。

理解了这一点，这个项目的很多"非主流"选择就顺理成章了。整个权限体系分三层：

```text
┌──────────── 认证层：你是谁 ────────────────────────┐
│ login → token → localStorage                       │
│ axios 拦截器：Authorization: Bearer <token>          │
│ 响应 code 401/601 → 踢回 /login（后端兜底）          │
└──────────────┬─────────────────────────────────────┘
               ▼
┌──────────── 功能层：你能看到什么、点什么 ──────────────┐
│ GET /sysPermission/getPermissionByUser（靠 token 识别）│
│   ├─ buttonPermissionMap → localStorage.btnObj       │
│   └─ menuPermissionMap  → localStorage.menuObj       │
│ 判断：hasBtnAuth / hasAuth（同步读 localStorage）      │
│ 消费：v-auth（按钮） · v-menuAuth（模块入口）           │
│       hasAuth + v-if（Tab） · hasAuth（下拉菜单项）    │
└──────────────┬─────────────────────────────────────┘
               ▼
┌──────────── 数据层：你能拿到哪些数据 ──────────────────┐
│ 前端：无显式实现（部门=用户属性，地区=业务筛选条件）     │
│ 实际：后端按 token 过滤数据范围，前端只负责渲染          │
└─────────────────────────────────────────────────────┘
```

一个和主流后台模板（vue-element-admin 那套）很不一样的前提：**这个系统没有多路由页面**。整个应用只有 `/login` 和 `/home` 两个路由，所有业务模块（设施管理、指标体系、评估任务、用户管理……）都是 `/home` 内部通过 `moduleStore.activeModule` + `v-if` 切换的组件。所以经典方案里的"动态路由 + addRoute + permission store"在这完全没有用武之地——**没有路由可以管权限，权限管的是组件**。这个前提让整个权限方案变得非常薄，也暴露了它最本质的形态。

<a id="sec1"></a>

## 1. 登录与凭证：token 从哪来，权限快照存哪去

### 1.1 登录链路

登录逻辑收在 Pinia 的 user store 里，流程是清晰的四步：

```typescript
// src/store/user.ts（有删减）
async login(params: any) {
	sysUserlogin(params).then(res => {
		if (res.code == 200) {
			this.setToken(res.data.token);              // ① 存 token
			let userInfo = {
				username: res.data.username,
				displayName: res.data.displayName
			};
			sysPermissionList().then(res => {            // ② 拉权限
				if (res.code == 200 && res.data) {
					localStorage.setItem('btnObj', JSON.stringify(res.data.buttonPermissionMap));
					localStorage.setItem('menuObj', JSON.stringify(res.data.menuPermissionMap));
					localStorage.setItem('userInfo', JSON.stringify(userInfo));  // ③ 存用户信息
					router.push('/home');            // ④ 跳首页
				}
			});
		}
	});
}
```

注意两个设计点：

**权限接口不带参数**。`GET /sysPermission/getPermissionByUser` 连 userId 都不传——后端从请求头的 token 里识别用户。这是正确的做法：**前端声明的身份不可信，凭证里的身份才可信**。同理，登录响应里也只有 `username/displayName/token`，没有角色列表。

**权限快照存 localStorage，而不是 Pinia state**。这是这个项目最值得说道的取舍。store/user.ts 的 state 里只有 `token` 和 `userInfo`，权限码两张 Map 完全不进 store。为什么？因为权限的消费方是**自定义指令**——指令的 `mounted` 钩子在组件 setup 之外执行，拿不到组件实例的 store 上下文，如果权限在 Pinia 里，指令就得在模块顶层手动初始化 pinia 再取 store（这个项目在 directive/index.ts 顶部确实 `import pinia from '@/store'` 做了这件事，但只用于引入结构）。localStorage 是同步全局可读的，指令里一行 `localStorage.getItem('btnObj')` 就能拿到，代价是**权限快照与登录态耦合在浏览器存储里**——第 8 节会讲这个代价。

### 1.2 请求层：token 注入与 401 兜底

axios 拦截器承担全部凭证工作：

```typescript
// src/service/service.ts（有删减）
const getToken = function (): string {
	return window.localStorage.getItem('token') || '';
};

service.interceptors.request.use(config => {
	if (getToken()) {
		config.headers.Authorization = 'Bearer ' + getToken();  // 有 token 则加 token
	}
	return config;
});

service.interceptors.response.use(response => {
	const { data } = response;

	// 文件流（excel / octet-stream）直接放行
	// ...

	if (data.code === 401 || data.code === 601) {
		router.replace({ path: '/login' });   // 唯一的登录态失效处理
		return Promise.reject(data);
	}
	if (data.code !== 200) {
		message.error(data.msg || '未知错误');
		return Promise.reject(data);
	}
	return data;
});
```

值得注意 `validateStatus: status >= 200 && status <= 500`——把 4xx/5xx 也当"成功"响应交给业务层，让后端业务码（200/401/601）成为唯一的状态语言。这样 401 跳转、错误提示的逻辑都收敛在响应拦截器一处。

### 1.3 一个"缺失"的路由守卫

这是整个权限体系里最让熟悉 vue-element-admin 的人意外的地方：

```typescript
// src/router/index.ts（完整）
router.beforeEach(to => {
	setTitle(to);   // 只设置 document.title
});
```

**没有 token 校验、没有白名单、没有重定向**。未登录直接访问 `/#/home` 不会被拦截——页面会正常渲染，只是因为 `btnObj/menuObj` 不存在而全部按钮被禁用，第一个接口请求回来 401 再被踢回 `/login`。

这算 bug 吗？功能上不算（后端兜底闭环了），体验上算（用户会看到一屏禁用按钮闪一下）。改进方案第 8 节给出，就三行代码的事。

<a id="sec2"></a>

## 2. 权限数据模型：两张 Map 与「模块:动作」权限码

后端下发的权限长这样（项目里留有一份完整的权限码清单 mock，可以直接当文档看）：

```json
{
	"buttonPermissionMap": {
		"set":      ["set:view", "set:delete", "set:add", "set:edit"],
		"model":    ["model:delete", "model:view", "model:edit", "model:add"],
		"rules":    ["rules:view"],
		"workflow": ["workflow:view"],
		"plan":     ["plan:add", "plan:edit", "plan:delete", "plan:view"],
		"facility": ["facility:add", "facility:view", "facility:edit", "facility:delete"]
	},
	"menuPermissionMap": {
		"indicator":  ["indicator:set", "indicator:model", "indicator:rules", "indicator:workflow", "indicator:system"],
		"assess":     ["assess:assessTask"],
		"management": ["management:facility"],
		"user":       ["user:userManagement", "user:permissionManagement"]
	}
}
```

模型要点：

1. **权限码命名规范是 `模块:动作`**。`plan:edit` 一眼可读：plan 模块的编辑权。不带 `:` 的是模块级权限（如 `user`），控制模块入口的可见性。
2. **按钮权限和菜单权限是两张独立的 Map**，而不是一棵树。后端数据库里它们是一棵权限树（第 6 节），下发时按用途拍平成两组。前端消费时互不干扰：按钮判断查 `btnObj`，菜单/Tab 判断查 `menuObj`。
3. **分组结构是"模块 → 权限码数组"**，判断时不需要知道权限码属于哪个模块——遍历所有模块找就是了。这带来一个好处：前端写 `v-auth="'plan:edit'"` 时不用关心后端把它归在哪个 key 下，耦合度更低。

为什么菜单和按钮要分开？因为它们的**失效表现不同**：菜单/Tab 没权限应该不可见或不可进入，按钮没权限可以是"可见但禁用"。同一个权限模型，两种交互策略。

<a id="sec3"></a>

## 3. 判断核心：hasBtnAuth / hasAuth 双函数

两个纯函数是所有权限判断的唯一出口，都在 `src/directive/index.ts`：

```typescript
export const hasBtnAuth = (permission: string): boolean => {
	const btnStr = localStorage.getItem('btnObj');
	if (!btnStr) return false;

	try {
		const btnObj = JSON.parse(btnStr);

		// 情况1: 权限是扁平数组
		if (Array.isArray(btnObj)) {
			return btnObj.includes(permission);
		}

		// 情况2: 权限是分组对象（当前格式）
		if (typeof btnObj === 'object' && btnObj !== null) {
			for (const moduleKey in btnObj) {
				const permissions = btnObj[moduleKey];
				if (Array.isArray(permissions) && permissions.includes(permission)) {
					return true;
				}
			}
			return false;
		}
		return false;
	} catch (e) {
		console.error('❌ 解析 btnObj 失败，请检查 localStorage 格式:', e);
		return false;
	}
};
```

`hasAuth`（查 menuObj）多一个分支：**不带 `:` 的参数按"模块级权限"处理**——只要 menuObj 里存在这个模块 key 就算有权限：

```typescript
export const hasAuth = (permission: string): boolean => {
	const menuStr = localStorage.getItem('menuObj');
	if (!menuStr) return false;
	const menuObj = JSON.parse(menuStr);

	// 情况1: permission 是模块名（不包含 ':'），如 'user'
	if (!permission.includes(':')) {
		return Object.prototype.hasOwnProperty.call(menuObj, permission);
	}
	// 情况2: 细粒度权限，如 'indicator:set'，遍历所有模块匹配
	for (const moduleKey in menuObj) {
		const permissions = menuObj[moduleKey];
		if (Array.isArray(permissions) && permissions.includes(permission)) return true;
	}
	return false;
};
```

三个值得吸收的细节：

- **兼容"扁平数组"和"分组对象"两种数据格式**。这不是过度设计——后端接口结构演进过一轮（mock 文件里两种格式的注释都留着），兼容分支让前端在格式切换期不用发版。对权限这种横切所有页面的基础设施，**消费端对数据格式的容忍度应该拉满**。
- **fail-closed：任何解析失败、数据缺失都返回 false**。权限判断的默认值只能是"无权限"。`!btnStr return false`、`catch return false`，方向一致。
- **try/catch 包住 JSON.parse**。localStorage 是用户可污染的存储，损坏的 JSON 不应该让整页指令抛错白屏。

<a id="sec4"></a>

## 4. 双指令：v-auth 与 v-menuAuth

指令在 main.ts 里一行注册：`authDirective(app)`。两个指令的实现几乎一样，差异只在判断函数：

```typescript
// src/directive/index.ts（有删减）
export const authDirective = (app: any) => {
	// 按钮级权限
	app.directive('auth', {
		mounted(el: any, options: any) {
			if (!hasBtnAuth(options.value)) {
				el.setAttribute('disabled', 'disabled');
				el.style.pointerEvents = 'none';   // 防点击
				el.style.opacity = '0.6';          // 视觉禁用
			}
		}
	});

	// 菜单/Tab 级权限
	app.directive('menuAuth', {
		mounted(el: any, options: any) {
			const { value: permission } = options;
			if (typeof permission !== 'string') {
				console.warn('⚠️ v-menuAuth 必须传入字符串权限标识');
				return;
			}
			if (!hasAuth(permission)) {
				el.setAttribute('disabled', 'disabled');
				el.style.pointerEvents = 'none';
				el.style.opacity = '0.6';
			}
		}
	});
};
```

**"禁用三件套"是这个方案的一个明确选择**：`disabled` 属性 + `pointerEvents: none` + `opacity: 0.6`。源码注释里留着被放弃的另一个方案：

```typescript
// if (!btnObj || !btnObj.includes(options.value)) {
// 	// 如果在用户信息 buttons 数组当中没有，就从 DOM 上干掉
// 	el.parentNode.removeChild(el);
// }
```

**移除 DOM vs 禁用**，两种策略的取舍：

| | 移除 DOM（removeChild） | 禁用三件套 |
| --- | --- | --- |
| 用户感知 | "功能不存在" | "功能存在但无权用" |
| 布局 | 兄弟元素位移，布局跳动 | 稳定 |
| 恢复 | 几乎不可逆（节点已丢） | 改 style 即可 |
| 适用 | 导航型（不该知道的入口） | 操作型（增删改按钮） |

这个项目最终统一用禁用，理由是后台场景里用户通常**需要知道"有这个功能，但我没权限"**——配上 tooltip 说明原因，比入口凭空消失的体验好；同时 DOM 稳定对自动化测试也更友好。代价是"藏起来的秘密入口"会暴露在 DOM 里——但回到第 0 节的立论：前端权限本来就不承担保密职责。

还有一个容易踩的坑值得点出：**指令只在 `mounted` 执行一次**。权限快照是登录后一次性写入 localStorage、会话内不变的，所以"一次判断"够用；但如果你的系统支持"不刷新页面热更新权限"，`mounted` 一次性的指令就不够了，需要用 `updated` 钩子或在权限变化时强制重渲染（比如给 router-view 加 key）。

<a id="sec5"></a>

## 5. 三层显隐实战：模块入口 → Tab → 按钮

### 5.1 第一层：顶部模块入口（v-menuAuth）

顶部导航的每个模块入口挂指令，权限码就是模块名：

```html
<!-- src/pages/home/components/home-header/index.vue（有删减） -->
<div class="home-tittle-bg"
	:class="['cp', isActive('targetFacilityVisible') ? 'acitve' : '']"
	@click.stop="setHearderActiveModule($event, 'targetFacilityVisible')"
	v-menuAuth="'management'">
	目标设施管理
</div>
<div class="home-tittle-bg" ... v-menuAuth="'indicator'">指标体系管理</div>

<!-- 下拉菜单项则直接用 v-if + hasAuth -->
<a-menu-item v-if="hasAuth('user')" @click.stop="setHearderActiveModule($event, 'userInfoManage')">
	用户管理
</a-menu-item>
```

同一个位置能看到两种写法并存：独立入口用指令（禁用置灰），下拉菜单项用 `v-if`（直接不渲染）。因为 Ant Design 的 `a-menu-item` 上叠 disabled 属性和行内样式会与其自身样式打架，`v-if` 反而干净——**指令不是唯一答案，合适的容器用合适的方式**。

### 5.2 第二层：模块内 Tab（hasAuth + 首个可见 Tab 自动选中）

每个模块内部的 Tab 页签也按权限显隐。这里有个容易被忽略的体验问题：**如果默认激活的 Tab 恰好没权限，页面会白屏**。项目的解法是一段 watch，自动选中第一个有权限的 Tab：

```typescript
// src/pages/IndexManagement/index.vue（有删减）
const activeKey = ref<string>('1'); // 1 指标集 2 指标管理 3 模型库 4 规则库 5 节点库

const ALL_TAB_KEYS = ['1', '2', '3', '4', '5'] as const;

// 判断每个 tab 是否可见（必须和模板中的 v-if 条件一致！）
const isTabVisible = (key: string): boolean => {
	switch (key) {
		case '1': return hasAuth('indicator:set');
		case '2': return hasAuth('indicator:system');
		case '3': return hasAuth('indicator:model');
		case '4': return hasAuth('indicator:rules');
		case '5': return hasAuth('indicator:workflow');
		default: return false;
	}
};

// 🔁 监听权限变化（或页面初始化），自动设置 activeKey
watch(
	() => [hasAuth('indicator:set'), hasAuth('indicator:system'), hasAuth('indicator:model'),
	       hasAuth('indicator:rules'), hasAuth('indicator:workflow')],
	() => {
		const firstVisibleKey = ALL_TAB_KEYS.find(key => isTabVisible(key));
		if (firstVisibleKey && !isTabVisible(activeKey.value)) {
			activeKey.value = firstVisibleKey;  // 只在当前 key 不可见时重置，不打扰用户手动切换
		}
	},
	{ immediate: true }
);
```

模板侧就是简单的 `v-if`：

```html
<a-tabs v-model:activeKey="activeKey" @change="handleChange">
	<a-tab-pane key="1" tab="指标集"   v-if="hasAuth('indicator:set')"><indexSetList v-if="activeKey == '1'" /></a-tab-pane>
	<a-tab-pane key="2" tab="指标管理" v-if="hasAuth('indicator:system')"><indexManageTree v-if="activeKey == '2'" /></a-tab-pane>
	<a-tab-pane key="3" tab="模型库"   v-if="hasAuth('indicator:model')"><model-base v-if="activeKey == '3'" /></a-tab-pane>
	<!-- ... -->
</a-tabs>
```

注意内层还套了一层 `v-if="activeKey == '1'"`——Tab 切走时**销毁**内容组件而不是仅隐藏，配合 `onBeforeUnmount` 里 `store.$reset()`，保证每个 Tab 进入时都是干净状态。权限显隐和状态清理是两件事，但经常在同一处代码交织。

这段代码也有个维护性隐患值得一提：`isTabVisible` 里的权限判断**必须手动和模板里的 v-if 保持一致**（代码注释里自己加了感叹号提醒）。权限项增删时要改两处，漏一处就是"页面空白但 Tab 还在"。更稳的做法是把权限码收进 Tab 配置数组，`v-for` 渲染——单一数据源，天然同步。

### 5.3 第三层：按钮（v-auth）

全项目约 60 处使用，模式高度统一：

```html
<a-button v-auth="'plan:add'" type="primary">新增</a-button>
<a-button v-auth="'plan:edit'">编辑</a-button>
<a-button v-auth="'plan:delete'" danger>删除</a-button>
```

卡片操作栏、列表行内操作、详情页按钮，一律 `v-auth="'模块:动作'"`。没有任何页面在**逻辑代码里**判断按钮权限后再调接口——按钮禁用了，入口就断了，这是"指令管显隐、后端管校验"的干净分工。

<a id="sec6"></a>

## 6. 权限的配置面：角色管理与权限树

前面都是消费端，现在看权限从哪配出来。用户管理模块有两个 Tab：用户管理 + 权限管理（实际是角色管理），标准的 RBAC 三件套：用户—角色—权限。

**角色配置弹窗**是配置面的核心：Element Plus 的 `el-tree` 复选树渲染整棵权限树，勾选结果随角色一起保存：

```html
<!-- src/pages/UserManage/components/permissionModel.vue（有删减） -->
<ModuleItem title="权限配置">
	<el-tree
		ref="treeRef"
		:data="treeData"
		show-checkbox
		node-key="permissionId"
		default-expand-all
		:default-checked-keys="formState.permissionIdList"
		:props="treeProps"
	/>
</ModuleItem>
```

```typescript
// 同文件（有删减）
// 权限树来自后端
const getPermissionTree = () => {
	sysPermissionTree().then(res => {
		if (res.code == 200 && res.data) {
			handleTree(res.data);
			treeData.value = res.data;
		}
	});
};

// 提交：勾选的节点 id 集合就是角色的权限清单
const handleOk = () => {
	if (treeRef.value!.getCheckedKeys(false).length == 0) {
		message.error('请选择权限');
		return;
	}
	roleInfoForm.value.validate().then(() => {
		emit('sendRole', {
			...formState,
			permissionIdList: treeRef.value!.getCheckedKeys(false)
		});
	});
};
```

几个实现细节：

- **权限树是后端的单一事实源**。树的结构（`permissionLevel` 1 级/2 级）、命名全部来自 `GET /sysPermission/getPermissionTree`，前端只做展示和回显（编辑时 `getRoleDetail` 带回 `permissionIdList` 填进 `default-checked-keys`）。**前端永远不该硬编码权限清单**——项目里那份 `utils/roleCode.ts` 的权限码 mock 全项目零引用，只当设计文档存在，这是对的位置。
- `node-key="permissionId"` + `getCheckedKeys(false)`：参数 `false` 表示"不含半选父节点"，只提交真正勾中的末级权限，父节点由后端推导，避免"勾了父带出全部子"的歧义数据。
- **用户侧挂角色**：用户新增/编辑弹窗里是角色**多选**（`roleIdList` 必填）+ 部门单选。一个用户多个角色，权限并集——这是 RBAC 的标准语义。

至此闭环完成：管理员在权限树勾出角色 → 用户绑定角色 → 用户登录后 `getPermissionByUser` 把并集权限拍平成两张 Map 下发 → 前端指令/函数消费。

<a id="sec7"></a>

## 7. 数据权限：这一层前端做了什么、该做什么

用户的需求原话是"数据权限"。把这一节留到最后，是因为它需要先辨析概念——**这个项目在前端没有任何数据权限实现，而这是一个合理的架构选择，不是缺失**。

### 7.1 先辨析：三个容易混淆的东西

调查过程中发现的三个"疑似数据权限"，逐一甄别：

| 现象 | 实质 | 是数据权限吗 |
| --- | --- | --- |
| 设施列表的地区级联筛选（国家/省/市/县） | 查询表单的业务条件，**任何用户都能任意选** | ❌ 是筛选，不是权限 |
| 用户资料里的部门字段（departmentId） | 用户的一个**属性**，没有任何接口按它过滤数据 | ❌ 是属性，不是权限 |
| 内置用户 `record.sign` 置灰编辑/删除按钮 | **数据状态保护**（内置数据不可改），与操作者身份无关 | ❌ 是保护，不是权限 |
| 全后端按 token 过滤返回的数据范围 | **真·数据权限**：同一接口，不同 token 拿到不同数据 | ✅ |

一句话区分：**筛选是用户主动选择看什么，权限是系统决定你能看什么**。前者的控制权在用户手里，后者的控制权在身份里。全项目搜索 `dataScope`/`数据范围` 为零结果——数据范围过滤没有前端参与的显式实现，完全由后端在接口层按 token 隐式完成。

### 7.2 为什么"前端零实现"是对的

回到第 0 节的立论就能推出：数据权限如果做在前端（比如前端拿到数据后按部门过滤再展示），等于**把过滤逻辑放在了攻击者可修改的环境里**——改一行 JS 就能看到全量数据。数据权限的**执行**只有一个安全的位置：后端。前端连"参与"都不应该。

这个项目的形态是数据权限最薄的样子：登录拿到 token → 每个请求带着 token → 后端认 token 给数据。前端甚至不知道数据被过滤过。优点是简单、绝对安全；缺点也明显——**前端无从给出"为什么这里没数据"的提示**（用户看到空列表，分不清是没数据还是没权限），以及交互上无法做"只能看自己部门"之类的默认约束。

### 7.3 如果要做得更好：前端的正确角色

主流的增强方案（比如若依的数据权限：按角色配"全部/本部门/本部门及以下/仅本人"的 dataScope）里，前端的参与方式仍然是**辅助而非执行**：

1. **后端把 dataScope 配置随权限接口下发**（比如 `{ dataScope: 'DEPT_BELOW', deptId: 103 }`）；
2. 前端用它做**呈现的礼貌**：查询表单的部门树默认定位到自己部门、禁选越权节点；列表的敏感列（成本、手机号）按权限隐藏；
3. **过滤本身永远在后端**——前端那步只是让 UI 不提供"注定返回空结果"的选项。

判断一个"前端数据权限"实现是否合格，就一条标准：**把它整段删掉，系统安全性不下降**。删掉后只是体验变差（能选到无权看的选项、发出注定失败的请求），数据一条都不该多出来。

<a id="sec8"></a>

## 8. 短板与改进清单

复盘不能只讲好的。按风险排序：

1. **401 处理不彻底**。响应拦截器里 `router.replace('/login')` 之后直接 reject，**没有清掉 localStorage 里的 token/btnObj/menuObj**。用户被踢回登录页后重新登录另一个账号，期间带着旧权限快照渲染了一屏；更糟的是如果 token 过期但快照还在，部分页面会出现"按钮都在、点击全 401"的灵异状态。改进：401 分支先 `localStorage.clear()` 再跳转，登录页 `onMounted` 顺手清一次。

2. **路由守卫缺登录态校验**。未登录直达 `/#/home` 会看到一屏禁用按钮 + 一串 401 报错。改进只需三行：

	```typescript
	router.beforeEach(to => {
		setTitle(to);
		if (to.path !== '/login' && !localStorage.getItem('token')) {
			return '/login';
		}
	});
	```

3. **权限快照无时效机制**。权限存在 localStorage，会话内不刷新。管理员中途改了某用户权限，该用户不重新登录就不生效（系统配置文件里那句"改了配置必须清空浏览器 localStorage"的注释是同一类问题）。改进：快照加时间戳，超时重拉；或后端权限变更时通过已有的 WebSocket 通道推一个 `permission-changed` 事件强制刷新。

4. **明文密码回显**。用户编辑弹窗的详情回显里带着 `cleartextPassword` 字段并填进表单——详情接口不该返回明文密码，编辑时留空表示"不修改"即可。同类问题：登录表单预填了演示账号密码，上线前应移除。

5. **Tab 权限判断双写**。`isTabVisible` 与模板 `v-if` 人工保持同步（5.2 节），应收敛为 Tab 配置数组单一数据源。

这些短板有一个共同背景：它们全部**不影响安全性**（后端兜底闭环了），只影响体验和可维护性——这反过来印证了第 0 节立论的正确性：前端权限做错了顶多难用，不会不安全；后端权限做错了才是事故。

<a id="sec9"></a>

## 9. 写在最后

| 需求 | 方案 | 关键文件 |
| --- | --- | --- |
| 登录凭证 | login → token → localStorage，axios 注入 `Bearer` | `store/user.ts`、`service/service.ts` |
| 权限下发 | `getPermissionByUser` 靠 token 识别，两张 Map 拍平下发 | `api/userApi.ts` |
| 权限存储 | localStorage 快照（指令在 setup 外同步消费） | `store/user.ts` |
| 权限模型 | `模块:动作` 权限码，按钮/菜单两张独立 Map | `utils/roleCode.ts`（文档） |
| 按钮显隐 | `v-auth` 指令 + 禁用三件套 | `directive/index.ts` |
| 模块入口 | `v-menuAuth` 指令 / `hasAuth` + v-if | `home-header/index.vue` |
| Tab 显隐 | `hasAuth` + v-if + 首个可见 Tab 自动选中 | `IndexManagement/index.vue` |
| 权限配置 | 角色绑定权限树（el-tree 勾选 → permissionIdList） | `UserManage/components/permissionModel.vue` |
| 数据权限 | 前端零实现，后端按 token 过滤（正确位置） | — |

**一句话记住**：前端权限是一套"体验协议"——用 `模块:动作` 的权限码约定语言，用指令和函数做显隐，用 localStorage 存快照；而数据权限的执行只有一个正确位置，是后端。前端在这一层最好的动作是"呈现的礼貌"，不是"过滤的执行"。

### 文件地图

```text
infrastructure-data-plus/src/
├── main.ts                           # authDirective(app) 注册双指令
├── router/index.ts                   # 路由守卫（仅 setTitle，无鉴权）
├── directive/index.ts                # ★ 权限核心：hasBtnAuth / hasAuth / v-auth / v-menuAuth
├── store/user.ts                     # 登录链路：token + 权限快照落 localStorage
├── service/service.ts                # axios 拦截器：Bearer 注入 + 401/601 踢回登录
├── api/userApi.ts                    # 登录/用户/角色/权限树全部接口
├── utils/roleCode.ts                 # 权限码清单 mock（零引用，当设计文档）
└── pages/
    ├── login/index.vue               # 登录表单
    ├── home/components/home-header/  # 模块入口 v-menuAuth / 菜单项 hasAuth
    ├── IndexManagement/index.vue     # Tab 权限 + 首个可见 Tab 自动选中
    ├── AttackMeans/index.vue         # 同款 Tab 权限方案
    ├── TargetFacilityManagement/     # v-auth 按钮权限（facility:*）
    └── UserManage/
        ├── index.vue                 # 用户管理 + 权限管理双 Tab
        ├── components/userList.vue       # 用户列表（角色筛选/启停）
        ├── components/userInfoModel.vue  # 用户新增编辑（角色多选 + 部门）
        ├── components/permissionList.vue # 角色列表
        └── components/permissionModel.vue # ★ 角色权限树配置（el-tree）
```

### 参考

- [vue-element-admin 权限方案（动态路由/指令的先驱实践）](https://panjiachen.github.io/vue-element-admin-site/zh/guide/essentials/permission.html)
- [Vue 3 官方文档 - 自定义指令](https://cn.vuejs.org/guide/reusability/custom-directives.html)
- [Pinia 官方文档](https://pinia.vuejs.org/zh/)
- [RBAC 模型：NIST/维基百科](https://en.wikipedia.org/wiki/Role-based_access_control)
