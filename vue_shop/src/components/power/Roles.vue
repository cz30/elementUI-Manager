<template>
  <div>
    <!-- 面包屑导航区域 -->
    <el-breadcrumb separator-class="el-icon-arrow-right">
      <el-breadcrumb-item :to="{ path: '/home' }">首页</el-breadcrumb-item>
      <el-breadcrumb-item>权限管理</el-breadcrumb-item>
      <el-breadcrumb-item>角色列表</el-breadcrumb-item>
    </el-breadcrumb>

    <!-- 卡片视图 -->
    <el-card>
      <!-- 添加角色按钮区域 -->
      <el-row  :gutter="20">
<!--        <el-col :span="8">-->
<!--          <el-input placeholder="请输入内容" v-model="queryInfo.query" clearable @clear="getRolesList">-->
<!--            <el-button slot="append" icon="el-icon-search" @click="getRolesList"></el-button>-->
<!--          </el-input>-->
<!--        </el-col>-->
        <el-col :span="4">
          <el-button type="primary" @click="addRightDialogVisible=true">添加角色</el-button>
        </el-col>
      </el-row>

      <!-- 角色列表区域 -->
      <el-table :data="rolelist" border stripe>
        <!-- 展开列 -->
        <el-table-column type="expand">
          <template slot-scope="scope">
            <el-row :class="['bdbottom', i1 === 0 ? 'bdtop' : '', 'vcenter']" v-for="(item1, i1) in scope.row.children"
                    :key="item1.id">
              <!-- 渲染一级权限 -->
              <el-col :span="5">
                <el-tag closable @close="removeRightById(scope.row, item1.id)">{{item1.authName}}</el-tag>
                <i class="el-icon-caret-right"></i>
              </el-col>
              <!-- 渲染二级和三级权限 -->
              <el-col :span="19">
                <!-- 通过 for 循环 嵌套渲染二级权限 -->
                <el-row :class="[i2 === 0 ? '' : 'bdtop', 'vcenter']" v-for="(item2, i2) in item1.children"
                        :key="item2.id">
                  <el-col :span="6">
                    <el-tag type="success" closable @close="removeRightById(scope.row, item2.id)">{{item2.authName}}
                    </el-tag>
                    <i class="el-icon-caret-right"></i>
                  </el-col>
                  <el-col :span="18">
                    <el-tag type="warning" v-for="item3 in item2.children" :key="item3.id" closable
                            @close="removeRightById(scope.row, item3.id)">{{item3.authName}}
                    </el-tag>
                  </el-col>
                </el-row>
              </el-col>
            </el-row>

            <!-- <pre>
              {{scope.row}}
            </pre> -->
          </template>
        </el-table-column>
        <!-- 索引列 -->
        <el-table-column type="index"></el-table-column>
        <el-table-column label="角色名称" prop="roleName"></el-table-column>
        <el-table-column label="角色描述" prop="roleDesc"></el-table-column>
        <el-table-column label="操作" width="300px">
          <template slot-scope="scope">
            <el-button size="mini" type="primary" icon="el-icon-edit" @click="showRightEditDialog(scope.row.id)">编辑
            </el-button>
            <el-button size="mini" type="danger" icon="el-icon-delete" @click="removeRightRolesById(scope.row.id)">删除</el-button>
            <el-button size="mini" type="warning" icon="el-icon-setting" @click="showSetRightDialog(scope.row)">分配权限
            </el-button>
          </template>
        </el-table-column>
      </el-table>
    </el-card>
<!--    添加用户对话框-->
    <el-dialog
      title="添加用户"
      :visible.sync="addRightDialogVisible"
      width="50%"
    >
      <el-form :model="addRightForm" :rules="addRightFormRules" ref="addRightFormFormRef" label-width="100px">
        <el-form-item label="角色名称" prop="roleName">
          <el-input v-model="addRightForm.roleName"></el-input>
        </el-form-item>
        <el-form-item label="角色描述" prop="roleDesc">
          <el-input v-model="addRightForm.roleDesc"></el-input>
        </el-form-item>
      </el-form>
    <el-button @click="addRightDialogVisible = false">取 消</el-button>
    <el-button type="primary" @click="addRightRoles">确 定</el-button>
    </el-dialog>
    <!--    编辑角色信息对话框-->
    <el-dialog
      title="编辑信息"
      :visible.sync="editRightDialogVisible"
      width="50%">
      <el-form :model="editRightForm" :rules="editRightFormRules" ref="editRightFormRef" label-width="100px">
        <el-form-item label="角色名称" prop="roleName">
          <el-input v-model="editRightForm.roleName"></el-input>
        </el-form-item>
        <el-form-item label="角色描述" prop="roleDesc">
          <el-input v-model="editRightForm.roleDesc"></el-input>
        </el-form-item>
      </el-form>

      <!--     底部区域-->
      <span slot="footer" class="dialog-footer">
    <el-button @click="editRightDialogVisible = false">取 消</el-button>
    <el-button type="primary" @click="editRightUserInfo">确 定</el-button>
  </span>
    </el-dialog>
    <!-- 分配权限的对话框 -->
    <el-dialog title="分配权限" :visible.sync="setRightDialogVisible" width="50%" @close="setRightDialogClosed">
      <!-- 树形控件 -->
      <el-tree :data="rightslist" :props="treeProps" show-checkbox node-key="id" default-expand-all
               :default-checked-keys="defKeys" ref="treeRef"></el-tree>
      <span slot="footer" class="dialog-footer">
        <el-button @click="setRightDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="allotRights">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
  export default {
    data() {
      return {
        // 获取角色列表的参数对象
        queryInfo: {
          query: '',
          // 当前的页数
          pagenum: 1,
          // 当前每页显示多少条数据
          pagesize: 2
        },
        total:0,
        // 所有角色列表数据
        rolelist: [],
        // 控制分配权限对话框的显示与隐藏
        setRightDialogVisible: false,
        // 所有权限的数据
        rightslist: [],
        // 树形控件的属性绑定对象
        treeProps: {
          label: 'authName',
          children: 'children'
        },
        // 默认选中的节点Id值数组
        defKeys: [],
        // 当前即将分配权限的角色id
        roleId: '',
        // 控制编辑对话框的
        editRightDialogVisible: false,
        //查询到的对象
        editRightForm: {},
        //编辑角色信息规则
        editRightFormRules: {
          roleName: [
            { required: true, message: '请输入角色名称', trigger: 'blur' }

          ],
          roleDesc: [
            { required: true, message: '请输入角色描述', trigger: 'blur' }
          ]
        },
        //添加角色信息对话框
        addRightDialogVisible:false,
        addRightForm:{
          roleName:'',
          roleDesc:'',
        },
      }
    },
    created() {
      this.getRolesList()
    },
    methods: {
      // 获取所有角色的列表
      async getRolesList() {
        const { data: res } = await this.$http.get('roles')

        if (res.meta.status !== 200) {
          return this.$message.error('获取角色列表失败！')
        }

        this.rolelist = res.data
      },
      // 根据Id删除对应的权限
      async removeRightById(role, rightId) {
        // 弹框提示用户是否要删除
        const confirmResult = await this.$confirm(
          '此操作将永久删除该文件, 是否继续?',
          '提示',
          {
            confirmButtonText: '确定',
            cancelButtonText: '取消',
            type: 'warning'
          }
        ).catch(err => err)

        if (confirmResult !== 'confirm') {
          return this.$message.info('取消了删除！')
        }

        const { data: res } = await this.$http.delete(
          `roles/${role.id}/rights/${rightId}`
        )

        if (res.meta.status !== 200) {
          return this.$message.error('删除权限失败！')
        }

        // this.getRolesList()
        role.children = res.data
      },
      // 展示分配权限的对话框
      async showSetRightDialog(role) {
        this.roleId = role.id
        // 获取所有权限的数据
        const { data: res } = await this.$http.get('rights/tree')

        if (res.meta.status !== 200) {
          return this.$message.error('获取权限数据失败！')
        }

        // 把获取到的权限数据保存到 data 中
        this.rightslist = res.data
        console.log(this.rightslist)

        // 递归获取三级节点的Id
        this.getLeafKeys(role, this.defKeys)

        this.setRightDialogVisible = true
      },
      // 通过递归的形式，获取角色下所有三级权限的id，并保存到 defKeys 数组中
      getLeafKeys(node, arr) {
        // 如果当前 node 节点不包含 children 属性，则是三级节点
        if (!node.children) {
          return arr.push(node.id)
        }

        node.children.forEach(item => this.getLeafKeys(item, arr))
      },
      // 监听分配权限对话框的关闭事件
      setRightDialogClosed() {
        this.defKeys = []
      },
      // 点击为角色分配权限
      async allotRights() {
        const keys = [
          ...this.$refs.treeRef.getCheckedKeys(),
          ...this.$refs.treeRef.getHalfCheckedKeys()
        ]

        const idStr = keys.join(',')

        const { data: res } = await this.$http.post(
          `roles/${this.roleId}/rights`,
          { rids: idStr }
        )

        if (res.meta.status !== 200) {
          return this.$message.error('分配权限失败！')
        }

        this.$message.success('分配权限成功！')
        this.getRolesList()
        this.setRightDialogVisible = false
      },
      // 点击编辑角色信息
      async showRightEditDialog(id) {

        const { data: res } = await this.$http.get('roles/' + id)
        if (res.meta.status !== 200) {
          return this.$message.error('获取角色信息失败')
        }
        this.editRightForm = res.data
        this.editRightDialogVisible = true
      },
      //提交修改后的角色信息
      editRightUserInfo() {
        this.$refs.editRightFormRef.validate(async valid => {
          if (!valid) return
          const { data: res } = await this.$http.put(
            'roles/' + this.editRightForm.roleId,
            {
              roleName: this.editRightForm.roleName,
              roleDesc: this.editRightForm.roleDesc
            }
          )

          if (res.meta.status !== 200) {
            return this.$message.error('更新角色信息失败！')
          }

          this.editRightDialogVisible = false
          this.getRolesList()
          this.$message.success('更新角色信息成功')
        })



      },
      //添加角色信息
      addRightRoles(){
        this.$refs.addRightFormFormRef.validate( async  valid => {
          if (!valid)return
          const{data:res}=await this.$http.post('roles',this.addRightForm)
          if (res.meta.status !==201) return this.$message.error('添加角色失败！')

          this.$message.success('添加角色成功')
          this.addRightDialogVisible=false
          this.getRolesList()
        })
      },
      //删除角色信息
      async removeRightRolesById(id){
        const confirmResult = await this.$confirm(
          '此操作将永久删除该用户, 是否继续?',
          '提示',
          {
            confirmButtonText: '确定',
            cancelButtonText: '取消',
            type: 'warning'
          }
        ).catch(err => err)
        // 如果用户确认删除，则返回值为字符串 confirm
        // 如果用户取消了删除，则返回值为字符串 cancel
        // console.log(confirmResult)
        if (confirmResult !=='confirm'){
          return this.$message.info('删除已取消')
        }
        const{data:res}=await this.$http.delete('roles/'+id)
        if (res.meta.status !==200){
          return this.$message.error('删除角色失败')
        }
        this.$message.success('删除角色成功')
        this.getRolesList()

      }
    }
  }
</script>

<style lang="less" scoped>
  .el-tag {
    margin: 7px;
  }

  .bdtop {
    border-top: 1px solid #eee;
  }

  .bdbottom {
    border-bottom: 1px solid #eee;
  }

  .vcenter {
    display: flex;
    align-items: center;
  }
</style>
