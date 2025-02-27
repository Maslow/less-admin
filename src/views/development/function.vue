<template>
  <div class="app-container">
    <div v-if="func" class="header">
      <span style="font-size: 26px;line-height: 40px"><b>{{ func.label }}</b> </span>
      <el-tag style="margin-left: 20px" size="small" type="success">{{ func.name }}</el-tag>
      <el-button
        v-permission="'function.read'"
        style="margin-left: 20px"
        icon="el-icon-refresh"
        type="text"
        :disabled="loading"
        size="default"
        @click="getFunction"
      >刷新</el-button>
      <el-button
        v-permission="'function.edit'"
        type="success"
        size="small"
        style="margin-left: 20px;"
        :disabled="loading || !func"
        @click="updateFunc"
      >保存(S)</el-button>
      <el-button size="medium" style="float: right;" type="primary" @click="showDebugPanel = true">显示调试面板(J)</el-button>
    </div>

    <div style="display: flex;">
      <div class="editor-container">
        <function-editor v-model="value" :height="700" :dark="false" @save="updateFunc" />
      </div>
      <div class="lastest-logs">
        <el-card shadow="never" :body-style="{ padding: '20px' }">
          <div slot="header">
            <span>最近执行</span>
            <el-button style="float: right; padding: 3px 0" type="text" @click="getLatestLogs">刷新</el-button>
          </div>
          <div v-for="log in lastestLogs" :key="log._id" class="log-item">

            <el-tag type="warning" size="normal" @click="showLogDetailDlg(log)">
              {{ log.created_at | parseTime('{y}-{m}-{d} {h}:{i}:{s}') }}
            </el-tag>

            <!-- <el-tag class="time" type="default" size="normal" @click="showLogDetailDlg(log)">查看</el-tag> -->
          </div>
        </el-card>
      </div>
    </div>

    <el-drawer
      title="调试面板"
      :visible.sync="showDebugPanel"
      direction="rtl"
      size="40%"
      :destroy-on-close="false"
      :show-close="true"
      :modal="true"
      :wrapper-closable="true"
    >

      <div class="invoke-panel">
        <div class="title">
          调用参数
          <el-button
            v-permission="'function.debug'"
            size="mini"
            type="success"
            style="margin-left: 10px"
            :disabled="loading || !func"
            @click="launch"
          >运行(B)</el-button>
        </div>
        <div class="editor">
          <json-editor
            v-model="invokeParams"
            :line-numbers="false"
            :height="200"
            :dark="false"
          />
        </div>
        <div v-if="invokeResult" class="invoke-result">
          <div class="title">
            执行日志
            <span
              v-if="invokeResult"
            >（ RequestId: {{ invokeResult.requestId }} ）</span>
          </div>
          <div class="logs">
            <div v-for="(log, index) in logs" :key="index" class="log-item">
              <pre>- {{ log }}</pre>
            </div>
          </div>
          <div class="title" style="margin-top: 20px">
            调用结果 <span v-if="invokeTime"> （ {{ invokeTime }} ms ）</span>
          </div>
          <div class="result">
            <pre>{{ invokeReturn }}</pre>
          </div>
        </div>
      </div>
    </el-drawer>

    <!-- 日志详情对话框 -->
    <el-dialog v-if="logDetail" :visible.sync="isShowLogDetail" title="日志详情">
      <FunctionLogDetail :data="logDetail" />
    </el-dialog>
  </div>
</template>

<script>
import FunctionLogDetail from './components/FunctionLogDetail'
import FunctionEditor from './components/FunctionEditor'
import jsonEditor from './components/JsonEditor/param'
import { db, cloud } from '@/api/cloud'

const defaultParamValue = {
  code: 'less'
}
export default {
  name: 'FunctionEditorPage',
  components: { FunctionEditor, jsonEditor, FunctionLogDetail },
  data() {
    return {
      loading: false,
      value: '',
      func: null,
      func_id: '',
      invokeParams: defaultParamValue,
      invokeResult: null,
      // 调试面板显示控制
      showDebugPanel: false,
      // 最近日志
      lastestLogs: [],
      // 当前查看日志详情
      logDetail: null,
      // 日志详情对话框显示控制
      isShowLogDetail: true
    }
  },
  computed: {
    // 调用云函数的日志
    logs() {
      if (!this.invokeResult) {
        return []
      }
      return this.invokeResult.logs
    },
    // 调用云函数返回的值
    invokeReturn() {
      if (!this.invokeResult) {
        return ''
      }
      return this.invokeResult.data
    },
    // 云函数执行用时
    invokeTime() {
      if (!this.invokeResult) {
        return null
      }
      return this.invokeResult.time_usage
    }
  },
  watch: {
    func(val) {
      this.value = val.code
    }
  },
  async created() {
    this.func_id = this.$route.params.id
    await this.getFunction()
    this.getLatestLogs()
    this.setTagViewTitle()
  },
  mounted() {
    document.addEventListener('keydown', this.bindShortKey, false)
  },
  beforeDestroy() {
    document.removeEventListener('keydown', this.bindShortKey, false)
  },
  methods: {
    /**
     * 获取函数数据
     */
    async getFunction() {
      const func_id = this.func_id
      this.loading = true
      const r = await db
        .collection('functions')
        .where({ _id: func_id })
        .getOne()

      if (!r.ok || !r.data) {
        this.$notify({
          type: 'error',
          title: '错误',
          message: '加载函数失败：' + r.error
        })
        return
      }

      this.func = r.data
      this.value = this.func.code
      this.invokeParams = this.parseInvokeParam(this.func.debugParams) ?? defaultParamValue
      this.loading = false
    },
    /**
     * 保存函数代码
     */
    async updateFunc(showTip = true) {
      if (this.loading) {
        return
      }
      if (this.validate()) {
        return
      }

      this.loading = true

      let param = this.invokeParams
      if (typeof param !== 'string') {
        param = JSON.stringify(this.invokeParams)
      }
      const r = await db
        .collection('functions')
        .where({
          _id: this.func._id
        })
        .update({
          code: this.value,
          update_at: Date.now(),
          debugParams: param
        })

      if (!r.ok) {
        this.$message('保存失败!')
        this.loading = false
        return
      }

      if (showTip) {
        await this.getFunction()
        await this.addFunctionHistory()
        this.$message.success('已保存')
      }

      this.loading = false
    },
    /**
     * 运行函数代码
     */
    async launch() {
      await this.updateFunc(false)
      if (this.loading) {
        return
      }

      const param = this.parseInvokeParam(this.invokeParams)

      const res = await cloud.invokeFunction(this.func.name, param, true)
        .catch(err => {
          console.error(err)
          this.$message.warning('运行失败： ' + err)
        })

      this.$message.success('运行成功')

      this.invokeResult = res
      this.getLatestLogs()
    },
    /**
     * 获取最近日志
     */
    async getLatestLogs() {
      this.loading = true

      // 执行数据查询
      const res = await db.collection('function_logs')
        .where({ func_id: this.func_id })
        .limit(20)
        .skip(0)
        .orderBy('created_at', 'desc')
        .get()
        .finally(() => { this.loading = false })

      this.lastestLogs = res.data || []
    },
    /**
     * 添加函数的更新记录
     */
    async addFunctionHistory() {
      const data = Object.assign({}, this.func)
      await db.collection('function_history')
        .add({
          func_id: this.func._id,
          data: data,
          created_at: Date.now()
        })
    },
    showLogDetailDlg(log) {
      this.logDetail = log
      this.isShowLogDetail = true
    },
    setTagViewTitle() {
      const label = this.func.label
      const title = this.$route.meta.title
      const route = Object.assign({}, this.$route, { title: `${title}: ${label}` })
      this.$store.dispatch('tagsView/updateVisitedView', route)
    },
    validate() {
      let error = null

      if (this.value === '') {
        error = '函数值不可为空'
      }

      if (error) {
        this.$message(error)
        return error
      }
      return null
    },
    // 解析函数调试参数
    parseInvokeParam(data) {
      let param
      try {
        param = JSON.parse(data)
      } catch (error) {
        console.log(data, error)
        param = data
      }

      return param
    },
    // 快捷键绑定
    bindShortKey(e) {
      // Ctrl + b 为调试运行，并弹出
      if ((e.ctrlKey || e.metaKey) && e.key === 'j') {
        this.showDebugPanel = !this.showDebugPanel
        e.preventDefault()
      }

      if ((e.ctrlKey || e.metaKey) && e.key === 'b') {
        this.showDebugPanel = true
        this.launch()
        e.preventDefault()
      }
    }
  }
}
</script>

<style lang="scss" scoped>
.app-container {
  padding-top: 10px;
}
.editor-container {
  position: relative;
  height: 100%;
  margin-top: 10px;
  width: 90%;
  border: 1px solid lightgray;
}

.lastest-logs {
  padding-left: 20px;
  padding-top: 10px;
  width: 15%;

  .log-item {
    margin-top: 10px;
    display: flex;
    justify-content: space-between;

    .time{
      margin-left: 10px;
    }
  }
}

.invoke-panel {
  padding-left: 20px;
  padding-top: 10px;
  width: 100%;
  height: calc(90vh);
  padding-bottom: 20px;
  overflow-y: scroll;
  overflow-x: hidden;

  .title {
    font-weight: bold;
    span {
      font-weight: normal;
      color: gray;
    }
  }
  .editor {
    margin-top: 10px;
    border: 1px dashed gray;
    margin-left: 2px;
    width: 100%;
  }
  .invoke-result {
    margin-top: 20px;
    .logs {
      margin-top: 10px;
      padding: 10px;
      padding-left: 20px;
      background: rgba(233, 243, 221, 0.472);
      border-radius: 10px;
      overflow-x: auto;
    }
    .result {
      margin-top: 10px;
      padding: 16px;
      background: rgba(233, 243, 221, 0.472);
      border-radius: 10px;
      overflow-x: auto;
    }
  }
}
</style>

