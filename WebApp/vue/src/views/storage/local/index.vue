<template>
  <div class="app-container" style="padding: 8px;">
    <div class="head-container">
      <!-- 搜索 -->
      <el-input
        v-model="listQuery.Filter"
        clearable
        size="small"
        placeholder="搜索..."
        style="width: 200px;"
        class="filter-item"
        @keyup.enter.native="handleFilter"
      />
      <el-button
        class="filter-item"
        size="mini"
        type="success"
        icon="el-icon-search"
        @click="handleFilter"
      >搜索</el-button>
      <div style="padding: 6px 0;">
        <el-button
          class="filter-item"
          size="mini"
          type="primary"
          icon="el-icon-upload"
          @click="handleUpload"
        >上传</el-button>
        <el-button
          slot="reference"
          class="filter-item"
          type="danger"
          icon="el-icon-delete"
          size="mini"
          @click="handleDelete()"
        >删除</el-button>
      </div>
    </div>
    <el-dialog
      :visible.sync="dialogFormVisible"
      :close-on-click-modal="false"
      :title="formTitle"
      @close="cancel()"
      width="500px"
    >
      <el-form
        ref="form"
        :inline="true"
        :model="form"
        :rules="rules"
        size="small"
        label-width="66px"
      >
        <el-form-item label="文件名" prop="name">
          <el-input v-model="form.name" style="width: 370px;" />
        </el-form-item>
        <el-form-item label="上传">
          <el-upload
            ref="upload"
            :limit="1"
            :before-upload="beforeUpload"
            :auto-upload="false"
            :on-success="handleSuccess"
            :on-error="handleError"
            :action="storageApi + '/upload?name=' + form.name"
          >
            <div class="upload">
              <i class="el-icon-upload" /> 添加文件
            </div>
            <div slot="tip" class="el-upload__tip">可上传任意格式文件，且不超过100M</div>
          </el-upload>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button size="small" type="text" @click="cancel">取消</el-button>
        <el-button size="small" v-loading="formLoading" type="primary" @click="upload">确认</el-button>
      </div>
    </el-dialog>

    <el-table
      ref="multipleTable"
      v-loading="listLoading"
      :data="list"
      size="small"
      style="width: 90%;"
      @sort-change="sortChange"
      @selection-change="handleSelectionChange"
      @row-click="handleRowClick"
    >
      <el-table-column type="selection" width="44px"></el-table-column>
      <el-table-column label="文件名" prop="name" sortable="custom" align="center" width="150px">
        <template slot-scope="scope">
          <el-popover
            :content="storageApi + scope.row.url"
            placement="top-start"
            title="路径"
            width="200"
            trigger="hover"
          >
            <a
              slot="reference"
              :href="storageApi + scope.row.url"
              class="el-link--primary"
              style="word-break:keep-all;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;color: #1890ff;font-size: 13px;"
              target="_blank"
            >{{ scope.row.name }}</a>
          </el-popover>
        </template>
      </el-table-column>
      <el-table-column label="预览图" prop="url" align="center">
        <template slot-scope="{row}">
          <el-image
            :src="storageApi + row.url"
            :preview-src-list="[storageApi + row.url]"
            fit="contain"
            lazy
            class="el-avatar"
          >
            <div slot="error">
              <i class="el-icon-document" />
            </div>
          </el-image>
        </template>
      </el-table-column>
      <el-table-column label="文件格式" prop="suffix" align="center" />
      <el-table-column label="类别" prop="type" align="center" />
      <el-table-column label="大小" prop="size" align="center" />
      <el-table-column label="创建日期" prop="creationTime" align="center" />
    </el-table>

    <pagination
      v-show="totalCount>0"
      :total="totalCount"
      :page.sync="page"
      :limit.sync="listQuery.MaxResultCount"
      @pagination="getList"
    />
  </div>
</template>

<script>
import Pagination from "@/components/Pagination";
import permission from "@/directive/permission/index.js";
import config from "../../../../static/config";

const defaultForm = {
  id: null,
  name: null,
};
export default {
  name: "Storage",
  components: { Pagination },
  directives: { permission },
  data() {
    return {
      storageApi: config.storage.ip,
      rules: {
        name: [{ required: true, message: "请输入岗位名", trigger: "blur" }],
      },
      form: Object.assign({}, defaultForm),
      list: null,
      totalCount: 0,
      listLoading: true,
      formLoading: false,
      listQuery: {
        Filter: "",
        Sorting: "",
        SkipCount: 0,
        MaxResultCount: 10,
      },
      page: 1,
      dialogFormVisible: false,
      multipleSelection: [],
      formTitle: "",
      isEdit: false,
    };
  },
  created() {
    this.getList();
  },
  methods: {
    // 上传文件
    upload() {
      this.$refs.upload.submit();
    },
    beforeUpload(file) {
      let isLt2M = true;
      isLt2M = file.size / 1024 / 1024 < 100;
      if (!isLt2M) {
        this.loading = false;
        this.$message.error("上传文件大小不能超过 100MB!");
      }
      this.form.name = file.name;
      return isLt2M;
    },
    handleSuccess(response, file, fileList) {
      this.$notify({
        title: "成功",
        message: "更新成功",
        type: "success",
        duration: 2000,
      });
      this.dialogFormVisible = false;
      this.getList();
    },
    // 监听上传失败
    handleError(e, file, fileList) {
      this.$notify({
        title: e,
        type: "error",
        duration: 4000,
      });
      this.loading = false;
    },
    getList() {
      this.listLoading = true;
      this.listQuery.SkipCount = (this.page - 1) * 10;
      this.$axios
        .gets(config.storage.ip + "/api/app/file", this.listQuery)
        .then((response) => {
          this.list = response.items;
          this.totalCount = response.totalCount;
          this.listLoading = false;
        });
    },
    handleFilter() {
      this.page = 1;
      this.getList();
    },
    save() {
      this.$refs.form.validate((valid) => {
        if (valid) {
          this.formLoading = true;
          this.form.roleNames = this.checkedRole;
          if (this.isEdit) {
            this.$axios
              .puts("/api/base/job/" + this.form.id, this.form)
              .then((response) => {
                this.formLoading = false;
                this.$notify({
                  title: "成功",
                  message: "更新成功",
                  type: "success",
                  duration: 2000,
                });
                this.dialogFormVisible = false;
                this.getList();
              })
              .catch(() => {
                this.formLoading = false;
              });
          } else {
            this.$axios
              .posts("/api/base/job", this.form)
              .then((response) => {
                this.formLoading = false;
                this.$notify({
                  title: "成功",
                  message: "新增成功",
                  type: "success",
                  duration: 2000,
                });
                this.dialogFormVisible = false;
                this.getList();
              })
              .catch(() => {
                this.formLoading = false;
              });
          }
        }
      });
    },
    handleUpload() {
      this.formTitle = "文件上传";
      this.isEdit = false;
      this.dialogFormVisible = true;
    },
    handleDelete(row) {
      var params = [];
      let alert = "";
      if (row) {
        params.push(row.id);
        alert = row.name;
      } else {
        if (this.multipleSelection.length === 0) {
          this.$message({
            message: "未选择",
            type: "warning",
          });
          return;
        }
        this.multipleSelection.forEach((element) => {
          let id = element.id;
          params.push(id);
        });
        alert = "选中项";
      }
      this.$confirm("是否删除" + alert + "?", "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$axios.posts("/api/base/job/delete", params).then((response) => {
            this.$notify({
              title: "成功",
              message: "删除成功",
              type: "success",
              duration: 2000,
            });
            this.getList();
          });
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已取消删除",
          });
        });
    },
    sortChange(data) {
      const { prop, order } = data;
      if (!prop || !order) {
        this.handleFilter();
        return;
      }
      this.listQuery.Sorting = prop + " " + order;
      this.handleFilter();
    },
    handleSelectionChange(val) {
      this.multipleSelection = val;
    },
    handleRowClick(row, column, event) {
      this.$refs.multipleTable.clearSelection();
      this.$refs.multipleTable.toggleRowSelection(row);
    },
    cancel() {
      this.form = Object.assign({}, defaultForm);
      this.dialogFormVisible = false;
      this.$refs.form.clearValidate();
    },
  },
};
</script>

<style rel="stylesheet/scss" lang="scss" scoped>
.upload {
  border: 1px dashed #c0ccda;
  border-radius: 5px;
  height: 45px;
  line-height: 45px;
  width: 368px;
}
</style>