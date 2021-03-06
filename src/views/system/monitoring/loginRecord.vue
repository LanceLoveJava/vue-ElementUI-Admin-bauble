<template>
  <div class="app-container">
    <el-row class="filter-container">
      <el-button
        size="small"
        type="primary"
        icon="el-icon-search"
        @click="searchHandle"
      >搜索</el-button>
      <el-button v-if="RP(['ent:drafts:batchDel'])" type="danger" icon="el-icon-delete" @click="batchDelete">批量删除</el-button>
    </el-row>

    <el-row>
      <el-table
        ref="tableData"
        :data="tableData"
        row-key="id"
        border
        :max-height="tableHeight"
        :highlight-current-row="true"
        @select="checkboxSelect"
        @select-all="checkboxSelect"
      >
        <el-table-column fixed type="selection" width="55" align="center" />
        <el-table-column fixed prop="ipaddr" label="登入IP" min-width="120" align="center" />
        <el-table-column prop="userName" label="登入用户" :show-overflow-tooltip="true" min-width="120" align="center" />
        <el-table-column prop="browser" label="登入设备" :show-overflow-tooltip="true" min-width="120" align="center" />
        <el-table-column prop="os" label="登入系统" :show-overflow-tooltip="true" min-width="120" align="center" />
        <el-table-column prop="loginTime" label="登入时间" min-width="150" align="center" />
        <el-table-column fixed="right" align="center" label="操作" min-width="220">
          <template slot-scope="scope">
            <el-button v-if="RP(['ent:drafts:edit'])" type="primary" style="margin-left: 8px;" size="mini" @click="handleLook(scope.row)">详情</el-button>
<!--            <el-button v-if="RP(['ent:drafts:del'])" type="danger" style="margin-left: 8px;" size="mini" @click="handleDelete(scope.row)">删除</el-button>-->
          </template>
        </el-table-column>
      </el-table>
    </el-row>
    <el-row class="footer-page">
      <el-pagination
        :total="filterForm.total"
        :hide-on-single-page="false"
        :current-page="filterForm.current"
        :page-size="filterForm.size"
        layout="total, sizes, prev, pager, next, jumper"
        :page-sizes="[25, 50, 100, 200]"
        @size-change="handleSizeChange"
        @current-change="handleCurrentChange"
      />
    </el-row>

    <el-dialog
      :visible.sync="dialogVisible"
      :close-on-click-modal="false"
      :title="dialogType==='add'?'新增':'编辑'"
      width="40%"
      @close="closeDialogFun('saveForm')"
    >
      <el-form
        ref="saveForm"
        :model="saveForm"
        label-position="left"
      >
        <el-form-item label="登入IP" prop="name">
          <span>{{ saveForm.ipaddr }}</span>
        </el-form-item>
        <el-form-item label="登入用户">
          <span>{{ saveForm.userName }}</span>
        </el-form-item>
        <el-form-item label="登入设备">
          <span>{{ saveForm.browser }}</span>
        </el-form-item>
        <el-form-item label="登入系统">
          <span>{{ saveForm.os }}</span>
        </el-form-item>
        <el-form-item label="登入时间">
          <span>{{ saveForm.loginTime }}</span>
        </el-form-item>
        <el-form-item label="地址">
          <span>{{ saveForm.loginLocation }}</span>
        </el-form-item>
      </el-form>
      <div style="text-align:right;">
        <el-button type="danger" @click="dialogVisible=false">取消</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import { mapGetters } from 'vuex';
import { get, post, deleteData } from '@/api/common';
import { delMessage } from '@/utils/common';
import { forEachRen } from '@/utils';

export default {
  name: 'LoginRecord',
  filters: {
    statusFil(val) {
      const status = ['状态1', '状态2', '状态3'];
      return status[val];
    }
  },
  data() {
    return {
      filterForm: {
        size: 25,
        total: 1,
        current: 1
      },
      saveForm: Object.assign({}),
      dialogVisible: false,
      dialogType: 'add',
      tableData: [],
      deleteList: []
    };
  },
  computed: {
    ...mapGetters(['tableHeight'])
  },
  mounted() {
    this.getTableData();
  },
  methods: {

    searchHandle() {
      this.filterForm.current = 1;
      this.getTableData();
    },
    getTableData() {
      get('/api/system/monitor/loginlog/page', this.filterForm).then(res => {
        this.tableData = res.data.records;
        this.filterForm.total = res.data.total;
        this.filterForm.size = res.data.size;
        this.$nextTick(() => {
          res.loading.close();
        });
      }).catch(() => {
      });
    },
    handleCurrentChange(page) {
      this.filterForm.current = page;
      this.getTableData();
    },
    handleSizeChange(size) {
      this.filterForm.size = size;
      this.getTableData();
    },
    batchDelete() {
      if (!this.deleteList.length) {
        return this.$message.error('请先选择数据!');
      }
      const delList = forEachRen(this.deleteList, 'id');
      const _this = this;
      this.$confirm('您确定要删除这些数据吗?', '警告', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      })
        .then(async() => {
          await post('/api/system/monitor/loginlog/delete/batch', {
            idList: delList
          }).then(() => {
            if (delList.length === this.tableData.length) {
              this.filterForm.current =
                  this.filterForm.current - 1 > 0 ? this.filterForm.current - 1 : 1;
            }
            _this.getTableData();
            this.deleteList.length = 0;
            _this.$message({
              type: 'success',
              message: '删除成功!'
            });
          });
        })
        .catch(() => {
          return false;
        });
    },
    handleLook(row) {
      get('/api/system/monitor/loginlog/detail/' + row.id).then(res => {
        this.saveForm = res.data;
        this.dialogVisible = true;
        this.$nextTick(() => {
          res.loading.close();
        });
      });
    },
    handleDelete(row) {
      delMessage('/api/collect/record/delete/' + row.id).then(res => {
        if (res) {
          if (this.tableData.length === 1) {
            this.filterForm.current =
              this.filterForm.current - 1 > 0 ? this.filterForm.current - 1 : 1;
          }
          this.getTableData();
        }
      });
    },
    checkboxSelect(select) {
      this.deleteList = select;
    }
  }
};
</script>

<style scoped>

</style>
