<template>
  <el-container>
    <task-sidebar></task-sidebar>
    <el-main>
      <el-form :inline="true" >
        <el-form-item label="任务ID">
          <el-input v-model.trim="searchParams.task_id"></el-input>
        </el-form-item>
        <el-form-item label="执行方式">
          <el-select v-model.trim="searchParams.protocol" placeholder="执行方式">
            <el-option label="全部" value=""></el-option>
            <el-option
            v-for="item in protocolList"
            :key="item.value"
            :label="item.label"
            :value="item.value">
            </el-option>
          </el-select>
        </el-form-item>
        <el-form-item label="状态">
          <el-select v-model.trim="searchParams.status">
            <el-option label="全部" value=""></el-option>
            <el-option
              v-for="item in statusList"
              :key="item.value"
              :label="item.label"
              :value="item.value">
            </el-option>
          </el-select>
        </el-form-item>
        <el-form-item>
          <el-button type="primary" @click="search()">搜索</el-button>
        </el-form-item>
      </el-form>
      <el-row type="flex" justify="end">
        <el-col :span="2">
          <el-button type="danger" v-if="this.$store.getters.user.isAdmin" @click="clearLog">清空全部日志</el-button>
        </el-col>
        <el-col :span="2">
          <el-button type="warning" v-if="this.$store.getters.user.isAdmin" @click="delFailedLog">删除失败日志</el-button>
        </el-col>
        <el-col :span="2">
          <el-button type="info" @click="refresh">刷新</el-button>
        </el-col>
      </el-row>
      <!-- page-size: 默认每页展示条数 -->
      <!-- page-sizes: 可选每页展示条数列表 -->
      <el-pagination
        background
        layout="prev, pager, next, sizes, total"
        :total="logTotal"
        :page-size="1"
        :page-sizes="[1, 2, 3, 5, 10, 20, 30, 40, 50, 100]"
        @size-change="changePageSize"
        @current-change="changePage"
        @prev-click="changePage"
        @next-click="changePage">
      </el-pagination>
      <el-table
        :data="logs"
        border
        ref="table"
        style="width: 100%">
        <el-table-column type="expand">
          <template slot-scope="scope">
            <el-form label-position="left">
              <el-form-item>
                  重试次数: {{scope.row.retry_times}} <br>
                  cron表达式: {{scope.row.spec}} <br>
                  命令: {{scope.row.command}}
              </el-form-item>
            </el-form>
          </template>
        </el-table-column>
        <el-table-column
          prop="id"
          label="ID">
        </el-table-column>
        <el-table-column
          prop="task_id"
          label="任务ID">
        </el-table-column>
        <el-table-column
          prop="name"
          label="任务名称"
        width="600">
        </el-table-column>
        <el-table-column
          prop="protocol"
          label="执行方式"
          :formatter="formatProtocol">
        </el-table-column>
        <el-table-column
          label="任务节点"
          width="300">
          <template slot-scope="scope">
            <div v-html="scope.row.hostname">{{scope.row.hostname}}</div>
          </template>
        </el-table-column>
        <el-table-column
          label="执行时长"
          width="250">
          <template slot-scope="scope">
            执行时长: {{scope.row.total_time > 0 ? scope.row.total_time : 1}}秒<br>
            开始时间: {{scope.row.start_time | formatTime}}<br>
            <span v-if="scope.row.status !== 1">结束时间: {{scope.row.end_time | formatTime}}</span>
          </template>
        </el-table-column>
        <el-table-column
          label="状态">
          <template slot-scope="scope">
            <span style="color:red" v-if="scope.row.status === 0">失败</span>
            <span style="color:green" v-else-if="scope.row.status === 1">执行中</span>
            <span v-else-if="scope.row.status === 2">成功</span>
            <span style="color:#4499EE" v-else-if="scope.row.status === 3">取消</span>
          </template>
        </el-table-column>
        <el-table-column
          label="执行结果"
          width="120" v-if="this.isAdmin">
          <template slot-scope="scope">
            <el-button type="success"
                       v-if="scope.row.status === 2"
                       @click="showTaskResult(scope.row)">查看结果</el-button>
            <el-button type="warning"
                       v-if="scope.row.status === 0"
                       @click="showTaskResult(scope.row)" >查看结果</el-button>
            <el-button type="danger"
                       v-if="scope.row.status === 1 && scope.row.protocol === 2"
                       @click="stopTask(scope.row)">停止任务
            </el-button>
          </template>
        </el-table-column>
        <el-table-column
          label="执行结果"
          width="120" v-else>
          <template slot-scope="scope">
            <el-button type="success"
                       v-if="scope.row.status === 2"
                       @click="showTaskResult(scope.row)">查看结果</el-button>
            <el-button type="warning"
                       v-if="scope.row.status === 0"
                       @click="showTaskResult(scope.row)" >查看结果</el-button>
          </template>
        </el-table-column>
      </el-table>
      <el-dialog title="任务执行结果" :visible.sync="dialogVisible">
        <div>
          <pre>{{currentTaskResult.command}}</pre>
        </div>
        <div>
          <pre>{{currentTaskResult.result}}</pre>
        </div>
      </el-dialog>
    </el-main>
  </el-container>
</template>

<script>
import taskSidebar from '../task/sidebar'
import taskLogService from '../../api/taskLog'

export default {
  name: 'task-log',
  data () {
    return {
      logs: [],
      logTotal: 0,
      searchParams: {
        page_size: 1, // 默认每页展示条数（注：2024/08/15 【游品SDK生产-同步抖音get_ecpm接口数据】定时任务打了大量日志，一次性展示10条会因获取的数据过多导致gocron web后台响应超时中断请求而无法展示日志，故每次仅展示1条）
        page: 1,
        task_id: '',
        protocol: '',
        status: ''
      },
      isAdmin: this.$store.getters.user.isAdmin,
      dialogVisible: false,
      currentTaskResult: {
        command: '',
        result: ''
      },
      protocolList: [
        {
          value: '1',
          label: 'http'
        },
        {
          value: '2',
          label: 'shell'
        }
      ],
      statusList: [
        {
          value: '1',
          label: '失败'
        },
        {
          value: '2',
          label: '执行中'
        },
        {
          value: '3',
          label: '成功'
        },
        {
          value: '4',
          label: '取消'
        }
      ]
    }
  },
  components: {taskSidebar},
  created () {
    if (this.$route.query.task_id) {
      this.searchParams.task_id = this.$route.query.task_id
    }
    this.search()
  },
  methods: {
    formatProtocol (row, col) {
      if (row[col.property] === 1) {
        return 'http'
      }
      return 'shell'
    },
    changePage (page) {
      this.searchParams.page = page
      this.search()
    },
    changePageSize (pageSize) {
      this.searchParams.page_size = pageSize
      this.search()
    },
    search (callback = null) {
      taskLogService.list(this.searchParams, (data) => {
        this.logs = data.data
        this.logTotal = data.total

        if (callback) {
          callback()
        }
      })
    },
    clearLog () {
      this.$appConfirm(() => {
        taskLogService.clear(() => {
          this.searchParams.page = 1
          this.search()
        })
      })
    },
    delFailedLog () {
      this.$appConfirm(() => {
        taskLogService.delFailed(() => {
          this.searchParams.page = 1
          this.search()
        })
      })
    },
    stopTask (item) {
      taskLogService.stop(item.id, item.task_id, () => {
        this.search()
      })
    },
    showTaskResult (item) {
      this.dialogVisible = true
      this.currentTaskResult.command = item.command
      this.currentTaskResult.result = item.result
    },
    refresh () {
      this.search(() => {
        this.$message.success('刷新成功')
      })
    }
  }
}
</script>
<style scoped>
  pre {
    white-space: pre-wrap;
    word-wrap: break-word;
    padding:10px;
    background-color: #4C4C4C;
    color: white;
  }
</style>
