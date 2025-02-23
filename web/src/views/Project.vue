<template>
  <div>
    <el-table :data="tableData" stripe style="width: 100%"
              @sort-change="sortChange"
              table-layout="auto"
              v-loading="loading">
      <el-table-column fixed="left" prop="Url" label="项目" sortable="custom">
        <template #default="scope">
          <el-link type="primary" :href="scope.row.Url" target="_blank">{{ scope.row.Url.slice(19) }}</el-link>
        </template>
      </el-table-column>
      <el-table-column prop="Language" label="语言" sortable="custom" />
      <el-table-column prop="Suite" label="查询套件" width="100px">
        <template #default="scope">
          <el-tag v-for="(item, index) in scope.row.Suite"
                  :key="index"
                  type="info"
                  style="margin-right: 5px">
            {{ item }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column prop="Mode" :formatter="modeFormatter" label="扫描模式" sortable="custom"/>
      <el-table-column label="暂停" prop="Pause" sortable="custom">
        <template #default="scope">
          <el-icon v-if="scope.row.Pause" :size="20" style="margin-top: 8px">
            <VideoPause/>
          </el-icon>
          <el-icon v-if="!scope.row.Pause" :size="20" color="#a3d280" style="margin-top: 8px">
            <VideoPlay/>
          </el-icon>
        </template>
      </el-table-column>
      <el-table-column prop="LatestVersion" label="最新版本">
        <template #default="scope">
          <el-tooltip
              effect="dark"
              :content="scope.row.LatestVersionErrorInfo"
              placement="top"
              :disabled="!(scope.row.LatestVersion==='[Error]')"
          >
            <span v-if="scope.row.Mode == 0">{{ releaseVersionFormatter(scope.row.LatestVersion) }}</span>
            <span v-if="scope.row.Mode == 1">{{ commitVersionFormatter(scope.row.LatestVersion) }}</span>
          </el-tooltip>
        </template>
      </el-table-column>
      <el-table-column prop="LatestVersionUpdateTime" :formatter="(row, col, value, index)=>timeFormatter(value)"
                       label="更新时间" sortable="custom"/>
      <el-table-column prop="LastAnalyzeTime" :formatter="(row, col, value, index)=>timeFormatter(value)"
                       label="最近扫描时间" sortable="custom"/>
      <el-table-column label="最近扫描版本">
        <template #default="scope">
          <span v-if="scope.row.Mode == 0">{{ releaseVersionFormatter(scope.row.LastAnalyzeReleaseTag) }}</span>
          <span v-if="scope.row.Mode == 1">{{ commitVersionFormatter(scope.row.LastAnalyzeDatabaseCommit) }}</span>
        </template>
      </el-table-column>

      <el-table-column fixed="right" label="" width="132px">
        <template #header>
          <el-button style="float: right" :icon="Plus" @click="createData" circle/>
        </template>
        <template #default="scope">
          <el-tooltip
              content="加入扫描队列"
              placement="left-start"
          >
            <el-button :icon="Aim" circle @click="runTask(scope.row.ID)"/>
          </el-tooltip>
          <el-button :icon="Edit" circle @click="updateData(scope.row)" style="margin-left: 6px"/>
          <el-popconfirm title="确认删除?" :hide-after="0" @confirm="deleteData(scope.row.ID)">
            <template #reference>
              <el-button :icon="Delete" circle style="margin-left: 6px"/>
            </template>
          </el-popconfirm>
        </template>
      </el-table-column>
    </el-table>

    <el-pagination
        style="margin-top: 20px"
        v-model:current-page="currentPage"
        v-model:page-size="pageSize"
        :page-sizes="[1, 5, 10, 15, 20, 50]"
        layout="total, sizes, prev, pager, next"
        v-model:total="total"
        @size-change="fetchData"
        @current-change="fetchData"
    />

    <el-dialog v-model="dialogFormVisible" :title="form.ID==0?'新增项目':'编辑项目'">
      <el-form :model="form">
        <el-form-item label="项目地址">
          <el-input v-model="form.Url" autocomplete="off" placeholder="https://github.com/owner/repo"/>
        </el-form-item>
        <el-form-item label="项目语言">
          <el-select v-model="form.Language" filterable allow-create placeholder="Select" style="width:100%">
            <el-option
                v-for="item in ['java','go','python','cpp','csharp','swift','javascript','ruby']"
                :value="item"
            />
          </el-select>
        </el-form-item>
        <el-form-item label="查询套件">
          <el-select v-model="form.Suite" multiple
                     filterable
                     clearable
                     ref="suiteSelect"
                     @change="suiteSelectChange"
                     placeholder="Select" style="width:100%">
            <el-option
                v-for="item in suites"
                :value="item.Name"
            />
          </el-select>
        </el-form-item>
        <el-form-item label="扫描模式">
          <el-select v-model="form.Mode" class="m-2" placeholder="Select" style="width:100%">
            <el-option
                v-for="item in [{'label':'Release',value:0},{'label':'原有数据库',value:1}]"
                :label="item.label"
                :value="item.value"
            />
          </el-select>
        </el-form-item>
        <el-form-item v-if="form.Mode!=1" label="编译命令">
          <el-input v-model="form.Command" autocomplete="off" placeholder="一般可留空，CodeQL会自动识别"/>
        </el-form-item>
        <el-form-item label="暂停监控">
          <el-switch v-model="form.Pause" style="--el-switch-on-color: #ff4949;"/>
        </el-form-item>
      </el-form>
      <template #footer>
      <span class="dialog-footer">
        <el-button @click="dialogFormVisible = false">Cancel</el-button>
        <el-button type="primary" @click="saveData">
          Confirm
        </el-button>
      </span>
      </template>
    </el-dialog>
  </div>
</template>

<script lang="ts" setup>
import {onMounted, reactive, ref} from 'vue'
import {deleteProject, getProjects, saveProject} from '../api/project.js'
import {getSuites} from "../api/suite.js"
import {timeFormatter} from "../utils/formatter.js"
import {runTaskByID} from "../api/task.js"
import {Aim, Delete, Edit, Plus, VideoPause, VideoPlay} from '@element-plus/icons-vue'
import {ElMessage} from "element-plus";

const emit = defineEmits(["refresh"]);

const loading = ref(true)

const tableData = ref()
const currentPage = ref(1)
const pageSize = ref(20)
const total = ref(100)


const sort = reactive({
  name: "CreatedAt",
  order: "descending"
})

const sortChange = (column) => {
  sort.name = column.prop
  sort.order = column.order
  fetchData()
}

const fetchData = () => {
  loading.value = true
  getProjects(currentPage.value, pageSize.value, sort.name, sort.order).then(response => {
    tableData.value = response["data"];
    total.value = <number>response["total"];
    loading.value = false
  }).catch(err => {
    loading.value = false
  })
}

const modeFormatter = (row, col, value, index) => {
  if (value === 0) {
    return "Release";
  } else if (value === 1) {
    return "原有数据库";
  }
  return value;
}


const dialogFormVisible = ref(false)
const form = reactive({
  ID: 0,
  Url: "",
  Mode: 1,
  Language: "",
  Command: "",
  Suite: [],
  Pause: false
})
const suites = ref()

const initSuites = () => {
  getSuites(1, -1, null, null).then(response => {
    suites.value = response.data
  })
}

const createData = () => {
  form.ID = 0;
  form.Url = "";
  form.Mode = 1;
  form.Language = "";
  form.Command = "";
  form.Suite = [];
  form.Pause = false
  dialogFormVisible.value = true;
}
const updateData = (row) => {
  form.ID = row.ID;
  form.Url = row.Url;
  form.Mode = row.Mode;
  form.Language = row.Language;
  form.Command = row.Command;
  form.Suite = row.Suite;
  form.Pause = row.Pause;
  dialogFormVisible.value = true;
}
const saveData = () => {
  saveProject(form).then(response => {
    fetchData();
    dialogFormVisible.value = false;
    ElMessage.success("保存成功")
  })
}

const deleteData = (ID) => {
  deleteProject(ID).then(response => {
    fetchData();
    ElMessage.success("删除成功")
  })
}

const runTask = (ID) => {
  runTaskByID(ID).then(response => {
    emit("refresh")
    if (response.data.ok) {
      ElMessage.success("已添加到任务队列")
    } else {
      ElMessage.info("任务已在进行或队列中")
    }
  })
}

const releaseVersionFormatter = (tag) => {
  if (tag == "") {
    return "/"
  }
  return tag
}

const commitVersionFormatter = (commit) => {
  if (commit == "") {
    return "/"
  }
  return commit.substring(0, 7)
}


const suiteSelect = ref(null);
const suiteSelectTimeout = ref(null);
const suiteSelectChange = () => {
  if(form.Suite.length>0){
    if (suiteSelectTimeout.value){
      clearTimeout(suiteSelectTimeout.value)
    }
    suiteSelectTimeout.value = setTimeout(() => {
      suiteSelect.value.blur()
    }, 10)
  }
}

onMounted(() => {
  fetchData();
  initSuites();
})
</script>