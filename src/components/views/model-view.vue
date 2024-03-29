<template>
    <!--表格头部-->
    <div class="flex justify-between">
        <div class="flex items-center space-x-2">
            <el-button v-if="showAddButton" @click="onAdd" type="primary" :icon="Plus">新增</el-button>
            <el-button v-if="showRefreshButton" @click="onRefresh" :icon="Refresh" :loading="status.status.refresh">刷新</el-button>
        </div>

        <div v-if="hasSearch" class="search-section flex items-center space-x-2">
            <el-form :model="searchData" inline>
                <slot name="search-form-items" :data="searchData"></slot>
                <el-form-item>
                    <el-button @click="search" type="primary">搜索</el-button>
                </el-form-item>
            </el-form>
        </div>
    </div>

    <!--表格-->
    <el-table :data="tableData" border class="mt-2" default-expand-all row-key="id" :indent="15" v-loading="status.status.refresh">
        <el-table-column v-if="showSelectionColumn" type="selection" width="50" align="center"></el-table-column>
        <el-table-column v-if="showIndexColumn" type="index" label="序号" width="60" align="center"></el-table-column>
        <el-table-column v-if="showIdColumn" type="id" label="ID" prop="id" width="80" align="center"></el-table-column>
        <el-table-column v-if="showNameColumn" label="名称" prop="name" show-overflow-tooltip></el-table-column>

        <slot name="table-columns"></slot>

        <input-number-column v-if="showSortColumn" label="排列序号" prop="sort"></input-number-column>

        <el-table-column v-if="showCreatedAtColumn" label="创建时间" prop="created_at" align="center" width="170"></el-table-column>
        <el-table-column v-if="showUpdatedAtColumn" label="修改时间" prop="updated_at" align="center" width="170"></el-table-column>

        <switch-column v-if="showStatusColumn" label="状态" prop="status"></switch-column>

        <el-table-column v-if="showActionColumn" label="操作" align="center" :width="actionColumnWidth">
            <template #default="scope">
                <div class="flex items-center justify-center space-x-1">
                    <el-button @click="onView(scope.row)" type="primary" size="small">查看</el-button>
                    <el-button @click="onEdit(scope.row)" type="primary" size="small">修改</el-button>
                    <el-popconfirm @confirm="onDelete(scope)" title="删除后无法恢复，确定要删除吗？" cancel-button-type="default">
                        <template #reference>
                            <el-button :loading="cellStatus(scope,'delete')" type="danger" size="small">删除</el-button>
                        </template>
                    </el-popconfirm>
                    <slot name="action-buttons" :row="scope.row"></slot>
                </div>
            </template>
        </el-table-column>
    </el-table>

    <!--分页-->
    <div v-if="!isTree" class="mt-2 px-2 h-11 bg-gray-50 flex items-center justify-between">
        <el-pagination v-if="pageable" background small layout="total, prev, pager, next" v-model:current-page="lister.page" :total="lister.total"/>
        <el-pagination v-else background small layout="total" :total="lister.total"/>
    </div>

    <!--编辑对话框-->
    <extend-dialog
        v-model="editDialog.visible"
        @close="onEditDialogClosed"
        :title="editDialog.title"
        :close-on-click-modal="false"
        :loading="status.status.show||status.status[editDialog.label]"
    >
        <el-form ref="form" :model="formData" label-position="top" v-bind="formAttributes" @keyup.enter="onSubmit">
            <slot name="form-items" :data="formData" :list="tableData" :isEdit="isEdit"></slot>
        </el-form>

        <template #actions>
            <el-button @click="onCancel">取消</el-button>
            <el-button @click="onSubmit" type="primary" :loading="status.status.submit">提交</el-button>
        </template>
    </extend-dialog>

    <!--查看对话框-->
    <extend-dialog
        v-model="viewDialog.visible"
        :title="viewDialog.title"
        :close-on-click-modal="false"
        :loading="status.status.show||status.status[viewDialog.label]"
    >

        <el-tabs type="card" v-model="viewDataTab">
            <el-tab-pane label="常规" name="common">
                <el-descriptions :column="1" border>
                    <el-descriptions-item label="ID" label-class-name="w-48">{{ viewData.id }}</el-descriptions-item>
                    <el-descriptions-item v-if="viewData.name!==undefined" label="名称">{{ viewData.name }}</el-descriptions-item>

                    <slot name="view-items" :data="viewData"></slot>

                    <el-descriptions-item label="排列序号" v-if="viewData.sort!==undefined">{{ viewData.sort }}</el-descriptions-item>
                    <el-descriptions-item label="状态" v-if="viewData.status!==undefined">
                        <el-tag v-if="viewData.status" type="success" size="small">正常</el-tag>
                        <el-tag v-else type="danger" size="small">禁用</el-tag>
                    </el-descriptions-item>
                    <el-descriptions-item label="创建时间">{{ viewData.created_at }}</el-descriptions-item>
                    <el-descriptions-item label="更新时间">{{ viewData.updated_at }}</el-descriptions-item>
                </el-descriptions>
            </el-tab-pane>
            <el-tab-pane label="全部" name="all">
                <extend-descriptions :data="viewData"></extend-descriptions>
            </el-tab-pane>
        </el-tabs>

        <template #actions>
            <el-button @click="closeViewDialog">关闭</el-button>
        </template>
    </extend-dialog>
</template>

<script setup>
import {Plus, Refresh} from "@icon-park/vue-next";
import {computed, inject, provide, reactive, ref, useSlots, watch} from "vue";
import {useStatus} from "../../states/status";
import ExtendDialog from "../extend/extend-dialog.vue";
import {warning} from "../../utils/message";
import ExtendDescriptions from "../extend/extend-descriptions.vue";
import SwitchColumn from "../columns/switch-column.vue";
import InputNumberColumn from "../columns/input-number-column.vue";
import api from "../../utils/api";

// 请求状态
const status = useStatus()

// 组件属性
const props = defineProps({
    // 接口资源名
    resource: {
        type: String,
        required: true,
    },

    // 是否显示新增按钮
    showAddButton: {
        type: Boolean,
        default: true,
    },

    // 是否显示刷新按钮
    showRefreshButton: {
        type: Boolean,
        default: true,
    },

    // 是否分页
    pageable: Boolean,

    // 分页大小
    pageSize: {
        type: Number,
        default: 1,
    },

    // 是否树型表格
    isTree: Boolean,

    // 表格属性
    tableAttributes: {
        type: Object,
        default: () => ({}),
    },

    // 是否显示选择列
    showSelectionColumn: Boolean,

    // 是否显示索引列
    showIndexColumn: Boolean,

    // 是否显示 ID 列
    showIdColumn: {
        type: Boolean,
        default: true,
    },

    // 是否显示名称列
    showNameColumn: {
        type: Boolean,
        default: true,
    },

    // 是否显示排列序号列
    showSortColumn: Boolean,

    // 是否显示创建时间列
    showCreatedAtColumn: Boolean,

    // 是否显示更新时间列
    showUpdatedAtColumn: Boolean,

    // 是否显示状态列
    showStatusColumn: Boolean,

    // 是否显示操作列
    showActionColumn: {
        type: Boolean,
        default: true,
    },

    // 是否显示查看按钮
    showViewButton: {
        type: Boolean,
        default: true,
    },

    // 是否显示修改按钮
    showEditButton: {
        type: Boolean,
        default: true,
    },

    // 是否显示删除按钮
    showDeleteButton: {
        type: Boolean,
        default: true,
    },

    // 操作列宽度
    actionColumnWidth: {
        type: [String, Number],
        default: 200,
    },

    // 编辑表单默认数据
    formDefaults: {
        type: Object,
        default: () => ({})
    },

    // 编辑表单属性
    formAttributes: {
        type: Object,
        default: () => ({})
    },

    // 修改时是否刷新数据
    refreshEditData: Boolean,

    // 查看时是否刷新数据
    refreshViewData: Boolean,
})

// 解构一些常用属性
const {
    resource,
    pageable,
    pageSize,
    formDefaults,
    refreshEditData,
    refreshViewData,
} = props

// 当前模型的 api 接口
const apis = api(resource)

// 事件
const emits = defineEmits(['onEditDialogOpen', 'onViewDialogOpen'])

// 列表
const lister = ref(pageable ? {page: 1, size: pageSize, data: [], total: 0} : {page: 0, data: [], total: 0})

// 把数组转换成列表
const toLister = lister => lister.page === undefined ? {page: 0, data: lister, total: lister.length} : lister

// 表格数据
const tableData = computed(() => lister.value.data)

// 表格单元格请求标记
const cellLabel = (scope, action) => [scope.column.id, scope.$index, action].join('_')

// 表格单元格请求状态
const cellStatus = (scope, action) => status.status[cellLabel(scope, action)]

// 列表数据刷新后执行
const afterRefresh = inject('afterRefresh', null)

// 编辑表单赋值前执行
const beforeEdit = inject('beforeEdit', null)

// 编辑表单提交前执行
const beforeSubmit = inject('beforeSubmit', null)

// 对一个值执行回调函数
const callClosure = (value, closure) => typeof closure === 'function' ? closure(value) : value

// 刷新列表数据
const refresh = (params = {}) => {
    let config = {
        label: 'refresh',
    }

    config.params = params

    if (pageable) {
        config.params.page = lister.value.page
        config.params.size = lister.value.size
    }

    apis.list(config).then(res => {
        lister.value = callClosure(toLister(res), afterRefresh)
    })
}

// 点击刷新按钮
const onRefresh = () => refresh()

// 更新数据
const update = (scope, action) => {
    apis.update(scope.row, cellLabel(scope, action)).then(() => {
        refresh()
    })
}

// 删除数据
const destroy = (scope, action) => {
    apis.destroy(scope.row, cellLabel(scope, action)).then(() => {
        refresh()
    })
}

// 编辑对话框参数
const editDialog = reactive({
    visible: false,
    width: 30,
    top: 15,
    fullscreen: false,
    title: '编辑',
    label: 'editDialog'
})

// 当前编辑表单数据
const formData = ref({})

// 是否编辑模式
const isEdit = computed(() => !!formData.value['id'])

// 编辑表单引用
const form = ref()

// 提交表单数据
const onSubmit = () => {
    form.value.validate(valid => {
        if (valid) {
            let data = callClosure(formData.value, beforeSubmit)

            const request = isEdit.value
                ? apis.update(data, 'submit')
                : apis.store(data, 'submit')

            request.then(res => {
                closeEditDialog()
                refresh()
            })
        } else {
            warning('表单验证未通过，请重新填写')
        }
    })
}

// 显示编辑对话框
const showEditDialog = (title) => {
    editDialog.title = title
    editDialog.visible = true
    emits('onEditDialogOpen', isEdit.value)
}

// 关闭编辑对话框
const closeEditDialog = () => {
    editDialog.title = ''
    editDialog.visible = false
}

// 编辑对话框关闭
const onEditDialogClosed = () => {
    form.value.clearValidate()
}

// 编辑对话框加载标记
const editDialogLabel = () => editDialog.label

// 点击取消按钮
const onCancel = () => {
    editDialog.visible = false
}

// 点击新增按钮
const onAdd = () => {
    formData.value = callClosure(JSON.parse(JSON.stringify(formDefaults)), beforeEdit)
    showEditDialog('新增')
}

// 点击修改按钮
const onEdit = (row) => {
    formData.value = callClosure(JSON.parse(JSON.stringify(row)), beforeEdit)
    if (refreshEditData) {
        apis.show(row, {label: 'show', message: false}).then(res => {
            formData.value = callClosure(res, beforeEdit)
        })
    }
    showEditDialog('编辑')
}

// 所有插槽
const slots = useSlots()

// 是否有搜索插槽
const hasSearch = computed(() => slots['search-form-items'] !== undefined)

// 搜索关键词数据
const searchData = ref({})

// 是否含有搜索数据
const hasSearchData = computed(() => Object.values(searchData.value).filter(item => item !== '').length > 0)

// 搜索请求
const search = () => {
    if (hasSearchData.value) {
        refresh(searchData.value)
    } else {
        warning('请输入搜索条件')
    }
}

// 查看对话框参数
const viewDialog = reactive({
    visible: false,
    width: 30,
    top: 15,
    fullscreen: false,
    title: '查看详情',
    label: 'viewDialog',
})

// 打开查看对话框
const showViewDialog = () => {
    viewDialog.visible = true
    emits('onViewDialogOpen')
}

// 关闭查看对话框
const closeViewDialog = () => viewDialog.visible = false

// 编辑对话框加载标记
const viewDialogLabel = () => viewDialog.label

// 查看详情数据
const viewData = ref({})

// 查看详情选项卡
const viewDataTab = ref('common')

// 点击查看按钮
const onView = row => {
    viewData.value = row
    if (refreshViewData) {
        apis.show(row, {label: 'show', message: false}).then(res => {
            viewData.value = res
        })
    }
    showViewDialog()
}

// 点击删除按钮
const onDelete = scope => {
    destroy(scope, 'delete')
}

// 初始化列表数据
refresh()

// 监听页码变化
if (pageable) {
    watch(() => lister.value.page, page => refresh())
}

// 常用方法供给子组件
provide('refresh', refresh)
provide('update', update)
provide('cellLabel', cellLabel)
provide('cellStatus', cellStatus)

// 常用方法暴露给父组件
defineExpose({
    refresh,
    update,
    cellLabel,
    cellStatus,
    editDialogLabel,
    viewDialogLabel,
})

</script>

<style scoped>

</style>
