<template>
  <el-dialog
    :title="!dataForm.id ? '新增' : '修改'"
    :close-on-click-modal="false"
    :visible.sync="visible"
  >
    <el-form ref="dataForm" :model="dataForm" :rules="dataRule" label-width="100px" @keyup.enter.native="dataFormSubmit()">
      <el-form-item label="类型：" prop="type">
        <el-radio-group v-model="dataForm.type">
          <el-radio v-for="(type, index) in dataForm.typeList" :key="index" :label="index">{{ type }}</el-radio>
        </el-radio-group>
      </el-form-item>
      <el-form-item :label="dataForm.typeList[dataForm.type] + '名称：'" prop="name">
        <el-input v-model="dataForm.name" :placeholder="dataForm.typeList[dataForm.type] + '名称'" />
      </el-form-item>
      <el-form-item label="上级菜单：" prop="parentName">
        <el-popover
          ref="menuListPopover"
          placement="bottom-start"
          trigger="click"
        >
          <el-tree
            ref="menuListTree"
            :data="menuList"
            :props="menuListTreeProps"
            node-key="menuId"
            :default-expand-all="true"
            :highlight-current="true"
            :expand-on-click-node="false"
            @current-change="menuListTreeCurrentChangeHandle"
          />
        </el-popover>
        <el-input v-model="dataForm.parentName" v-popover:menuListPopover :readonly="true" placeholder="点击选择上级菜单" class="menu-list__input" />
      </el-form-item>
      <el-form-item v-if="dataForm.type === 1" label="菜单路由：" prop="path">
        <el-input v-model="dataForm.path" placeholder="菜单路由" />
      </el-form-item>
      <el-form-item v-if="dataForm.type === 1" label="菜单组件：" prop="component">
        <el-input v-model="dataForm.component" placeholder="菜单组件" />
      </el-form-item>
      <el-form-item v-if="dataForm.type !== 0" label="授权标识：" prop="perms">
        <el-input v-model="dataForm.perms" placeholder="多个用逗号分隔, 如: user:list,user:create" />
      </el-form-item>
      <el-form-item v-if="dataForm.type !== 2" label="排序号：" prop="orderNum">
        <el-input-number v-model="dataForm.orderNum" controls-position="right" :min="0" label="排序号" />
      </el-form-item>
      <el-form-item v-if="dataForm.type !== 2" label="菜单图标：" prop="icon">
        <el-row>
          <el-col :span="20">
            <el-popover
              ref="iconListPopover"
              placement="bottom-start"
              trigger="click"
              popper-class="mod-menu__icon-popover"
            >
              <div class="mod-menu__icon-inner">
                <div class="mod-menu__icon-list">
                  <el-button
                    v-for="(item, index) in iconList"
                    :key="index"
                    :class="{ 'is-active': item === dataForm.icon }"
                    @click="iconActiveHandle(item)"
                  >
                    <svg-icon :icon-class="item" />
                    <!--<icon-svg :name="item" />-->
                  </el-button>
                </div>
              </div>
            </el-popover>
            <el-input v-model="dataForm.icon" v-popover:iconListPopover :readonly="true" placeholder="菜单图标名称" class="icon-list__input" />
          </el-col>
          <el-col :span="2" class="icon-list__tips">
            <el-tooltip placement="top" effect="light">
              <div slot="content">全站推荐使用SVG Sprite, 详细请点击<a style="color: #4A9FF9;" href="https://panjiachen.github.io/vue-element-admin-site/feature/component/svg-icon.html#usage" target="_blank">参考文档</a></div>
              <i class="el-icon-warning" />
            </el-tooltip>
          </el-col>
        </el-row>
      </el-form-item>
    </el-form>
    <span slot="footer" class="dialog-footer">
      <el-button @click="visible = false">取消</el-button>
      <el-button type="primary" @click="dataFormSubmit()">确定</el-button>
    </span>
  </el-dialog>
</template>

<script>
import { listToTree } from '@/utils'
import svgIcons from '@/views/icons/svg-icons'
export default {
  data() {
    var validateUrl = (rule, value, callback) => {
      if (this.dataForm.type === 1 && !/\S/.test(value)) {
        callback(new Error('菜单URL不能为空'))
      } else {
        callback()
      }
    }
    return {
      visible: false,
      dataForm: {
        id: 0,
        type: 1,
        typeList: ['目录', '菜单', '按钮'],
        name: '',
        parentId: 0,
        parentName: '',
        path: '',
        component: '',
        perms: '',
        orderNum: 0,
        icon: '',
        iconList: []
      },
      dataRule: {
        name: [
          { required: true, message: '菜单名称不能为空', trigger: 'blur' }
        ],
        parentName: [
          { required: true, message: '上级菜单不能为空', trigger: 'change' }
        ],
        url: [
          { validator: validateUrl, trigger: 'blur' }
        ]
      },
      menuList: [],
      menuListTreeProps: {
        label: 'name',
        children: 'children'
      }
    }
  },
  created() {
    this.iconList = svgIcons// Icon.getNameList()
    console.log(this.iconList)
  },
  methods: {
    init(id) {
      this.dataForm.id = id || 0
      this.$http({
        url: '/sys/menu/select',
        method: 'get'
      }).then(({ data }) => {
        this.menuList = listToTree(data, 'menuId')
      }).then(() => {
        this.visible = true
        this.$nextTick(() => {
          this.$refs['dataForm'].resetFields()
        })
      }).then(() => {
        if (!this.dataForm.id) {
          // 新增
          this.menuListTreeSetCurrentNode()
        } else {
          // 修改
          this.$http({
            url: `/sys/menu/info/${this.dataForm.id}`,
            method: 'get'
          }).then(({ data }) => {
            this.dataForm.id = data.menuId
            this.dataForm.type = data.type
            this.dataForm.name = data.name
            this.dataForm.parentId = data.parentId
            this.dataForm.path = data.path
            this.dataForm.component = data.component
            this.dataForm.perms = data.perms
            this.dataForm.orderNum = data.orderNum
            this.dataForm.icon = data.icon
            this.menuListTreeSetCurrentNode()
          })
        }
      })
    },
    // 菜单树选中
    menuListTreeCurrentChangeHandle(data, node) {
      this.dataForm.parentId = data.menuId
      this.dataForm.parentName = data.name
    },
    // 菜单树设置当前选中节点
    menuListTreeSetCurrentNode() {
      this.$refs.menuListTree.setCurrentKey(this.dataForm.parentId)
      this.dataForm.parentName = (this.$refs.menuListTree.getCurrentNode() || {})['name']
    },
    // 图标选中
    iconActiveHandle(iconName) {
      this.dataForm.icon = iconName
    },
    // 表单提交
    dataFormSubmit() {
      this.$refs['dataForm'].validate((valid) => {
        if (valid) {
          this.$http({
            url: `/sys/menu/${!this.dataForm.id ? 'save' : 'update'}`,
            method: 'post',
            data: {
              'menuId': this.dataForm.id || undefined,
              'type': this.dataForm.type,
              'name': this.dataForm.name,
              'parentId': this.dataForm.parentId,
              'path': this.dataForm.path,
              'component': this.dataForm.component,
              'perms': this.dataForm.perms,
              'orderNum': this.dataForm.orderNum,
              'icon': this.dataForm.icon
            }
          }).then(({ msg }) => {
            this.$message({
              message: msg || '操作成功',
              type: 'success',
              duration: 1500,
              onClose: () => {
                this.visible = false
                this.$emit('refreshDataList')
              }
            })
          })
        }
      })
    }
  }
}
</script>

<style lang="scss">
  .mod-menu {
    .menu-list__input,
    .icon-list__input {
       > .el-input__inner {
        cursor: pointer;
      }
    }
    &__icon-popover {
      width: 458px;
      overflow: hidden;
    }
    &__icon-inner {
      width: 478px;
      max-height: 258px;
      overflow-x: hidden;
      overflow-y: auto;
    }
    &__icon-list {
      width: 458px;
      padding: 0;
      margin: -8px 0 0 -8px;
      > .el-button {
        padding: 8px;
        margin: 8px 0 0 8px;
        > span {
          display: inline-block;
          vertical-align: middle;
          width: 18px;
          height: 18px;
          font-size: 18px;
        }
      }
    }
    .icon-list__tips {
      font-size: 18px;
      text-align: center;
      color: #e6a23c;
      cursor: pointer;
    }
  }
</style>
