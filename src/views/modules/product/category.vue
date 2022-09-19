<!--  -->
<template>
  <div>
    <el-button type="danger" round @click="batchDelete">批量删除</el-button>
    <el-tree
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandeKey"
      :draggable="true"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >Append
          </el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >Delete
          </el-button>
          <el-button
            type="text"
            size="mini"
            @click="() => edit(data)"
          >edit
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog :title="title" :visible.sync="dialogVisible" width="30%"
               :close-on-click-modal="false"
    >
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
        <el-button type="primary" @click="submitData()">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
//这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
//例如：import 《组件名称》 from '《组件路径》';

export default {
  //import引入的组件需要注入到对象中才能使用
  components: {},
  propos: {},
  data () {
    return {
      updateNodes: [],
      maxLevel: 0,
      dialogType: '',//对话框的类型 点击edit打开则为edit 点击append打开则为add
      menus: [],
      category: {
        name: '',
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
        productUnit: '',
        icon: '',
      },
      expandeKey: [],
      defaultProps: {
        children: 'children',
        label: 'name',
      },
      dialogVisible: false,
      title: '',//弹窗信息提示
    }
  },
  methods: {
    handleNodeClick (data) {
      console.log(data)
    },
    //批量删除
    batchDelete: function () {
      let checkNodes = this.$refs.menuTree.getCheckedNodes()
      let catIds = []
      let checkNodesNames=[];
      console.log('被选中的元素:', checkNodes)
      for (let i = 0; i < checkNodes.length; i++) {
        catIds.push(checkNodes[i].catId)
        checkNodesNames.push(checkNodes[i].name)
      }
      this.$confirm(`是否批量删除${checkNodesNames}`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'post',
          data: this.$http.adornData(catIds, false),
        }).then(resp => {
          if (resp.data.code === 0) {
            console.log(resp)
            this.$message({
              type: 'success',
              message: resp.data.msg,
            })
            this.getMenus();
          }
        })
      }).catch(() => {
      })
    },
    handleDrop (draggingNode, dropNode, dropType, ev) {
      //draggingNode当前正在拖拽的节点
      //dropNode进入到哪个节点或者是平移到哪个节点之前或之后
      //dropType进入到节点的什么位置,这些都是ElementUI提供的
      console.log('handleDrop', draggingNode, dropNode, dropType)

      let pCid = 0
      let siblings = null
      //before 与 after 代表着平移,平移可能会移出到与当前节点的父节点相同的层级,也可能是与其兄弟节点交换位置
      //inner代表着将当前节点移入到其他节点之下,而dropType就是代表着这些字符
      //当进行平移时,我们可以从dropNode这个数据中得知
      //1.当前节点最新的父节点id
      if (dropType == 'before' || dropType == 'after') {
        //平移 保持当前节点的父id 与 平移到哪个节点之前或之后这个节点的父id相同即可
        pCid = dropNode.data.parentCid
        siblings = dropNode.parent.childNodes
      } else {
        //移入 :当前节点的父id就是 进入到哪个节点 的Cid
        pCid = dropNode.data.catId
        siblings = dropNode.childNodes
      }
      //2.当前节点的最新顺序
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          //判断当前节点的层级
          let catLevel = draggingNode.level
          if (siblings[i].level != draggingNode.level) {
            //修改拖拽的节点的层级
            catLevel = siblings[i].level
            //修改子节点的层级
            this.updateChildrenNodeLevel(siblings[i])
            // if (dropType == "before" || dropType == "after") {
            //   catLevel=dropNode.level;
            // }else {
            //   catLevel=dropNode.level+1;
            // }
          }
          //如果遍历的是当前拖拽的节点
          this.updateNodes.push({catId: siblings[i].data.catId, sort: i, parentCid: pCid, catLevel: catLevel})
        } else {
          this.updateNodes.push({catId: siblings[i].data.catId, sort: i})
        }
      }
      //3.当前拖拽节点的最新层级
      console.log('updateNodes', this.updateNodes)
      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updateNodes, false),
      }).then(resp => {
        if (resp.data.code === 0) {
          console.log(resp)
          this.$message({
            type: 'success',
            message: resp.data.msg,
          })
          //拖拽成功进行数据的刷新
          this.getMenus()
          //拖拽成功将默认展示的数据设置
          this.expandeKey = [pCid]
          this.updateNodes = []
          this.maxLevel = 0
        }
      })
    },
    updateChildrenNodeLevel (node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data
          this.updateNodes.push({catId: cNode.catId, catLevel: node.childNodes[i].level})
          this.updateChildrenNodeLevel(node.childNodes[i])
        }
      }
    },
    allowDrop (draggingNode, dropNode, type) {
      //1.判断被拖动的当前节点以及所在的父节点总层数不能大于3

      //被拖动的当前节点的总层数
      console.log(draggingNode, dropNode, type)
      this.countNodeLevel(draggingNode.data)
      //当前拖动的节点加上父节点所在的深度不大于3即可
      let deep = (this.maxLevel - draggingNode.data.catLevel) + 1
      if (type == 'inner') {
        return (deep + dropNode.level) <= 3 ? true : false
      } else {
        return (deep + dropNode.parent.level) <= 3 ? true : false
      }
    },
    countNodeLevel (node) {
      //找到所有子节点,求出最大深度
      if (node.children != null && node.children.length > 0) {
        for (let i = 0; i < node.children.length; i++) {
          if (node.children[i].catLevel > this.maxLevel) {
            this.maxLevel = node.children[i].catLevel
          }
          this.countNodeLevel(node.children[i])
        }
      }
    },
    submitData () {
      if (this.dialogType == 'add') {
        this.addCategory()
      }
      if (this.dialogType == 'edit') {
        this.editCategory()
      }
    },
    //修改功能
    editCategory () {
      var {catId, name, icon, productUnit} = this.category
      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData({catId, name, icon, productUnit}, false),
      }).then(resp => {
        if (resp.data.code === 0) {
          console.log(resp)
          this.$message({
            type: 'success',
            message: resp.data.msg,
          })
          //添加成功关闭添加表单
          this.dialogVisible = false
          //添加成功进行数据的刷新
          this.getMenus()
          //添加成功将默认展示的数据设置为点击的数据
          this.expandeKey = [this.category.parentCid]
          //清空表单数据
          this.category.name = ''
        }
      })
    },
    //跳出修改弹窗,此data为该项数据的全部信息
    edit (data) {
      console.log('要修改的数据', data)
      this.dialogType = 'edit'
      this.dialogVisible = true
      this.title = '修改分类'
      //发送请求获取当前节点最新的数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: 'get',
        params: this.$http.adornParams({})
      }).then(resp => {
        if (resp.data.code === 0) {
          console.log(resp)
          //category为表单数据信息;
          this.category.name = resp.data.data.name
          this.category.catId = resp.data.data.catId
          this.category.icon = resp.data.data.icon
          this.category.productUnit = resp.data.data.productUnit
          this.category.parentCid = resp.data.data.parentCid
        }
      })
    },
    //查询菜单数据
    getMenus () {
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get',
      }).then(({data}) => {
        console.log(data.data)
        this.menus = data.data
      })
    }
    ,
    //弹出添加表单
    append (data) {
      console.log(data)
      this.dialogType = 'add'
      this.dialogVisible = true
      this.category.parentCid = data.catId
      this.category.catLevel = data.catLevel * 1 + 1
      this.title = '添加分类'
      //清空表单数据
      this.category.catId = null
      this.category.parentCid = data.catId
      this.category.catLevel = data.catLevel * 1 + 1
      this.category.name = ''
      this.category.icon = ''
      this.category.productUnit = ''
      this.category.sort = 0
    }
    ,

    //进行数据的添加
    addCategory () {
      console.log('提交的三级分类数据', this.category)
      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false),
      }).then((resp) => {
        if (resp.data.code === 0) {
          //console.log(resp);
          this.$message({
            type: 'success',
            message: resp.data.msg,
          })
          //添加成功关闭添加表单
          this.dialogVisible = false
          //添加成功进行数据的刷新
          this.getMenus()
          //添加成功将默认展示的数据设置为点击的数据
          this.expandeKey = [this.category.parentCid]
          //清空表单数据
          this.category.name = ''
        }
      })
    }
    ,
    remove (node, data) {
      var ids = [data.catId]
      this.$confirm(`是否删除【${data.name}】菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning',
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'post',
          data: this.$http.adornData(ids, false),
        }).then((resp) => {
          if (resp.data.code === 0) {
            console.log(resp)
            this.$message({
              type: 'success',
              message: resp.data.msg,
            })
            //刷新出新的菜单
            this.getMenus()
            //设置需要默认展开的菜单
            this.expandeKey = [node.parent.data.catId]

          } else {
            this.$message({
              type: 'error',
              message: resp.data.msg,
            })
          }
        })
      })
        .catch(() => {
        })
      console.log('remove', node, data)
    }
    ,
  },

  //监听属性 类似于data概念
  computed:
    {}
  ,
//监控data中的数据变化
  watch: {}
  ,

//生命周期 - 创建完成（可以访问当前this实例）
  created () {
  }
  ,
//生命周期 - 挂载完成（可以访问DOM元素）
  mounted () {
  }
  ,
  beforeCreate () {
  }
  , //生命周期 - 创建之前
  beforeMount () {
  }
  , //生命周期 - 挂载之前
  beforeUpdate () {
  }
  , //生命周期 - 更新之前
  updated () {
  }
  , //生命周期 - 更新之后
  beforeDestroy () {
  }
  , //生命周期 - 销毁之前
  destroyed () {
  }
  , //生命周期 - 销毁完成
  activated () {
    this.getMenus()
  }
  , //如果页面有keep-alive缓存功能，这个函数会触发
}
</script>
<style lang='scss' scoped>
//@import url(); 引入公共css类
</style>
