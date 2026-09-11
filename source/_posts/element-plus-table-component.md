---
title: 封装一个 Element Plus 通用表格组件：配置驱动、插槽透传与单选模拟
date: 2026-09-09 21:30:00
description: 中后台页面一半代码是 el-table-column 和 el-pagination 的复制粘贴。把表格封装成 data/header/config 三个 prop 的配置驱动组件：列用 JSON 声明、自定义单元格走插槽透传、跨页多选靠 reserve-selection、单选用多选模拟、分页内置并统一 pageChange 事件，最后附完整代码与设计复盘。
categories:
  - Vue 进阶
tags:
  - Vue3
  - ElementPlus
  - Typescript
---

写过中后台的同学都有体感：列表页的模板代码一半是 `<el-table-column>` 的复制粘贴，另一半是 `<el-pagination>` 的事件处理。一个页面十几列，`loading`、空状态、序号列、多选框、操作列插槽……每个页面把这些再写一遍，改一个列宽要去模板里翻半天。

之前项目里沉淀了一个基于 Element Plus 的通用表格组件，最近又在新项目里用一个「后端下发表头」的动态列场景喂了它一波，正好把完整代码和设计思路整理出来。组件不大，两百多行，但里面有几个值得抠的设计点：**配置驱动、插槽透传、单选模拟、跨页多选、分页状态归属**。

<!-- more -->

## 一、API 设计：三个 prop、三个事件、一个 expose

先看组件对外的契约，封装一个组件最重要的就是先把 API 定稳：

```text
props
  data     : any[]          行数据
  header   : HeaderItem[]   列配置（JSON 声明，见下表）
  config   : TableConfig    表格行为配置（loading/total/多选/序号…）

emits
  pageChange      { pageNum, pageSize }   翻页或改页大小（内置分页）
  sortChange      { prop, order }         排序（已归一化为 asc/desc/''）
  selectionChange rows                    多选/单选变化

expose
  pageReset()     分页重置回第一页并触发 pageChange
  tableRefs       el-table 实例引用（父组件需要时能直接调原生方法）
```

`header` 数组每一项的字段：

| 字段 | 作用 | 备注 |
| --- | --- | --- |
| `key` / `title` | 字段名 / 列标题 | 对应 el-table-column 的 `prop` / `label` |
| `isCheck` | 是否显示该列 | computed 里过滤掉 `false` 的列 |
| `colWidth` / `minWidth` | 固定宽 / 最小宽 | 二选一 |
| `fixed` | 固定列 | `'left'` / `'right'`，操作列常用 |
| `sort` | 是否可排序 | 透传 `sortable` |
| `slot` | 自定义单元格插槽名 | 命中则该列渲染具名插槽 |
| `headerSlot` | 自定义表头插槽名 | 同上，渲染在 `#header` |
| `formatter` | 格式化函数 | 直接透传 el-table-column 的 `formatter` |

`config` 的字段：`loading`、`total`、`rowKey`、`isSelection`（多选列）、`isSerialNo`（序号列）、`isRadio`（单选模式）、`isBorder`、`noPagination`（关掉分页）。

这个设计的目标是：**常规页面零模板代码**——列声明在 JS 里，数据驱动渲染；需要自定义的格子通过插槽escape hatch出去，不会把组件做成大杂烩。

## 二、模板：配置驱动的列渲染 + 插槽透传

完整模板（Vue 3 `<script setup>` + Element Plus）：

```html
<template>
  <div class="table-container">
    <el-table
      :data="data"
      :border="setBorder"
      v-bind="$attrs"
      :row-key="config?.rowKey || 'id'"
      stripe
      style="width: 100%"
      v-loading="config.loading"
      @selection-change="onSelectionChange"
      @sort-change="sortChange"
      ref="tableRefs"
      :class="[config?.isRadio ? 'show-radio' : '']"
    >
      <el-table-column
        type="selection"
        :reserve-selection="true"
        align="center"
        width="40"
        v-if="config.isSelection"
      />
      <el-table-column
        type="index"
        label="序号"
        align="center"
        width="60"
        v-if="config.isSerialNo"
      />
      <el-table-column
        v-for="(item, index) in setHeader"
        :key="index"
        :show-overflow-tooltip="item.type === 'image' ? false : true"
        :prop="item.key"
        :width="item.colWidth"
        :min-width="item.minWidth"
        :label="item.title"
        :fixed="item.fixed"
        :sortable="item.sort"
        :formatter="item.formatter"
        align="center"
      >
        <template v-if="item.headerSlot" #header>
          <slot :name="item.headerSlot"></slot>
        </template>
        <template v-if="item.slot" v-slot="scope">
          <slot :name="item.slot" v-bind="scope"></slot>
        </template>
      </el-table-column>
      <template #empty>
        <el-empty description="暂无数据" />
      </template>
    </el-table>
    <div class="table-footer mt15" v-if="!config.noPagination">
      <el-pagination
        v-model:current-page="state.page.pageNum"
        v-model:page-size="state.page.pageSize"
        :pager-count="5"
        :page-sizes="[10, 20, 30]"
        :total="config.total"
        layout="total, sizes, prev, pager, next, jumper"
        background
        @size-change="onHandleSizeChange"
        @current-change="onHandleCurrentChange"
      >
      </el-pagination>
    </div>
  </div>
</template>
```

四个值得停下来的点：

**1. `v-bind="$attrs"` 保留原生能力。** 封装最容易翻车的是「包了一层，原生 API 全丢了」——父组件想设 `height`、想监听 `row-click` 怎么办？`$attrs` 一行透传，el-table 的所有 props 和事件照常可用。组件不是墙，是门。

**2. 多选列的 `:reserve-selection="true"`。** 默认情况下翻页会丢失已勾选的行——el-table 的多选是「当前页多选」。开启 reserve-selection 后配合 `row-key`，跨页勾选会累积保留。这是多选场景最容易漏的配置。

**3. 插槽透传的动态具名插槽。** 列配置里声明 `slot: 'action'`，模板里 `<slot :name="item.slot" v-bind="scope">` 把作用域插槽转发出去——父组件用普通的 `<template #action="scope">` 就能自定义任意格子。**配置声明「哪列要自定义」，插槽决定「怎么自定义」**，两层的关注点是分开的。`v-bind="scope"` 把 `{ row, column, $index }` 原样带出去，父组件拿到的和直接写 el-table-column 一样。

**4. 空状态统一。** `<template #empty>` 里放 `el-empty`，所有页面「暂无数据」的视觉一致，不用每个页面各配一次。

## 三、脚本：单选模拟、排序归一化、分页状态归属

```ts
<script setup lang="ts">
import { reactive, computed, nextTick, ref, watch } from 'vue'

declare type EmptyObjectType<T = any> = { [key: string]: T }
declare type RefType<T = any> = T | null

const props = defineProps({
  data:   { type: Array<EmptyObjectType>, default: () => [] },
  header: { type: Array<EmptyObjectType>, default: () => [] },
  config: { type: Object, default: () => {} },
  printName: { type: String, default: () => '' },
})

const emit = defineEmits(['pageChange', 'sortChange', 'selectionChange'])

const tableRefs = ref<RefType>(null)
const state = reactive({
  page: { pageNum: 1, pageSize: 10 },
  selectlist: [] as EmptyObjectType[],
})

// 外部把 clearSelection 置 true → 清空勾选（布尔脉冲）
watch(
  () => props.config?.clearSelection,
  (val) => {
    if (val === true) {
      nextTick(() => {
        tableRefs.value?.clearSelection()
      })
    }
  },
)

// 排序：把 el-table 的 descending/ascending 归一化成后端约定的 desc/asc
const sortChange = (data: { column: any; prop: string; order: any }) => {
  switch (data?.order) {
    case 'descending': data.order = 'desc'; break
    case 'ascending':  data.order = 'asc';  break
    default:           data.order = ''
  }
  emit('sortChange', data)
}

const setBorder = computed(() => !!props.config.isBorder)
const setHeader = computed(() => props.header.filter((v) => v.isCheck))

const cur = ref({})
// 多选回调 / 单选模式：用多选模拟
const onSelectionChange = (val: EmptyObjectType[]) => {
  if (props.config?.isRadio) {
    cur.value = val[val.length - 1]
    val.length > 1 && tableRefs.value.toggleRowSelection(val[0])  // 取消上一行
    state.selectlist = val.length > 1 ? [val[1]] : val
    emit('selectionChange', state.selectlist)
  } else {
    state.selectlist = val
    emit('selectionChange', val)
  }
}

// 分页
const onHandleSizeChange = (val: number) => {
  state.page.pageSize = val
  emit('pageChange', state.page)
}
const onHandleCurrentChange = (val: number) => {
  state.page.pageNum = val
  emit('pageChange', state.page)
}
// 搜索时：分页还原成第一页
const pageReset = () => {
  state.page.pageNum = 1
  state.page.pageSize = 10
  emit('pageChange', state.page)
}

defineExpose({ pageReset, tableRefs })
</script>
```

### 单选模拟：Element Plus 没有 radio 列

Element Plus（截至 2.x）没有内置单选列。社区通用解法就是**用多选列模拟**，配合一段 CSS 把表头的全选框藏掉：

```ts
if (props.config?.isRadio) {
  // 新选中的行永远是 val 的最后一项
  cur.value = val[val.length - 1]
  // 勾选了第二行时，把上一行的勾选状态取消
  val.length > 1 && tableRefs.value.toggleRowSelection(val[0])
  state.selectlist = val.length > 1 ? [val[1]] : val
}
```

```scss
.show-radio {
  // 单选模式下隐藏表头的全选框
  :deep(.el-table__header .el-checkbox) {
    display: none;
  }
}
```

这个 hack 的巧处在 `selection-change` 回调的时序：用户每次点击一行，新勾选的行**追加在数组末尾**，于是「数组长度大于 1」意味着「切换了选择」，把 `val[0]`（旧行）取消勾选即可。看起来别扭，但它完全建立在 el-table 的公开行为上，比自己造一套 radio 列（要处理高亮、行点击、数据回显）省得多。

### 排序归一化：边界翻译收在组件里

el-table 的 `sort-change` 事件给的是 `descending` / `ascending`，而绝大多数后端接口要的是 `desc` / `asc`。这种「两个世界的方言差异」就该由封装层翻译——组件的使用者拿到的永远是能直接拼进请求参数的值，每个页面不用再写一遍 switch。

### 分页状态归属：一个反直觉的取舍

注意分页的 `pageNum/pageSize` 放在**组件内部的 state** 里，而不是 props。这意味着：

- 父组件不需要维护一份分页状态，翻页只管接 `pageChange` 事件、带着新页码发请求；
- 但父组件**不能**直接改页码——搜索后想回第一页，得调 `defineExpose` 出去的 `pageReset()`。

这是「受控/非受控」的经典权衡。选择非受控是因为列表页的真实数据流是单向的：翻页事件 → 改请求参数 → 拉数据 → 回填 `data`/`total`。分页状态留在组件内，父组件的 `param.pageNum` 只作为**请求参数的快照**存在，两边不会打架。代价是像「删除最后一页最后一条后自动前翻一页」这种场景，父组件只能间接操作——这是我会记在下次改进清单上的一条。

`watch(clearSelection)` 也是个有意思的约定：外部把 `config.clearSelection` 置 `true`，组件在 `nextTick` 后清空勾选。因为 `true → true` 不触发 watch，实际用的时候外部得先切回 `false` 再置 `true`，形成一个「布尔脉冲」——能用，但语义不如暴露一个 `clearSelection()` 方法直白，同样是改进项。

## 四、实战：动态表头 + 操作列插槽

实际项目里一个列表页用起来是这样的（以设施列表页为例，节选）：

```html
<Table ref="tableRef" v-bind="state.tableData" @pageChange="onTablePageChange">
  <template #action="scope">
    <el-button link type="primary" size="small" @click="openEditDialog(scope.row)">
      编辑
    </el-button>
    <el-popconfirm title="此操作将永久删除该设施，是否继续?" @confirm="handleDelete(scope.row)">
      <template #reference>
        <el-button link type="danger" size="small">删除</el-button>
      </template>
    </el-popconfirm>
  </template>
</Table>
```

```ts
const state = reactive({
  tableData: {
    data: [] as any[],
    header: [] as any[],
    config: {
      total: 0,
      loading: false,
      rowKey: 'id',
      isSerialNo: true,     // 序号列
      noPagination: false,
    },
    param: { pageNum: 1, pageSize: 10, keyword: '' },
  },
})

const getTableData = () => {
  state.tableData.config.loading = true
  getFacilityList({
    ...state.tableData.param,
    pageNum: state.tableData.param.pageNum,
    pageSize: state.tableData.param.pageSize,
  }).then((result: any) => {
    // 亮点：表头由后端下发（dataIndex/title），前端只补操作列
    const head: any[] = result.head || []
    state.tableData.header = [
      ...head.map((col: any) => ({
        key: col.dataIndex || col.key,
        title: col.title || col.label,
        isCheck: col.isCheck !== false,
        minWidth: col.minWidth || col.width || 120,
        slot: col.slot || undefined,
      })),
      { key: 'action', colWidth: 180, title: '操作', slot: 'action', fixed: 'right', isCheck: true },
    ]
    state.tableData.data = result.data || []
    state.tableData.config.total = Number(result.total) || 0
  }).finally(() => {
    state.tableData.config.loading = false
  })
}

function onSearch() {
  tableRef.value?.pageReset()   // 搜索 = 回第一页重新拉取
}

const onTablePageChange = (page: { pageNum: number; pageSize: number }) => {
  state.tableData.param.pageNum = page.pageNum
  state.tableData.param.pageSize = page.pageSize
  getTableData()
}
```

两个实践模式：

1. **`v-bind="state.tableData"` 一把传三 prop。** `data/header/config` 打包在一个 reactive 对象里，模板上只有一行——「这一个对象就是这张表的全部状态」，查问题的时候只看一个地方；
2. **动态表头是配置驱动的隐藏福利。** 因为列本来就是 JSON，后端下发 `head` 数组时前端只需做一层字段名归一化（`dataIndex → key`）再拼上固定的操作列。「表格列配置」从模板代码变成了**数据**，可以从接口来、可以存用户偏好、可以做列显隐设置——这些都是写死 `<el-table-column>` 时不敢想的。

这个组件在项目里被用户、角色、设施三个管理页复用。当前页面主要用到了序号列、分页和插槽；多选、单选、排序是按通用中后台场景预置的能力，在别的项目里已经过实战，这里一并保留。

## 五、设计复盘：好的与该改的

| 维度 | 做法 | 评价 |
| --- | --- | --- |
| 原生能力 | `v-bind="$attrs"` 透传 | ✅ 封装不阉割，el-table 全量 API 可用 |
| 自定义格子 | 列配置声明 slot + 动态插槽转发 | ✅ 配置管「哪列」，插槽管「怎么渲染」 |
| 跨页多选 | `reserve-selection` + `row-key` | ✅ 一行配置解决翻页丢勾选 |
| 单选 | 多选模拟 + CSS 藏表头全选框 | ⚠️ 可用，但依赖 selection 数组时序，升级 Element Plus 要回归测试 |
| 排序 | descending → desc 归一化 | ✅ 方言翻译收进组件 |
| 分页 | 非受控 + `pageReset()` 暴露 | ⚠️ 搜索回第一页顺手，但父组件无法直接控制页码 |
| 清空勾选 | watch 布尔脉冲 | ❌ 应该直接 expose 一个 `clearSelection()` 方法 |
| 类型 | `EmptyObjectType`、config 无类型 | ❌ 该用泛型 `TableProps<T>` + `HeaderItem` 接口约束，IDE 提示和防错都缺 |

复盘里最后两条是诚实账：`config` 是个无类型 Object，字段拼错不会有任何提示，纯靠文档和人肉记忆。理想形态是导出 `HeaderItem`、`TableConfig` 接口并让 `data` 支持泛型推导——列少的时候无所谓，列多、多人协作时类型就是第一生产力。

## 总结

```text
封装前（每页）                        封装后（每页）
─────────────────                    ─────────────────
<el-table> + loading     模板 ~60 行   <Table v-bind="tableData">  1 行
<el-table-column> ×N                  + 需要自定义的具名插槽
<el-pagination> + 2 个事件            header: JSON 声明列
空状态 / 序号列 / 多选列               翻页只接 pageChange 一个事件
```

**一句话记住**：表格组件封装的本质不是「包一层」，而是把**稳定的骨架**（列渲染、分页、选择、空态）收进组件，把**易变的血肉**（哪些列、列怎么自定义、数据从哪来）留给配置和插槽——判断封装好坏的标准就一条：用的人还有没有机会碰到 el-table 本身解决不了的问题（`$attrs` 和 `defineExpose` 就是留给这个的）。
