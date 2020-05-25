<!--分类管理 -->
<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拓转"
      inactive-text="关闭拖拽">
    </el-switch>
    <el-button v-if="draggable" @click="batchSave">批量保存</el-button>
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <!--   树形菜单   draggable拖拽属性   :allow-drop="allowDrop"能否放置到指定位置  @node-drop="handleDrop"拖拽后的回调函数-->
    <el-tree :data="menus" :props="defaultProps"
             :expand-on-click-node="false"
             show-checkbox
             node-key="catId"
             :draggable="draggable"
             @node-drop="handleDrop"
             :allow-drop="allowDrop"
             :default-expanded-keys="expandedKey"
             ref="menuTree">
      <!--  node-key="id"标识每个节点为唯一-->
      <!--显示append和delete按钮  :expand-on-click-node="false"点击操作时候不让节点伸缩，只有点击箭头时候才会伸缩/展开-->
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
        <!--  新增-->
          <el-button v-if="node.level<=2" type="text" size="mini" @click="() => append(data)"> Append  </el-button>
          <!-- 修改-->
           <el-button type="text" size="mini" @click="edit(data)"> edit  </el-button>
          <!-- 删除-->
          <!--当前节点的子集合为0说明没有子节点，才能够删除-->
          <el-button v-if="node.childNodes.length==0" type="text" size="mini"
                     @click="() => remove(node, data)"> Delete </el-button>
        </span>
      </span>
    </el-tree>

    <!--dialog对话框，用于新增   close-on-click-modal="false"防止点击遮罩关闭对话框-->
    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      :close-on-click-modal="false"
      width="30%">

      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>


      <span slot="footer" class="dialog-footer">
          <el-button @click="dialogVisible = false">取 消</el-button>
        <!--  <el-button type="primary" @click="addCategory">确 定</el-button>-->
          <el-button type="primary" @click="submitData">确 定</el-button>
     </span>
    </el-dialog>
  </div>
</template>

<script>
  export default {
    data() {
      return {
        pCid: [],//拖拽保存应用，存父元素
        draggable: false,//拖拽功能是否开启，默认关闭
        updateNodes: [],//拖拽触发函数，用来存放排序数据
        maxLevel: 0,//当前节点的子节点的最大深度，用于拖拽
        title: '',//弹框显示文字标题
        dialogType: '', //  区分点击确定按钮是新增还是修改 edit,add
        category: {name: '', parentCid: 0, catLevel: 0, showStatus: 1, sort: 0, catId: null, icon: '', productUnit: ''},//表单绑定对象
        menus: [],
        dialogVisible: false,
        expandedKey: [],//里面设置catid，设置哪个，哪个菜单默认就会展开
        defaultProps: {
          children: 'children',
          label: 'name'
        }
      }
    },
    methods: {
      getMenus() {
        this.$http({
          url: this.$http.adornUrl('/product/category/list/tree'),
          method: 'get'
        }).then(({data}) => {
          console.log('成功获取到数据', data.data)
          this.menus = data.data
        })
      },
      //批量删除
      batchDelete() {
        let catIds = []
        let checkedNodes = this.$refs.menuTree.getCheckedNodes();
        for (let i = 0; i < checkedNodes.length; i++) {
          catIds.push(checkedNodes[i].catId)
        }
        this.$confirm(`是否删除【${catIds}】菜单`, '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {//确定
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(catIds, false)
          }).then(({data}) => {
            this.$message({
              message: '菜单批量删除成功',
              type: 'success'
            })
            this.getMenus()
          })
        }).catch(() => {

        })
      },
      //批量保存拖拽
      batchSave() {
        this.$http({
          url: this.$http.adornUrl('/product/category/update/sort'),
          method: 'post',
          data: this.$http.adornData(this.updateNodes, false)
        }).then(({data}) => {
          this.$message({
            message: '菜单顺序等修改成功',
            type: 'success'
          })
          this.getMenus();
          this.expandedKey = this.pCid;
          this.updateNodes = [];
          this.maxLevel = 0
          //this.pCid=0
        })
      },
      /**
       * 拖拽效果的限制条件
       * @param draggingNode  当前拖动的节点
       * @param dropNode     被拖动到到了哪个节点
       * @param type        进入到节点的什么位置
       * @returns {boolean}
       */
      allowDrop(draggingNode, dropNode, type) {
        //1.被拖动的当前节点的总层数
        this.countNodeLevel(draggingNode.data);
        //当前正在拖动的节点的深度（当前节点的子节点的层级数 ）+父节点所在的深度的节点不大于3
        let deep = Math.abs(this.maxLevel - draggingNode.level) + 1
        if (type == "inner") {//拖到菜单里面
          return (deep + dropNode.level) <= 3
        } else {//菜单外面，菜单前后
          return (deep + dropNode.parent.level) <= 3
        }

      },
      //拖拽的回调函数
      handleDrop(draggingNode, dropNode, dropType, ev) {
        console.log('tree drop: ', dropNode.label, dropType);
        //1.当前节点最新的父节点id
        let pCid = 0
        //2.当前拖拽节点的最新顺序
        let siblings = null
        if (dropType == 'before' || dropType == 'after') {//前后兄弟关系拖拽
          pCid = dropNode.parent.data.catId == undefined ? 0 : dropNode.parent.data.catId
          siblings = dropNode.parent.childNodes
        } else {//否则是inner，也就是当前进入元素的id
          pCid = dropNode.data.catId
          //如果是进入的方式inner，那么兄弟元素就是childNodes
          siblings = dropNode.childNodes
        }
        this.pCid.push(pCid)
        //2.当前拖拽节点的最新顺序
        for (let i = 0; i < siblings.length; i++) {
          //如果遍历的是当前正在拖拽的节点,那么还要放入当前的父id
          if (draggingNode.data.catId == siblings[i].data.catId) {
            let catLevel = draggingNode.level
            //3.当前拖拽节点的最新层级
            if (siblings[i].level != draggingNode.level) {
              //如果两个层级不一样，说明当前节点的层级发生了变化
              catLevel = siblings[i].level
              //修改子节点的层级
              this.updateChildNodelevel(siblings[i])
            }
            this.updateNodes.push({catId: siblings[i].data.catId, sort: i, parentCid: pCid, catLevel: catLevel})
          } else {//兄弟元素只改顺序
            this.updateNodes.push({catId: siblings[i].data.catId, sort: i})
          }
        }
        //发送请求，批量修改排序等，，优化，放到了批量保存里面
        /*     this.$http({
               url: this.$http.adornUrl('/product/category/update/sort'),
               method: 'post',
               data: this.$http.adornData(this.updateNodes, false)
             }).then(({data}) => {
               this.$message({
                 message: '菜单顺序等修改成功',
                 type: 'success'
               })
               this.getMenus();
               this.expandedKey=[pCid];
               this.updateNodes=[];
               this.maxLevel=0
             })*/
      },
      //递归修改子节点的层级
      updateChildNodelevel(node) {
        if (node.childNodes.length > 0) {
          for (let i = 0; i < node.childNodes.length; i++) {
            var cNode = node.childNodes[i].data
            this.updateNodes.push({catId: cNode.catId, catLevel: node.childNodes[i].level})
            this.updateChildNodelevel(node.childNodes[i])
          }
        }
      },
      //递归查找计算当前节点的总层数
      countNodeLevel(node) {
        //找到所有子节点，求出最大深度
        if (node.childNodes != null && node.childNodes.length > 0) {
          for (let i = 0; i < node.childNodes.length; i++) {
            if (node.childNodes[i].level > this.maxLevel) {//如果当前节点的深度（第几级菜单）大于存储的深度，就赋值
              this.maxLevel = node.childNodes[i].level
            }
            this.countNodeLevel(node.childNodes[i])
          }
        }
      },
      //点击确定新增/修改的操作
      submitData() {
        if (this.dialogType == 'add') {
          this.addCategory()
        }

        if (this.dialogType == 'edit') {
          this.editCategory()
        }
      },
      //修改三级分类数据
      editCategory() {
        //修改分类id，name，icon，计量单位，只给后台传递这几个参数
        //直接发送到后台，解构写法
        const {catId, name, icon, productUnit} = this.category;
        this.$http({
          url: this.$http.adornUrl('/product/category/update'),
          method: 'post',
          data: this.$http.adornData({catId, name, icon, productUnit}, false)
        }).then(({data}) => {
          this.$message({
            message: '菜单修改成功',
            type: 'success'
          })
          //关闭对话框
          this.dialogVisible = false
          //刷新菜单
          this.getMenus()
          //设置需要展开的菜单,当前节点的父节点的id
          this.expandedKey = [this.category.parentCid]
        })
      },
      //添加分类
      addCategory() {
        this.$http({
          url: this.$http.adornUrl('/product/category/save'),
          method: 'post',
          data: this.$http.adornData(this.category, false)
        }).then(({data}) => {//保存成功
          this.$message({
            message: '菜单保存成功',
            type: 'success'
          })
          //关闭对话框
          this.dialogVisible = false
          //刷新菜单
          this.getMenus()
          //设置需要展开的菜单,当前节点的父节点的id
          this.expandedKey = [this.category.parentCid]
        })
      },
      //追加按钮，出现弹框
      append(data) {
        //区分按钮点击是新增还是修改
        this.dialogType = 'add'
        this.title = '添加分类'
        this.dialogVisible = true
        //当前在哪个节点操作的，就是新增节点的父id
        this.category.parentCid = data.catId
        //当前节点层级+1，*1为了转为数字类型
        this.category.catLevel = data.catLevel * 1 + 1
        //清空编辑时候回显的数据
        this.category.catId = null
        this.category.name = null
        this.category.icon = ""
        this.category.productUnit = ""
        this.category.sort = 0
        this.category.showStatus = 1
      },
      //点击修改 data当前节点的数据
      edit(data) {
        //区分按钮点击是新增还是修改
        this.dialogType = 'edit'
        this.title = '修改分类'
        console.log('要修改的数据', data)
        this.dialogVisible = true

        //发送请求获取当前节点最新的数据
        this.$http({
          url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
          method: 'get',
        }).then(({data}) => {
          //成功
          console.log('selectData', data.data)
          this.category.name = data.data.name
          this.category.catId = data.data.catId
          this.category.icon = data.data.icon
          this.category.productUnit = data.data.productUnit
          this.category.parentCid = data.data.parentCid
        })
      },
      //移除
      remove(node, data) {
        const ids = [data.catId]

        this.$confirm(`是否删除【${data.name}】菜单`, '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {//确定
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(ids, false)
          }).then(({data}) => {
            //console.log("删除成功")
            this.$message({
              message: '菜单删除成功',
              type: 'success'
            })
            //刷新出新的菜单
            this.getMenus()
            //设置需要展开的菜单,当前节点的父节点的id
            console.log("asdfasfdas", this.category.parentCid)
            this.expandedKey = [node.parent.data.catId];
          })
        }).catch(() => {//取消

        })
      },
    },
    // 计算属性 类似于data概念
    computed: {},
    // 监控data中的数据变化
    watch: {},
    // 生命周期 - 创建完成（可以访问当前this实例）
    created() {
      this.getMenus()
    },
    // 生命周期 - 挂载完成（可以访问DOM元素）
    mounted() {
    },
    beforeCreate() {
    }, // 生命周期 - 创建之前
    beforeMount() {
    }, // 生命周期 - 挂载之前
    beforeUpdate() {
    }, // 生命周期 - 更新之前
    updated() {
    }, // 生命周期 - 更新之后
    beforeDestroy() {
    }, // 生命周期 - 销毁之前
    destroyed() {
    }, // 生命周期 - 销毁完成
    activated() {
    } // 如果页面有keep-alive缓存功能，这个函数会触发
  }
</script>
<style>
</style>
