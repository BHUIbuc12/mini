<template>
  <div>
    <el-card class="container-card" shadow="always">
      <el-form :inline="true" :model="params" class="demo-form-inline" size="mini">
        <el-form-item label="角色名称">
          <el-input v-model.trim="params.name" clearable placeholder="角色名称" @clear="search" />
        </el-form-item>
        <el-form-item label="关键字">
          <el-input v-model.trim="params.keyword" clearable placeholder="关键字" @clear="search" />
        </el-form-item>
        <el-form-item label="角色状态">
          <el-select v-model.trim="params.status" clearable placeholder="角色状态" @change="search" @clear="search">
            <el-option :value="1" label="正常" />
            <el-option :value="2" label="禁用" />
          </el-select>
        </el-form-item>
        <el-form-item>
          <el-button :loading="loading" icon="el-icon-search" type="primary" @click="search">查询</el-button>
        </el-form-item>
        <el-form-item>
          <el-button :loading="loading" icon="el-icon-plus" type="warning" @click="create">新增</el-button>
        </el-form-item>
        <el-form-item>
          <el-button
            :disabled="multipleSelection.length === 0"
            :loading="loading"
            icon="el-icon-delete"
            type="danger"
            @click="batchDelete"
          >批量删除
          </el-button>
        </el-form-item>
      </el-form>

      <el-table
        v-loading="loading"
        :data="tableData"
        border
        stripe
        style="width: 100%"
        @selection-change="handleSelectionChange"
      >
        <el-table-column align="center" type="selection" width="55" />
        <el-table-column label="角色名称" prop="name" show-overflow-tooltip sortable />
        <el-table-column label="关键字" prop="keyword" show-overflow-tooltip sortable />
        <el-table-column label="等级" prop="sort" show-overflow-tooltip sortable />
        <el-table-column align="center" label="角色状态" prop="status" show-overflow-tooltip sortable>
          <template slot-scope="scope">
            <el-tag :type="scope.row.status === 1 ? 'success':'danger'" disable-transitions size="small">
              {{ scope.row.status === 1 ? '正常' : '禁用' }}
            </el-tag>
          </template>
        </el-table-column>
        <el-table-column label="创建人" prop="creator" show-overflow-tooltip sortable />
        <el-table-column label="说明" prop="desc" show-overflow-tooltip sortable />
        <el-table-column align="center" fixed="right" label="操作" width="140">
          <template slot-scope="scope">
            <el-tooltip content="编辑" effect="dark" placement="top">
              <el-button circle icon="el-icon-edit" size="mini" type="primary" @click="update(scope.row)" />
            </el-tooltip>
            <el-tooltip content="权限" effect="dark" placement="top">
              <el-button circle icon="el-icon-key" size="mini" type="warning" @click="updatePermission(scope.row.ID)" />
            </el-tooltip>
            <el-tooltip content="删除" effect="dark" placement="top">
              <el-popconfirm style="margin-left:10px" title="确定删除吗？" @onConfirm="singleDelete(scope.row.ID)">
                <el-button slot="reference" circle icon="el-icon-delete" size="mini" type="danger" />
              </el-popconfirm>
            </el-tooltip>
          </template>
        </el-table-column>
      </el-table>

      <el-pagination
        :current-page="params.pageNum"
        :page-size="params.pageSize"
        :page-sizes="[1, 5, 10, 30]"
        :total="total"
        background
        layout="total, prev, pager, next, sizes"
        style="margin-top: 10px;float:right;margin-bottom: 10px;"
        @size-change="handleSizeChange"
        @current-change="handleCurrentChange"
      />

      <el-dialog :title="dialogFormTitle" :visible.sync="dialogFormVisible" width="580px">
        <el-form
          ref="dialogForm"
          :inline="true"
          :model="dialogFormData"
          :rules="dialogFormRules"
          label-width="100px"
          size="small"
        >
          <el-form-item label="角色名称" prop="name">
            <el-input v-model.trim="dialogFormData.name" placeholder="角色名称" style="width: 420px" />
          </el-form-item>
          <el-form-item label="关键字" prop="keyword">
            <el-input v-model.trim="dialogFormData.keyword" placeholder="关键字" style="width: 420px" />
          </el-form-item>
          <el-form-item label="角色状态" prop="status">
            <el-select v-model.trim="dialogFormData.status" placeholder="请选择角色状态" style="width: 180px">
              <el-option :value="1" label="正常" />
              <el-option :value="2" label="禁用" />
            </el-select>
          </el-form-item>
          <el-form-item label="等级(1最高)" prop="sort">
            <el-input-number v-model.number="dialogFormData.sort" :max="999" :min="1" controls-position="right" />
          </el-form-item>
          <el-form-item label="说明" prop="desc">
            <el-input
              v-model.trim="dialogFormData.desc"
              maxlength="100"
              placeholder="说明"
              show-word-limit
              style="width: 420px"
              type="textarea"
            />
          </el-form-item>
        </el-form>
        <div slot="footer">
          <el-button size="mini" @click="cancelForm()">取 消</el-button>
          <el-button :loading="submitLoading" size="mini" type="primary" @click="submitForm()">确 定</el-button>
        </div>
      </el-dialog>

      <el-dialog :visible.sync="permsDialogVisible" custom-class="perms-dialog" title="修改权限" width="580px">
        <el-tabs>
          <el-tab-pane>
            <span slot="label"><svg-icon class-name="role-menu" icon-class="menu1" /> 角色菜单</span>
            <el-tree
              ref="roleMenuTree"
              v-loading="menuTreeLoading"
              :data="menuTree"
              :default-checked-keys="defaultCheckedRoleMenu"
              :props="{children: 'children',label: 'title'}"
              check-strictly
              node-key="ID"
              show-checkbox
            />

          </el-tab-pane>

          <el-tab-pane>
            <span slot="label"><svg-icon class-name="role-menu" icon-class="api1" /> 角色接口</span>
            <el-tree
              ref="roleApiTree"
              v-loading="apiTreeLoading"
              :data="apiTree"
              :default-checked-keys="defaultCheckedRoleApi"
              :props="{children: 'children',label: 'desc'}"
              node-key="ID"
              show-checkbox
            />

          </el-tab-pane>
        </el-tabs>
        <div slot="footer">
          <el-button :loading="permissionLoading" size="mini" @click="cancelPermissionForm()">取 消</el-button>
          <el-button size="mini" type="primary" @click="submitPermissionForm()">确 定</el-button>
        </div>
      </el-dialog>

    </el-card>
  </div>
</template>

<script>
import {
  batchDeleteRoleByIds,
  createRole,
  getRoleApisById,
  getRoleMenusById,
  getRoles,
  updateRoleApisById,
  updateRoleById,
  updateRoleMenusById
} from '@/api/system/role'
import { getMenuTree } from '@/api/system/menu'
import { getApiTree } from '@/api/system/api'

export default {
  name: 'Role',
  data() {
    return {
      // 查询参数
      params: {
        name: '',
        keyword: '',
        status: '',
        pageNum: 1,
        pageSize: 10
      },
      // 表格数据
      tableData: [],
      total: 0,
      loading: false,

      // dialog对话框
      submitLoading: false,
      dialogFormTitle: '',
      dialogType: '',
      dialogFormVisible: false,
      dialogFormData: {
        name: '',
        keyword: '',
        status: 1,
        sort: 999,
        desc: ''
      },
      dialogFormRules: {
        name: [
          { required: true, message: '请输入角色名称', trigger: 'blur' },
          { min: 1, max: 20, message: '长度在 1 到 20 个字符', trigger: 'blur' }
        ],
        keyword: [
          { required: true, message: '请输入关键字', trigger: 'blur' },
          { min: 1, max: 20, message: '长度在 1 到 20 个字符', trigger: 'blur' }
        ],
        status: [
          { required: true, message: '请选择角色状态', trigger: 'change' }
        ],
        desc: [
          { required: false, message: '说明', trigger: 'blur' },
          { min: 0, max: 100, message: '长度在 0 到 100 个字符', trigger: 'blur' }
        ]
      },

      // 删除按钮弹出框
      popoverVisible: false,
      // 表格多选
      multipleSelection: [],

      // 修改权限
      permsDialogVisible: false,
      permissionLoading: false,
      menuTree: [],
      defaultCheckedRoleMenu: [],
      apiTree: [],
      defaultCheckedRoleApi: [],

      // 被修改权限的角色ID
      roleId: 0
    }
  },
  created() {
    this.getTableData()
    this.getMenuTree()
    this.getApiTree()
  },
  methods: {
    // 查询
    search() {
      this.params.pageNum = 1
      this.getTableData()
    },

    // 获取表格数据
    async getTableData() {
      this.loading = true
      try {
        const { data } = await getRoles(this.params)
        this.tableData = data.roles
        this.total = data.total
      } finally {
        this.loading = false
      }
    },

    // 新增
    create() {
      this.dialogFormTitle = '新增角色'
      this.dialogType = 'create'
      this.dialogFormVisible = true
    },

    // 修改
    update(row) {
      this.dialogFormData.ID = row.ID
      this.dialogFormData.name = row.name
      this.dialogFormData.keyword = row.keyword
      this.dialogFormData.sort = row.sort
      this.dialogFormData.status = row.status
      this.dialogFormData.desc = row.desc

      this.dialogFormTitle = '修改角色'
      this.dialogType = 'update'
      this.dialogFormVisible = true
    },

    // 提交表单
    submitForm() {
      this.$refs['dialogForm'].validate(async valid => {
        if (valid) {
          this.submitLoading = true
          let msg = ''
          try {
            if (this.dialogType === 'create') {
              const { message } = await createRole(this.dialogFormData)
              msg = message
            } else {
              const { message } = await updateRoleById(this.dialogFormData.ID, this.dialogFormData)
              msg = message
            }
          } finally {
            this.submitLoading = false
          }

          this.resetForm()
          this.getTableData()
          this.$message({
            showClose: true,
            message: msg,
            type: 'success'
          })
        } else {
          this.$message({
            showClose: true,
            message: '表单校验失败',
            type: 'error'
          })
          return false
        }
      })
    },

    // 提交表单
    cancelForm() {
      this.resetForm()
    },

    resetForm() {
      this.dialogFormVisible = false
      this.$refs['dialogForm'].resetFields()
      this.dialogFormData = {
        name: '',
        keyword: '',
        status: 1,
        sort: 999,
        desc: ''
      }
    },

    // 批量删除
    batchDelete() {
      this.$confirm('此操作将永久删除, 是否继续?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(async res => {
        this.loading = true
        const roleIds = []
        this.multipleSelection.forEach(x => {
          roleIds.push(x.ID)
        })
        let msg = ''
        try {
          const { message } = await batchDeleteRoleByIds({ roleIds: roleIds })
          msg = message
        } finally {
          this.loading = false
        }

        this.getTableData()
        this.$message({
          showClose: true,
          message: msg,
          type: 'success'
        })
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '已取消删除'
        })
      })
    },

    // 表格多选
    handleSelectionChange(val) {
      this.multipleSelection = val
    },

    // 单个删除
    async singleDelete(id) {
      this.loading = true
      let msg = ''
      try {
        const { message } = await batchDeleteRoleByIds({ roleIds: [id] })
        msg = message
      } finally {
        this.loading = false
      }

      this.getTableData()
      this.$message({
        showClose: true,
        message: msg,
        type: 'success'
      })
    },

    // 修改权限按钮
    async updatePermission(roleId) {
      this.roleId = roleId
      this.permsDialogVisible = true
      this.getMenuTree()
      this.getApiTree()
      this.getRoleMenusById(roleId)
      this.getRoleApisById(roleId)
    },

    // 获取菜单树
    async getMenuTree() {
      this.menuTreeLoading = true
      try {
        const { data } = await getMenuTree()
        this.menuTree = data.menuTree
      } finally {
        this.menuTreeLoading = false
      }
    },

    // 获取接口树
    async getApiTree() {
      this.apiTreeLoading = true
      try {
        const { data } = await getApiTree()
        this.apiTree = data.apiTree
      } finally {
        this.apiTreeLoading = false
      }
    },

    // 获取角色的权限菜单
    async getRoleMenusById(roleId) {
      this.permissionLoading = true
      let rseData = []
      try {
        const { data } = await getRoleMenusById(roleId)
        rseData = data
      } finally {
        this.permissionLoading = false
      }

      const menus = rseData.menus
      const ids = []
      menus.forEach(x => {
        ids.push(x.ID)
      })
      this.defaultCheckedRoleMenu = ids
      this.$refs.roleMenuTree.setCheckedKeys(this.defaultCheckedRoleMenu)
    },

    // 获取角色的权限接口
    async getRoleApisById(roleId) {
      this.permissionLoading = true
      let resData = []
      try {
        const { data } = await getRoleApisById(roleId)
        resData = data
      } finally {
        this.permissionLoading = false
      }

      const apis = resData.apis
      const ids = []
      apis.forEach(x => {
        ids.push(x.ID)
      })
      this.defaultCheckedRoleApi = ids
      this.$refs.roleApiTree.setCheckedKeys(this.defaultCheckedRoleApi)
    },

    // 修改角色菜单
    async updateRoleMenusById() {
      this.permissionLoading = true
      let ids = this.$refs.roleMenuTree.getCheckedKeys()
      const idsHalf = this.$refs.roleMenuTree.getHalfCheckedKeys()
      ids = ids.concat(idsHalf)
      ids = [...new Set(ids)]

      try {
        await updateRoleMenusById(this.roleId, { menuIds: ids })
      } finally {
        this.permissionLoading = false
      }

      this.permsDialogVisible = false
      this.$message({
        showClose: true,
        message: '更新角色菜单成功',
        type: 'success'
      })
    },

    // 修改角色接口
    async updateRoleApisById() {
      this.permissionLoading = true
      const ids = this.$refs.roleApiTree.getCheckedKeys(true)
      try {
        await updateRoleApisById(this.roleId, { apiIds: ids })
      } finally {
        this.permissionLoading = false
      }

      this.permsDialogVisible = false
      this.$message({
        showClose: true,
        message: '更新角色接口成功',
        type: 'success'
      })
    },

    // 确定修改角色权限
    submitPermissionForm() {
      this.updateRoleMenusById()
      this.updateRoleApisById()
    },

    // 取消修改角色权限
    cancelPermissionForm() {
      this.permsDialogVisible = false
    },

    // 分页
    handleSizeChange(val) {
      this.params.pageSize = val
      this.getTableData()
    },
    handleCurrentChange(val) {
      this.params.pageNum = val
      this.getTableData()
    }
  }
}
</script>

<style scoped>
.container-card {
  margin: 10px;
}

.role-menu {
  font-size: 15px;
}
</style>

<style lang="scss">
.perms-dialog > .el-dialog__body {
  padding-top: 0;
  padding-bottom: 15px;
}
</style>

