<template>
  <div class="app-container">
    <el-form :model="queryParams" ref="queryForm" v-show="showSearch" :inline="true">
      <el-form-item label="角色名稱" prop="roleName">
        <el-input
          v-model="queryParams.roleName"
          placeholder="請輸入角色名稱"
          clearable
          size="small"
          style="width: 240px"
          @keyup.enter.native="handleQuery"
        />
      </el-form-item>
      <el-form-item label="權限字元" prop="roleKey">
        <el-input
          v-model="queryParams.roleKey"
          placeholder="請輸入權限字元"
          clearable
          size="small"
          style="width: 240px"
          @keyup.enter.native="handleQuery"
        />
      </el-form-item>
      <el-form-item label="狀態" prop="status">
        <el-select
          v-model="queryParams.status"
          placeholder="角色狀態"
          clearable
          size="small"
          style="width: 240px"
        >
          <el-option
            v-for="dict in statusOptions"
            :key="dict.dictValue"
            :label="dict.dictLabel"
            :value="dict.dictValue"
          />
        </el-select>
      </el-form-item>
      <el-form-item label="創建時間">
        <el-date-picker
          v-model="dateRange"
          size="small"
          style="width: 240px"
          value-format="yyyy-MM-dd"
          type="daterange"
          range-separator="-"
          start-placeholder="開始日期"
          end-placeholder="結束日期"
        ></el-date-picker>
      </el-form-item>
      <el-form-item>
        <el-button type="cyan" icon="el-icon-search" size="mini" @click="handleQuery">搜索</el-button>
        <el-button icon="el-icon-refresh" size="mini" @click="resetQuery">重設</el-button>
      </el-form-item>
    </el-form>

    <el-row :gutter="10" class="mb8">
      <el-col :span="1.5">
        <el-button
          type="primary"
          icon="el-icon-plus"
          size="mini"
          @click="handleAdd"
          v-hasPermi="['system:role:add']"
        >新增</el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button
          type="success"
          icon="el-icon-edit"
          size="mini"
          :disabled="single"
          @click="handleUpdate"
          v-hasPermi="['system:role:edit']"
        >修改</el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button
          type="danger"
          icon="el-icon-delete"
          size="mini"
          :disabled="multiple"
          @click="handleDelete"
          v-hasPermi="['system:role:remove']"
        >刪除</el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button
          type="warning"
          icon="el-icon-download"
          size="mini"
          @click="handleExport"
          v-hasPermi="['system:role:export']"
        >導出</el-button>
      </el-col>
      <right-toolbar :showSearch.sync="showSearch" @queryTable="getList"></right-toolbar>
    </el-row>

    <el-table v-loading="loading" :data="roleList" @selection-change="handleSelectionChange">
      <el-table-column type="selection" width="55" align="center" />
      <el-table-column label="角色編號" prop="roleId" width="120" />
      <el-table-column label="角色名稱" prop="roleName" :show-overflow-tooltip="true" width="150" />
      <el-table-column label="權限字元" prop="roleKey" :show-overflow-tooltip="true" width="150" />
      <el-table-column label="顯示順序" prop="roleSort" width="100" />
      <el-table-column label="狀態" align="center" width="100">
        <template slot-scope="scope">
          <el-switch
            v-model="scope.row.status"
            active-value="0"
            inactive-value="1"
            @change="handleStatusChange(scope.row)"
          ></el-switch>
        </template>
      </el-table-column>
      <el-table-column label="創建時間" align="center" prop="createTime" width="180">
        <template slot-scope="scope">
          <span>{{ parseTime(scope.row.createTime) }}</span>
        </template>
      </el-table-column>
      <el-table-column label="操作" align="center" class-name="small-padding fixed-width">
        <template slot-scope="scope">
          <el-button
            size="mini"
            type="text"
            icon="el-icon-edit"
            @click="handleUpdate(scope.row)"
            v-hasPermi="['system:role:edit']"
          >修改</el-button>
          <el-button
            size="mini"
            type="text"
            icon="el-icon-circle-check"
            @click="handleDataScope(scope.row)"
            v-hasPermi="['system:role:edit']"
          >數據權限</el-button>
          <el-button
            size="mini"
            type="text"
            icon="el-icon-delete"
            @click="handleDelete(scope.row)"
            v-hasPermi="['system:role:remove']"
          >刪除</el-button>
        </template>
      </el-table-column>
    </el-table>

    <pagination
      v-show="total>0"
      :total="total"
      :page.sync="queryParams.pageNum"
      :limit.sync="queryParams.pageSize"
      @pagination="getList"
    />

    <!-- 添加或修改角色配置對話框 -->
    <el-dialog :title="title" :visible.sync="open" width="500px" append-to-body>
      <el-form ref="form" :model="form" :rules="rules" label-width="80px">
        <el-form-item label="角色名稱" prop="roleName">
          <el-input v-model="form.roleName" placeholder="請輸入角色名稱" />
        </el-form-item>
        <el-form-item label="權限字元" prop="roleKey">
          <el-input v-model="form.roleKey" placeholder="請輸入權限字元" />
        </el-form-item>
        <el-form-item label="角色順序" prop="roleSort">
          <el-input-number v-model="form.roleSort" controls-position="right" :min="0" />
        </el-form-item>
        <el-form-item label="狀態">
          <el-radio-group v-model="form.status">
            <el-radio
              v-for="dict in statusOptions"
              :key="dict.dictValue"
              :label="dict.dictValue"
            >{{dict.dictLabel}}</el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="菜單權限">
          <el-checkbox v-model="menuExpand" @change="handleCheckedTreeExpand($event, 'menu')">展開/摺疊</el-checkbox>
          <el-checkbox v-model="menuNodeAll" @change="handleCheckedTreeNodeAll($event, 'menu')">全選/全不選</el-checkbox>
          <el-checkbox v-model="form.menuCheckStrictly" @change="handleCheckedTreeConnect($event, 'menu')">父子同步</el-checkbox>
          <el-tree
            class="tree-border"
            :data="menuOptions"
            show-checkbox
            ref="menu"
            node-key="id"
            :check-strictly="!form.menuCheckStrictly"
            empty-text="載入中，請稍後"
            :props="defaultProps"
          ></el-tree>
        </el-form-item>
        <el-form-item label="備註">
          <el-input v-model="form.remark" type="textarea" placeholder="請輸入內容"></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="primary" @click="submitForm">確 定</el-button>
        <el-button @click="cancel">取 消</el-button>
      </div>
    </el-dialog>

    <!-- 分配角色數據權限對話框 -->
    <el-dialog :title="title" :visible.sync="openDataScope" width="500px" append-to-body>
      <el-form :model="form" label-width="80px">
        <el-form-item label="角色名稱">
          <el-input v-model="form.roleName" :disabled="true" />
        </el-form-item>
        <el-form-item label="權限字元">
          <el-input v-model="form.roleKey" :disabled="true" />
        </el-form-item>
        <el-form-item label="權限範圍">
          <el-select v-model="form.dataScope">
            <el-option
              v-for="item in dataScopeOptions"
              :key="item.value"
              :label="item.label"
              :value="item.value"
            ></el-option>
          </el-select>
        </el-form-item>
        <el-form-item label="數據權限" v-show="form.dataScope == 2">
          <el-checkbox v-model="deptExpand" @change="handleCheckedTreeExpand($event, 'dept')">展開/摺疊</el-checkbox>
          <el-checkbox v-model="deptNodeAll" @change="handleCheckedTreeNodeAll($event, 'dept')">全選/全不選</el-checkbox>
          <el-checkbox v-model="form.deptCheckStrictly" @change="handleCheckedTreeConnect($event, 'dept')">父子同步</el-checkbox>
          <el-tree
            class="tree-border"
            :data="deptOptions"
            show-checkbox
            default-expand-all
            ref="dept"
            node-key="id"
            :check-strictly="!form.deptCheckStrictly"
            empty-text="載入中，請稍後"
            :props="defaultProps"
          ></el-tree>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="primary" @click="submitDataScope">確 定</el-button>
        <el-button @click="cancelDataScope">取 消</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import { listRole, getRole, delRole, addRole, updateRole, exportRole, dataScope, changeRoleStatus } from "@/api/system/role";
import { treeselect as menuTreeselect, roleMenuTreeselect } from "@/api/system/menu";
import { treeselect as deptTreeselect, roleDeptTreeselect } from "@/api/system/dept";

export default {
  name: "Role",
  data() {
    return {
      // 遮罩層
      loading: true,
      // 選中數組
      ids: [],
      // 非單個禁用
      single: true,
      // 非多個禁用
      multiple: true,
      // 顯示搜索條件
      showSearch: true,
      // 總條數
      total: 0,
      // 角色表格數據
      roleList: [],
      // 彈出層標題
      title: "",
      // 是否顯示彈出層
      open: false,
      // 是否顯示彈出層（數據權限）
      openDataScope: false,
      menuExpand: false,
      menuNodeAll: false,
      deptExpand: true,
      deptNodeAll: false,
      // 日期範圍
      dateRange: [],
      // 狀態數據字典
      statusOptions: [],
      // 數據範圍選項
      dataScopeOptions: [
        {
          value: "1",
          label: "全部數據權限"
        },
        {
          value: "2",
          label: "自定數據權限"
        },
        {
          value: "3",
          label: "本部門數據權限"
        },
        {
          value: "4",
          label: "本部門及以下數據權限"
        },
        {
          value: "5",
          label: "僅本人數據權限"
        }
      ],
      // 菜單列表
      menuOptions: [],
      // 部門列表
      deptOptions: [],
      // 查詢參數
      queryParams: {
        pageNum: 1,
        pageSize: 10,
        roleName: undefined,
        roleKey: undefined,
        status: undefined
      },
      // 表單參數
      form: {},
      defaultProps: {
        children: "children",
        label: "label"
      },
      // 表單校驗
      rules: {
        roleName: [
          { required: true, message: "角色名稱不能為空", trigger: "blur" }
        ],
        roleKey: [
          { required: true, message: "權限字元不能為空", trigger: "blur" }
        ],
        roleSort: [
          { required: true, message: "角色順序不能為空", trigger: "blur" }
        ]
      }
    };
  },
  created() {
    this.getList();
    this.getDicts("sys_normal_disable").then(response => {
      this.statusOptions = response.data;
    });
  },
  methods: {
    /** 查詢角色列表 */
    getList() {
      this.loading = true;
      listRole(this.addDateRange(this.queryParams, this.dateRange)).then(
        response => {
          this.roleList = response.rows;
          this.total = response.total;
          this.loading = false;
        }
      );
    },
    /** 查詢菜單樹結構 */
    getMenuTreeselect() {
      menuTreeselect().then(response => {
        this.menuOptions = response.data;
      });
    },
    /** 查詢部門樹結構 */
    getDeptTreeselect() {
      deptTreeselect().then(response => {
        this.deptOptions = response.data;
      });
    },
    // 所有菜單節點數據
    getMenuAllCheckedKeys() {
      // 目前被選中的菜單節點
      let checkedKeys = this.$refs.menu.getCheckedKeys();
      // 半選中的菜單節點
      let halfCheckedKeys = this.$refs.menu.getHalfCheckedKeys();
      checkedKeys.unshift.apply(checkedKeys, halfCheckedKeys);
      return checkedKeys;
    },
    // 所有部門節點數據
    getDeptAllCheckedKeys() {
      // 目前被選中的部門節點
      let checkedKeys = this.$refs.dept.getCheckedKeys();
      // 半選中的部門節點
      let halfCheckedKeys = this.$refs.dept.getHalfCheckedKeys();
      checkedKeys.unshift.apply(checkedKeys, halfCheckedKeys);
      return checkedKeys;
    },
    /** 根據角色ID查詢菜單樹結構 */
    getRoleMenuTreeselect(roleId) {
      return roleMenuTreeselect(roleId).then(response => {
        this.menuOptions = response.menus;
        return response;
      });
    },
    /** 根據角色ID查詢部門樹結構 */
    getRoleDeptTreeselect(roleId) {
      return roleDeptTreeselect(roleId).then(response => {
        this.deptOptions = response.depts;
        return response;
      });
    },
    // 角色狀態修改
    handleStatusChange(row) {
      let text = row.status === "0" ? "啟用" : "停用";
      this.$confirm('確認要"' + text + '""' + row.roleName + '"角色嗎?', "警告", {
          confirmButtonText: "確定",
          cancelButtonText: "取消",
          type: "warning"
        }).then(function() {
          return changeRoleStatus(row.roleId, row.status);
        }).then(() => {
          this.msgSuccess(text + "成功");
        }).catch(function() {
          row.status = row.status === "0" ? "1" : "0";
        });
    },
    // 取消按鈕
    cancel() {
      this.open = false;
      this.reset();
    },
    // 取消按鈕（數據權限）
    cancelDataScope() {
      this.openDataScope = false;
      this.reset();
    },
    // 表單重設
    reset() {
      if (this.$refs.menu != undefined) {
        this.$refs.menu.setCheckedKeys([]);
      }
      this.menuExpand = false,
      this.menuNodeAll = false,
      this.deptExpand = true,
      this.deptNodeAll = false,
      this.form = {
        roleId: undefined,
        roleName: undefined,
        roleKey: undefined,
        roleSort: 0,
        status: "0",
        menuIds: [],
        deptIds: [],
        menuCheckStrictly: true,
        deptCheckStrictly: true,
        remark: undefined
      };
      this.resetForm("form");
    },
    /** 搜索按鈕操作 */
    handleQuery() {
      this.queryParams.pageNum = 1;
      this.getList();
    },
    /** 重設按鈕操作 */
    resetQuery() {
      this.dateRange = [];
      this.resetForm("queryForm");
      this.handleQuery();
    },
    // 多選框選中數據
    handleSelectionChange(selection) {
      this.ids = selection.map(item => item.roleId)
      this.single = selection.length!=1
      this.multiple = !selection.length
    },
    // 樹權限（展開/摺疊）
    handleCheckedTreeExpand(value, type) {
      if (type == 'menu') {
        let treeList = this.menuOptions;
        for (let i = 0; i < treeList.length; i++) {
          this.$refs.menu.store.nodesMap[treeList[i].id].expanded = value;
        }
      } else if (type == 'dept') {
        let treeList = this.deptOptions;
        for (let i = 0; i < treeList.length; i++) {
          this.$refs.dept.store.nodesMap[treeList[i].id].expanded = value;
        }
      }
    },
    // 樹權限（全選/全不選）
    handleCheckedTreeNodeAll(value, type) {
      if (type == 'menu') {
        this.$refs.menu.setCheckedNodes(value ? this.menuOptions: []);
      } else if (type == 'dept') {
        this.$refs.dept.setCheckedNodes(value ? this.deptOptions: []);
      }
    },
    // 樹權限（父子同步）
    handleCheckedTreeConnect(value, type) {
      if (type == 'menu') {
        this.form.menuCheckStrictly = value ? true: false;
      } else if (type == 'dept') {
        this.form.deptCheckStrictly = value ? true: false;
      }
    },
    /** 新增按鈕操作 */
    handleAdd() {
      this.reset();
      this.getMenuTreeselect();
      this.open = true;
      this.title = "添加角色";
    },
    /** 修改按鈕操作 */
    handleUpdate(row) {
      this.reset();
      const roleId = row.roleId || this.ids
      const roleMenu = this.getRoleMenuTreeselect(roleId);
      getRole(roleId).then(response => {
        this.form = response.data;
        this.open = true;
        this.$nextTick(() => {
          roleMenu.then(res => {
            this.$refs.menu.setCheckedKeys(res.checkedKeys);
          });
        });
        this.title = "修改角色";
      });
    },
    /** 分配數據權限操作 */
    handleDataScope(row) {
      this.reset();
      const roleDeptTreeselect = this.getRoleDeptTreeselect(row.roleId);
      getRole(row.roleId).then(response => {
        this.form = response.data;
        this.openDataScope = true;
        this.$nextTick(() => {
          roleDeptTreeselect.then(res => {
            this.$refs.dept.setCheckedKeys(res.checkedKeys);
          });
        });
        this.title = "分配數據權限";
      });
    },
    /** 提交按鈕 */
    submitForm: function() {
      this.$refs["form"].validate(valid => {
        if (valid) {
          if (this.form.roleId != undefined) {
            this.form.menuIds = this.getMenuAllCheckedKeys();
            updateRole(this.form).then(response => {
              this.msgSuccess("修改成功");
              this.open = false;
              this.getList();
            });
          } else {
            this.form.menuIds = this.getMenuAllCheckedKeys();
            addRole(this.form).then(response => {
              this.msgSuccess("新增成功");
              this.open = false;
              this.getList();
            });
          }
        }
      });
    },
    /** 提交按鈕（數據權限） */
    submitDataScope: function() {
      if (this.form.roleId != undefined) {
        this.form.deptIds = this.getDeptAllCheckedKeys();
        dataScope(this.form).then(response => {
          this.msgSuccess("修改成功");
          this.openDataScope = false;
          this.getList();
        });
      }
    },
    /** 刪除按鈕操作 */
    handleDelete(row) {
      const roleIds = row.roleId || this.ids;
      this.$confirm('是否確認刪除角色編號為"' + roleIds + '"的數據項?', "警告", {
          confirmButtonText: "確定",
          cancelButtonText: "取消",
          type: "warning"
        }).then(function() {
          return delRole(roleIds);
        }).then(() => {
          this.getList();
          this.msgSuccess("刪除成功");
        })
    },
    /** 導出按鈕操作 */
    handleExport() {
      const queryParams = this.queryParams;
      this.$confirm('是否確認導出所有角色數據項?', "警告", {
          confirmButtonText: "確定",
          cancelButtonText: "取消",
          type: "warning"
        }).then(function() {
          return exportRole(queryParams);
        }).then(response => {
          this.download(response.msg);
        })
    }
  }
};
</script>