<template>
  <div class="container list" v-loading="loading">

    <div class="handle-box">
      <el-button type="primary" :icon="Plus" @click="add">新增</el-button>
    </div>

    <el-row>
      <el-table :data="items" :row-key="row => row.id" table-layout="auto">
        <el-table-column prop="platform" label="所属平台">
          <template #default="scope">
            <span class="sort" :data-id="scope.row.id">{{ scope.row.platform }}</span>
          </template>
        </el-table-column>
        <el-table-column prop="name" label="模型名称"/>
        <el-table-column prop="value" label="模型值"/>
        <el-table-column prop="weight" label="对话权重"/>
        <el-table-column prop="enabled" label="启用状态">
          <template #default="scope">
            <el-switch v-model="scope.row['enabled']" @change="enable(scope.row)"/>
          </template>
        </el-table-column>

        <el-table-column label="创建时间">
          <template #default="scope">
            <span>{{ dateFormat(scope.row['created_at']) }}</span>
          </template>
        </el-table-column>

        <el-table-column label="操作" width="180">
          <template #default="scope">
            <el-button size="small" type="primary" @click="edit(scope.row)">编辑</el-button>
            <el-popconfirm title="确定要删除当前记录吗?" @confirm="remove(scope.row)">
              <template #reference>
                <el-button size="small" type="danger">删除</el-button>
              </template>
            </el-popconfirm>
          </template>
        </el-table-column>
      </el-table>
    </el-row>

    <el-dialog
        v-model="showDialog"
        :title="title"
        style="width: 90%; max-width: 600px;"
    >
      <el-form :model="item" label-width="120px" ref="formRef" :rules="rules">
        <el-form-item label="所属平台：" prop="platform">
          <el-select v-model="item.platform" placeholder="请选择平台">
            <el-option v-for="item in platforms" :value="item.value" :key="item.value">{{ item.name }}</el-option>
          </el-select>
        </el-form-item>

        <el-form-item label="模型名称：" prop="name">
          <el-input v-model="item.name" autocomplete="off"/>
        </el-form-item>

        <el-form-item label="模型值：" prop="value">
          <el-input v-model="item.value" autocomplete="off"/>
        </el-form-item>

        <el-form-item label="对话权重：" prop="weight">

          <template #default>
            <div class="tip-input">
              <el-input-number :min="1" v-model="item.weight" autocomplete="off"/>
              <div class="info">
                <el-tooltip
                    class="box-item"
                    effect="dark"
                    content="对话权重，每次对话扣减多少次对话额度"
                    placement="right"
                >
                  <el-icon>
                    <InfoFilled/>
                  </el-icon>
                </el-tooltip>
              </div>
            </div>
          </template>
        </el-form-item>

        <el-form-item label="启用状态：" prop="enable">
          <el-switch v-model="item.enabled"/>
        </el-form-item>
      </el-form>

      <template #footer>
            <span class="dialog-footer">
              <el-button @click="showDialog = false">取消</el-button>
              <el-button type="primary" @click="save">提交</el-button>
            </span>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import {onMounted, reactive, ref} from "vue";
import {httpGet, httpPost} from "@/utils/http";
import {ElMessage} from "element-plus";
import {dateFormat, removeArrayItem} from "@/utils/libs";
import {InfoFilled, Plus} from "@element-plus/icons-vue";
import {Sortable} from "sortablejs";

// 变量定义
const items = ref([])
const item = ref({})
const showDialog = ref(false)
const title = ref("")
const rules = reactive({
  platform: [{required: true, message: '请选择平台', trigger: 'change',}],
  name: [{required: true, message: '请输入模型名称', trigger: 'change',}],
  value: [{required: true, message: '请输入模型值', trigger: 'change',}]
})
const loading = ref(true)
const formRef = ref(null)
const platforms = ref([
  {name: "【OpenAI】ChatGPT", value: "OpenAI"},
  {name: "【讯飞】星火大模型", value: "XunFei"},
  {name: "【清华智普】ChatGLM", value: "ChatGLM"},
  {name: "【百度】文心一言", value: "Baidu"},
  {name: "【微软】Azure", value: "Azure"},
])

// 获取数据
httpGet('/api/admin/model/list').then((res) => {
  if (res.data) {
    // 初始化数据
    const arr = res.data;
    for (let i = 0; i < arr.length; i++) {
      arr[i].last_used_at = dateFormat(arr[i].last_used_at)
    }
    items.value = arr
  }
  loading.value = false
}).catch(() => {
  ElMessage.error("获取数据失败");
})


onMounted(() => {
  const drawBodyWrapper = document.querySelector('.el-table__body tbody')

  // 初始化拖动排序插件
  Sortable.create(drawBodyWrapper, {
    sort: true,
    animation: 500,
    onEnd({newIndex, oldIndex, from}) {
      if (oldIndex === newIndex) {
        return
      }

      const sortedData = Array.from(from.children).map(row => row.querySelector('.sort').getAttribute('data-id'));
      const ids = []
      const sorts = []
      sortedData.forEach((id, index) => {
        ids.push(parseInt(id))
        sorts.push(index)
        items.value[index].sort_num = index
      })

      httpPost("/api/admin/model/sort", {ids: ids, sorts: sorts}).then(() => {
      }).catch(e => {
        ElMessage.error("排序失败：" + e.message)
      })
    }
  })
})

const add = function () {
  title.value = "新增模型"
  showDialog.value = true
  item.value = {enabled: true, weight: 1}
}

const edit = function (row) {
  title.value = "修改模型"
  showDialog.value = true
  item.value = row
}

const save = function () {
  formRef.value.validate((valid) => {
    if (valid) {
      showDialog.value = false
      httpPost('/api/admin/model/save', item.value).then((res) => {
        ElMessage.success('操作成功！')
        if (!item.value['id']) {
          const newItem = res.data
          items.value.push(newItem)
        }
      }).catch((e) => {
        ElMessage.error('操作失败，' + e.message)
      })
    } else {
      return false
    }
  })
}

const enable = (row) => {
  httpPost('/api/admin/model/enable', {id: row.id, enabled: row.enabled}).then(() => {
    ElMessage.success("操作成功！")
  }).catch(e => {
    ElMessage.error("操作失败：" + e.message)
  })
}

const remove = function (row) {
  httpGet('/api/admin/model/remove?id=' + row.id).then(() => {
    ElMessage.success("删除成功！")
    items.value = removeArrayItem(items.value, row, (v1, v2) => {
      return v1.id === v2.id
    })
  }).catch((e) => {
    ElMessage.error("删除失败：" + e.message)
  })
}
</script>

<style lang="stylus" scoped>
@import "@/assets/css/admin-form.styl"
.list {

  .opt-box {
    padding-bottom: 10px;
    display flex;
    justify-content flex-end

    .el-icon {
      margin-right: 5px;
    }
  }

  .el-select {
    width: 100%
  }

}
</style>