<template>
  <div class="app-container">
    <!--搜索栏 -->
    <el-form ref="adminListQueryForm" :model="adminListQuery" :inline="true">
      <el-form-item label="管理员名称" prop="name">
        <el-input v-model="adminListQuery.name" placeholder="请输入管理员名称" clearable size="small" style="width: 240px" />
      </el-form-item>
      <el-form-item>
        <el-button type="primary" icon="el-icon-search" size="mini" @click="adminListQueryFormSubmit">搜索</el-button>
        <el-button icon="el-icon-refresh" size="mini" @click="adminListQueryFormReset">重置</el-button>
      </el-form-item>
    </el-form>

    <!-- 工具栏 -->
    <el-row :gutter="10" class="mb8">
      <el-col :span="1.5">
        <el-button type="primary" icon="el-icon-plus" size="mini" @click="handleAddClick">新增</el-button>
      </el-col>
    </el-row>

    <!-- 管理员列表 -->
    <el-table v-loading="adminListLoading" :data="adminList" row-key="id">
      <el-table-column prop="username" label="账号" width="200" :show-overflow-tooltip="true" />
      <el-table-column prop="name" label="员工名字" width="200" :show-overflow-tooltip="true" />
      <el-table-column prop="department.name" label="部门" width="200" :show-overflow-tooltip="true" />
      <el-table-column prop="roles" label="角色" width="200" :show-overflow-tooltip="true" />
      <!-- TODO 芋艿：接入数据字典 -->
      <el-table-column prop="status" label="在职状态"width="50" />
      <el-table-column label="创建时间" align="center">
        <template slot-scope="scope">
          <span>{{ scope.row.createTime | parseTime('{y}-{m}-{d}') }}</span>
        </template>
      </el-table-column>
      <el-table-column label="操作">
        <template slot-scope="scope">
          <el-button type="text" size="mini" icon="el-icon-edit" @click="handleUpdateClick(scope.row)">修改</el-button>
          <el-button type="text" size="mini" icon="el-icon-circle-check" @click="handleAssignAdminRoleClick(scope.row)">分配角色</el-button>
          <el-button type="text" size="mini" icon="el-icon-delete" @click="handleDeleteClick(scope.row)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>
    <!-- 管理员列表的分页 -->
    <pagination
      v-show="adminListTotal > 0"
      :total="adminListTotal"
      :page.sync="adminListQuery.pageNo"
      :limit.sync="adminListQuery.pageSize"
      @pagination="getAdminList"
    />

    <!-- 管理员添加与修改表单 -->
    <el-dialog
      v-loading="adminFormLoading"
      :title="adminFormTitle"
      :visible.sync="adminFormVisible"
      width="600px"
      append-to-body
      element-loading-text="提交中..."
      element-loading-spinner="el-icon-loading"
    >
      <el-form ref="adminForm" :model="adminForm" :rules="adminFormRule" label-width="80px">
        <el-row>
          <el-col :span="24">
            <el-form-item label="员工名字" prop="name">
              <el-input v-model="adminForm.name" placeholder="请输入员工名字" />
            </el-form-item>
          </el-col>
          <el-col :span="24">
            <el-form-item label="部门" prop="departmentId">
              <el-input v-model="adminForm.departmentId" placeholder="请输入部门" />
            </el-form-item>
          </el-col>
          <el-col :span="24">
            <el-form-item label="账号" prop="username">
              <el-input v-model="adminForm.username" placeholder="请输入账号" />
            </el-form-item>
          </el-col>
          <el-col :span="24">
            <el-form-item label="密码" prop="password">
              <el-input v-model="adminForm.password" placeholder="请输入密码" />
            </el-form-item>
          </el-col>
        </el-row>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="primary" @click="handleFormSubmit">确 定</el-button>
        <el-button @click="handleFormCancel">取 消</el-button>
      </div>
    </el-dialog>

    <!-- 管理员授权表单 -->
    <el-dialog
      v-loading="assignAdminRoleFormLoading"
      title="分配角色"
      :visible.sync="assignAdminRoleFormVisible"
      width="600px"
      append-to-body
      element-loading-text="提交中..."
      element-loading-spinner="el-icon-loading"
    >
      <el-form ref="assignAdminRoleForm" :model="assignAdminRoleForm" :rules="assignAdminRoleFormRule" label-width="80px">
        <el-row>
          <el-col :span="24">
            <el-form-item label="角色">
              <el-select v-model="assignAdminRoleForm.roleIds" multiple placeholder="请选择角色">
                <el-option v-for="role in assignAdminRoleFormRoleList" :key="role.id" :label="role.name" :value="role.id" />
              </el-select>
            </el-form-item>
          </el-col>
        </el-row>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="primary" @click="handleAssignAdminRoleFormSubmit">确 定</el-button>
        <el-button @click="handleAssignAdminRoleFormCancel">取 消</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import { pageAdmin, createAdmin, updateAdmin, deleteAdmin } from '@/api/admin/admin'
import { listAllRoles } from '@/api/permission/role'
import { listAdminRoles, assignAdminRole } from '@/api/permission/permission'

import Pagination from '@/components/Pagination'

export default {
  name: 'AdminList',
  components: { Pagination },
  data() {
    return {
      // 管理员列表
      adminList: [],
      // 管理员总数
      adminListTotal: 0,
      // 进度条
      adminListLoading: true,
      // 查询条件
      adminListQuery: {
        pageNo: 1,
        pageSize: 10,
        name: undefined
      },

      // 管理员添加与修改表单
      adminForm: {},
      // 校验规则
      adminFormRule: {
        name: [
          { required: true, message: '员工名称不能为空', trigger: 'blur' }
        ],
        departmentId: [
          { required: true, message: '部门不能为空', trigger: 'blur' }
        ],
        username: [
          { required: true, message: '账号不能为空', trigger: 'blur' }
        ],
        password: [
          { required: true, message: '密码不能为空', trigger: 'blur' }
        ]
      },
      // 是否可见
      adminFormVisible: false,
      // 标题
      adminFormTitle: '',
      // 进度条
      adminFormLoading: false,

      // 管理员添加与修改表单
      assignAdminRoleForm: {},
      // 校验规则
      assignAdminRoleFormRule: {},
      // 是否可见
      assignAdminRoleFormVisible: false,
      // 进度条
      assignAdminRoleFormLoading: false,
      // 角色列表
      assignAdminRoleFormRoleList: []
    }
  },
  created() {
    this.getAdminList()
  },
  methods: {
    // 加载管理员列表
    getAdminList() {
      this.adminListLoading = true
      pageAdmin(this.adminListQuery).then(response => {
        // 取消加载中
        this.adminListLoading = false
        // 设置数据
        this.adminList = response.data.list
        this.adminListTotal = response.data.total
      }).catch(() => {
        // 取消加载中
        this.adminListLoading = false
      })
    },
    // 添加弹窗
    handleAddClick(row) {
      this.adminFormVisible = true
      this.adminFormTitle = '添加员工'
      // 重置表单
      this.resetForm('adminForm')
      this.adminFormRule.password[0].required = true
    },
    // 修改弹窗
    handleUpdateClick(row) {
      this.adminFormVisible = true
      this.adminFormTitle = '修改员工'
      // 重置表单
      this.resetForm('adminForm')
      this.adminFormRule.password[0].required = false
      // 设置修改的表单
      this.adminForm = {
        ...row,
        children: undefined // TODO 芋艿：有什么办法剔除非表单的字段
      }
    },
    // 表单提交
    handleFormSubmit() {
      this.$refs['adminForm'].validate(valid => {
        if (!valid) {
          return
        }
        // 设置加载中，避免重复点击
        this.adminFormLoading = true

        // 更新
        if (this.adminForm.id) {
          updateAdmin(this.adminForm).then(() => {
            // 取消加载中
            this.adminFormLoading = false
            // 提示成功
            this.messageSuccess('修改成功')
            // 关闭表单
            this.adminFormVisible = false
            // 重新加载管理员列表
            this.getAdminList()
          }).catch(() => {
            // 取消加载中
            this.adminFormLoading = false
          })
          // 新增
        } else {
          createAdmin(this.adminForm).then(() => {
            // 取消加载中
            this.adminFormLoading = false
            // 提示成功
            this.messageSuccess('新增成功')
            // 关闭表单
            this.adminFormVisible = false
            // 重新加载管理员树
            this.getAdminList()
          }).catch(() => {
            // 取消加载中
            this.adminFormLoading = false
          })
        }
      })
    },
    // 表单取消
    handleFormCancel() {
      this.adminFormVisible = false
    },
    // 删除弹窗
    handleDeleteClick(row) {
      this.$confirm('确认删除名字为"' + row.name + '"的管理员?', '提示', {
        type: 'warning',
        confirmButtonText: '确定',
        cancelButtonText: '取消'
      }).then(() => {
        deleteAdmin(row.id).then(response => {
          // 提示成功
          this.messageSuccess('删除成功')
          // 重新加载管理员列表
          this.getAdminList()
        })
      })
    },
    // 搜索表单提交
    adminListQueryFormSubmit() {
      // 重置到第一页
      this.adminListQuery.pageNo = 1
      // 加载管理员列表
      this.getAdminList()
    },
    // 搜索表单重置
    adminListQueryFormReset() {
      // 重置表单
      this.resetForm('adminListQueryForm')
      // 加载管理员列表
      this.getAdminList()
    },
    // 分配角色弹窗
    handleAssignAdminRoleClick(row) {
      this.assignAdminRoleFormVisible = true
      // 重置表单
      this.resetForm('assignAdminRoleForm')
      // 设置表单
      this.assignAdminRoleForm.adminId = row.id
      // 获得角色列表
      listAllRoles().then(response => {
        this.assignAdminRoleFormRoleList = response.data
      })
      // 设置角色的选中
      listAdminRoles(row.id).then(response => {
        this.assignAdminRoleForm.roleIds = response.data
      })
    },
    // 表单提交
    handleAssignAdminRoleFormSubmit() {
      this.$refs['assignAdminRoleForm'].validate(valid => {
        if (!valid) {
          return
        }
        // 设置加载中，避免重复点击
        this.assignAdminRoleFormLoading = true

        // 更新
        assignAdminRole({
          ...this.assignAdminRoleForm,
          roleIds: this.assignAdminRoleForm.roleIds.join(',')
        }).then(response => {
          // 取消加载中
          this.assignAdminRoleFormLoading = false
          // 提示成功
          this.messageSuccess('分配成功')
          // 关闭表单
          this.assignAdminRoleFormVisible = false
          // 重新加载管理员列表
          this.getAdminList()
        }).catch(() => {
          // 取消加载中
          this.assignAdminRoleFormLoading = false
        })
      })
    },
    // 表单取消
    handleAssignAdminRoleFormCancel() {
      this.assignAdminRoleFormVisible = false
    }
  }
}

</script>

<style scoped>

</style>