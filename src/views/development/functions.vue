<template>
  <div class="app-container">
    <!-- 数据检索区 -->
    <div class="filter-container">
      <el-input
        v-model="listQuery.keyword"
        placeholder="搜索"
        style="width: 200px;margin-right: 10px;"
        class="filter-item"
        @keyup.enter.native="handleFilter"
      />
      <el-button class="filter-item" type="primary" icon="el-icon-search" @click="handleFilter">
        搜索
      </el-button>
      <el-button
        :loading="downloadLoading"
        class="filter-item"
        type="primary"
        icon="el-icon-download"
        @click="handleDownload"
      >
        导出
      </el-button>
      <el-button v-permission="'function.create'" class="filter-item" type="primary" icon="el-icon-search" @click="showCreateForm">
        新建函数
      </el-button>
    </div>

    <!-- 标签类别 -->

    <div class="tag-selector">
      <div class="label">标签类别</div>
      <el-radio-group v-model="listQuery.tag" size="mini" @change="getList">
        <el-radio-button :label="''" border>全部</el-radio-button>
        <el-radio-button v-for="tag in all_tags" :key="tag" :label="tag" border>{{ tag }}</el-radio-button>
      </el-radio-group>
    </div>

    <!-- 表格 -->
    <el-table
      :key="tableKey"
      v-loading="listLoading"
      :data="list"
      border
      fit
      highlight-current-row
      style="width: 100%;"
    >
      <el-table-column
        label="ID"
        prop="_id"
        align="center"
        min-width="100"
      >
        <template slot-scope="{row}">
          <span>{{ row._id }}</span>
        </template>
      </el-table-column>
      <el-table-column label="函数标识" min-width="100">
        <template slot-scope="{row}">
          <div class="func-name">
            {{ row.name }}
          </div>
        </template>
      </el-table-column>
      <el-table-column label="函数标题" min-width="100px">
        <template slot-scope="{row}">
          <span class="link-type" @click="showUpdateForm(row)">{{ row.label }}</span>
        </template>
      </el-table-column>
      <el-table-column label="标签" width="150">
        <template slot-scope="{row}">
          <el-tag v-for="tag in row.tags" :key="tag" style="margin-right: 6px;" type="primary" size="mini">{{ tag }}</el-tag>
        </template>
      </el-table-column>
      <el-table-column label="创建/更新时间" width="150px" align="center">
        <template slot-scope="{row}">
          <span v-if="row.created_at">{{ row.created_at | parseTime('{y}-{m}-{d} {h}:{i}') }}</span>
          <span v-else>-</span>
          <br>
          <span v-if="row.updated_at">{{ row.updated_at | parseTime('{y}-{m}-{d} {h}:{i}') }}</span>
          <span v-else>-</span>
        </template>
      </el-table-column>
      <el-table-column label="HTTP访问" class-name="status-col" width="100">
        <template slot-scope="{row}">
          <el-tag v-if="row.enableHTTP" type="success" size="mini" style="font-weight: bold">
            可
          </el-tag>
          <el-tag v-else type="info" size="mini" style="font-weight: bold">
            不
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column label="启停状态" class-name="status-col" width="80">
        <template slot-scope="{row}">
          <el-tag v-if="row.status === 1" type="success" size="mini" style="font-weight: bold">
            启
          </el-tag>
          <el-tag v-if="row.status === 0" type="info" size="mini" style="font-weight: bold">
            停
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column fixed="right" label="操作" align="center" width="380" class-name="small-padding">
        <template slot-scope="{row,$index}">
          <el-button v-permission="'function.edit'" type="primary" size="mini" @click="showUpdateForm(row)">
            编辑
          </el-button>
          <el-button v-permission="'function.debug'" type="success" size="mini" @click="handleShowDetail(row)">
            调试
          </el-button>
          <el-button v-permission="'function.debug'" size="mini" @click="handleShowLogs(row)">
            日志
          </el-button>
          <el-button v-permission="'function.edit'" size="mini" @click="handleTriggers(row)">
            触发器
          </el-button>
          <el-button v-if="row.status!='deleted'" v-permission="'function.delete'" size="mini" type="danger" @click="handleDelete(row,$index)">
            删除
          </el-button>
        </template>
      </el-table-column>
    </el-table>

    <!-- 分页 -->
    <pagination
      v-show="total>0"
      :total="total"
      :page.sync="listQuery.page"
      :limit.sync="listQuery.limit"
      @pagination="getList"
    />

    <!-- 表单对话框 -->
    <el-dialog :title="textMap[dialogStatus]" :visible.sync="dialogFormVisible">
      <el-form
        ref="dataForm"
        :rules="rules"
        :model="form"
        label-position="left"
        label-width="120px"
        style="width: 400px; margin-left:20px;"
      >
        <el-form-item label="显示名称" prop="label">
          <el-input v-model="form.label" placeholder="函数显示名，可为中文" />
        </el-form-item>
        <el-form-item label="函数标识" prop="name">
          <el-input v-model="form.name" placeholder="函数的唯一标识，如 get-user" />
        </el-form-item>
        <el-form-item label="HTTP访问" prop="enableHTTP">
          <el-switch v-model="form.enableHTTP" :active-value="true" :inactive-value="false" />
        </el-form-item>
        <el-form-item label="标签分类" prop="tags">
          <el-tag v-for="(tag, index) in form.tags" :key="tag" style="margin-right: 10px;" type="" size="medium" closable @close="removeTag(index)">{{ tag }}</el-tag>
          <el-autocomplete v-model="form._tag_input" :fetch-suggestions="suggestTags" class="input-new-tag" clearable size="mini" type="text" placeholder="添加" @select="addTag" @change="addTag" />
        </el-form-item>
        <el-form-item label="启用" prop="status">
          <el-switch v-model="form.status" :active-value="1" :inactive-value="0" />
        </el-form-item>
        <el-form-item label="函数描述">
          <el-input
            v-model="form.description"
            :autosize="{ minRows: 3, maxRows: 6}"
            type="textarea"
            placeholder="函数描述"
          />
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible = false">
          取消
        </el-button>
        <el-button type="primary" @click="dialogStatus==='create'?handleCreate():handleUpdate()">
          确定
        </el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import Pagination from '@/components/Pagination' // secondary package based on el-pagination
import { db } from '@/api/cloud'
import XLSX from 'xlsx'

const defaultCode = `
import cloud from '@/cloud-sdk'

exports.main = async function (ctx) {
  // body, query 为请求参数, auth 是授权对象
  const { auth, body, query } = ctx

  // 数据库操作
  const db = cloud.database()
  const db_res = await db.collection('admins').get()
  console.log(db_res)

  // 网络操作
  const res = await cloud.fetch("https://www.v2ex.com/api/topics/hot.json")
  console.log(res.status)

  return {
    data: db_res.data,
    fetch: res.data
  }
}
`

// 默认化创建表单的值
function getDefaultFormValue() {
  return {
    _id: undefined,
    name: '',
    label: '',
    description: '',
    status: 1,
    tags: [],
    enableHTTP: true,
    version: 0,
    created_at: Date.now(),
    updated_at: Date.now(),
    code: defaultCode,
    _tag_input: ''
  }
}

const formRules = {
  name: [{ required: true, message: '函数标识不可为空', trigger: 'blur' }],
  label: [{ required: true, message: '函数显示名称不可为空', trigger: 'blur' }]
}

export default {
  name: 'FunctionListPage',
  components: { Pagination },
  directives: { },
  filters: {
    statusFilter(status) {
      status = status ?? 0
      const statusMap = {
        0: 'disabled',
        1: 'enabled'
      }
      return statusMap[status]
    }
  },
  data() {
    return {
      tableKey: 0,
      list: null,
      total: 0,
      listLoading: true,
      listQuery: {
        page: 1,
        limit: 20,
        keyword: undefined,
        tag: ''
      },
      showReviewer: false,
      form: getDefaultFormValue(),
      dialogFormVisible: false,
      dialogStatus: '',
      textMap: {
        update: '编辑',
        create: '创建'
      },
      rules: formRules,
      downloadLoading: false,
      all_tags: []
    }
  },
  computed: {
  },
  created() {
    this.getList()
    this.getAllTags()
  },
  methods: {
    /** 获取所有标签 */
    async getAllTags() {
      const r = await db.collection('functions')
        .field(['tags'])
        .where({
          tags: db.command.exists(true)
        })
        .get()

      const tags = []
      for (const item of r.data) {
        tags.push(...(item?.tags || []))
      }
      this.all_tags = Array.from(new Set(tags))
    },
    /**
     * 获取数据列表
     */
    async getList() {
      this.listLoading = true

      // 拼装查询条件 by this.listQuery
      const { limit, page, keyword, tag } = this.listQuery
      const query = { }
      if (keyword) {
        query['$or'] = [
          { name: db.RegExp({ regexp: `.*${keyword}.*` }) },
          { label: db.RegExp({ regexp: `.*${keyword}.*` }) },
          { description: db.RegExp({ regexp: `.*${keyword}.*` }) }
        ]
      }

      if (tag !== '') {
        query['tags'] = tag
      }

      // 执行数据查询
      const res = await db.collection('functions')
        .where(query)
        .limit(limit)
        .skip((page - 1) * limit)
        .get()
        .catch(() => { this.listLoading = false })

      this.list = res.data

      // 获取数据总数
      const { total } = await db.collection('functions')
        .where(query)
        .limit(limit)
        .skip((page - 1) * limit)
        .count()
        .catch(() => { this.listLoading = false })

      this.total = total
      this.listLoading = false
    },
    // 搜索
    handleFilter() {
      this.listQuery.page = 1
      this.getList()
    },
    // 显示创建表单
    showCreateForm() {
      this.form = getDefaultFormValue()
      this.dialogStatus = 'create'
      this.dialogFormVisible = true
      this.$nextTick(() => {
        this.$refs['dataForm'].clearValidate()
      })
    },
    // 创建请求
    handleCreate() {
      this.$refs['dataForm'].validate(async(valid) => {
        if (!valid) { return }

        const data = Object.assign({}, this.form)
        delete data['_tag_input']

        // 执行创建请求
        const r = await db.collection('functions')
          .add(data)

        if (!r.id) {
          this.$notify({
            type: 'error',
            title: '操作失败',
            message: '创建失败！' + r.error
          })
          return
        }

        this.$notify({
          type: 'success',
          title: '操作成功',
          message: '创建成功！'
        })

        this.getList()
        this.dialogFormVisible = false
      })
    },
    // 显示更新表单
    showUpdateForm(row) {
      this.form = Object.assign(getDefaultFormValue(), row) // copy obj
      this.dialogStatus = 'update'
      this.dialogFormVisible = true
      this.$nextTick(() => {
        this.$refs['dataForm'].clearValidate()
      })
    },
    // 更新请求
    handleUpdate() {
      this.$refs['dataForm'].validate(async(valid) => {
        if (!valid) { return }

        // 执行创建请求
        const r = await db.collection('functions')
          .where({ _id: this.form._id })
          .update({
            name: this.form.name,
            label: this.form.label,
            tags: this.form.tags || [],
            description: this.form.description,
            enableHTTP: this.form.enableHTTP ?? true,
            status: this.form.status ?? 1,
            updated_at: Date.now()
          })

        if (!r.ok) {
          this.$notify({
            type: 'error',
            title: '操作失败',
            message: '更新失败！' + r.error
          })
          return
        }

        this.$notify({
          type: 'success',
          title: '操作成功',
          message: '更新成功！'
        })

        this.getList()
        this.dialogFormVisible = false
      })
    },
    // 删除请求
    async handleDelete(row, index) {
      await this.$confirm('确认要删除此数据？', '删除确认')

      // 执行删除请求
      const r = await db.collection('functions')
        .where({ _id: row._id })
        .remove()

      if (!r.ok) {
        this.$notify({
          type: 'error',
          title: '操作失败',
          message: '删除失败！' + r.error
        })
        return
      }

      this.$notify({
        type: 'success',
        title: '操作成功',
        message: '删除成功！'
      })

      this.list.splice(index, 1)
    },
    // 查看详情
    async handleShowDetail(row) {
      this.$router.push(`functions/${row._id}`)
    },
    // 查看日志详情
    async handleShowLogs(row) {
      this.$router.push(`function-logs/${row._id}`)
    },
    // 设置触发器
    async handleTriggers(row) {
      this.$router.push(`triggers/${row._id}`)
    },
    // 导出数据
    handleDownload() {
      this.downloadLoading = true

      if (!this.list || !this.list.length) {
        this.$message('函数列表暂无数据')
        this.downloadLoading = false
        return
      }

      const tableHeaders = Object.keys(this.list[0])

      const tableData = this.list.map(li => Object.values(li))
      tableData.unshift(tableHeaders)

      const ws = XLSX.utils.aoa_to_sheet(tableData)
      const wb = XLSX.utils.book_new()
      XLSX.utils.book_append_sheet(wb, ws, '云函数')
      XLSX.writeFile(wb, '云函数.xlsx')

      this.downloadLoading = false
    },
    // 搜索建议标签
    async suggestTags(queryString, cb) {
      const data = this.all_tags
        .filter(it => {
          return it.indexOf(queryString) >= 0
        })
        .map(it => {
          return { value: it }
        })

      cb(data)
    },
    // 删除标签
    removeTag(index) {
      const tags = this.form.tags || []
      if (!tags.length) return
      tags.splice(index, 1)
    },
    // 添加标签
    addTag() {
      const val = this.form._tag_input
      console.log('val: ', val)

      if (!val) return
      if (!this.form.tags.includes(val)) {
        this.form.tags.push(val)
      }
      this.form._tag_input = ''
    }
  }
}
</script>

<style lang="scss" scoped>
.tag-selector {
  margin: 20px 0;
  display: flex;

  .label {
    font-size: 14px;
    color: gray;
    align-self: center;
    padding-right: 20px;
  }
}

.func-name {
  color: black;
  // font-weight: bold;
  font-size: 16px;
}
.input-new-tag {
    width: 120px;
    margin-left: 10px;
    vertical-align: bottom;
}
</style>
