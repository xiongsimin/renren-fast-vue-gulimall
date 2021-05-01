<template>
  <div>
    <!-- vue属性中，加了:的是变量，没加的是字面量 -->
    <el-tree
      :data="menu"
      :props="defaultProps"
      :expand-on-click-node="false"
      :show-checkbox="true"
      node-key="catId"
      :default-expanded-keys="expandedKey"
      :draggable="true"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            新增
          </el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">
            编辑
          </el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            删除
          </el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog
      :title="titleName"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category" label-width="80px">
        <el-form-item label="分类名称">
          <el-input v-model="category.name"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
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
  data() {
    return {
      maxLevel: 0,
      titleName: "对话框标题",
      //对话框类型，add为新增，edit为编辑
      dialogType: "",
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        icon: "",
        productUnit: "",
        catId: null,
      },
      menu: [],
      expandedKey: [],
      dialogVisible: false,
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  //监听属性 类似于data概念
  computed: {},
  //监控data中的数据变化
  watch: {},
  //方法集合
  methods: {
    getMenu() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        console.log(data.data);
        this.menu = data.data;
      });
    },
    allowDrop(draggingNode, dropNode, type) {
      //draggingNode 被拖动的当前节点
      //dropNode 目标节点
      //type   [prev,inner,next]
      console.log("allowDrop", draggingNode, dropNode, type);

      //初始化maxLevel的值
      this.maxLevel = 0;
      //1、被拖动的节点以及目标父节点总层数不能大于3
      //1）、计算被拖动的当前节点总层数
      this.countNodeLevel(draggingNode.data);
      let deep = this.maxLevel - draggingNode.data.catLevel + 1;
      console.log("当前正在拖动节点的最大深度：", deep);
      //往内部拖动时计算计算的是目标层级
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      }
      //往前或往后拖计算的是父级
      else {
        return deep + dropNode.parent.level <= 3;
      }
    },
    //计算被拖动的当前节点总层数（递归）
    countNodeLevel(node) {
      //找得到所有子节点，求出最大深度
      if (node.children != null && node.children.length > 0) {
        for (var i = 0; i < node.children.length; i++) {
          if (node.children[i].catLevel > this.maxLevel) {
            this.maxLevel = node.children[i].catLevel;
          }
          this.countNodeLevel(node.children[i]);
        }
      } else {
        if (node.catLevel > this.maxLevel) {
          this.maxLevel = node.catLevel;
        }
      }
    },
    handleDrop(draggingNode, dropNode, dropType, event) {
      console.log("handleDrop：", draggingNode, dropNode, dropType);
    },
    append(data) {
      this.titleName = "新增分类";
      this.dialogType = "add";
      this.dialogVisible = true;
      console.log(data);
      //点击新增的时候，对category的属性进行赋值
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
      //设置默认值
      this.category.name = "";
      this.category.catId = null;
      this.category.icon = "";
      this.category.productUnit = "";
      this.category.sort = 0;
      this.category.showStatus = 1;
    },
    edit(data) {
      this.titleName = "编辑分类";
      this.dialogType = "edit";
      console.log("将要修改的分类", data);
      this.dialogVisible = true;
      //http请求获取当前分类最新数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        //请求成功
        console.log("回显数据：", data);
        //填充后台返回的最新数据
        this.category.name = data.data.name;
        this.category.catId = data.data.catId;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
        this.category.catLevel = data.data.catLevel;
        this.category.sort = data.data.sort;
        this.category.showStatus = data.data.showStatus;
      });
    },
    submitData(data) {
      if (this.dialogType == "add") {
        this.addCategory(data);
      } else if (this.dialogType == "edit") {
        this.editCategory(data);
      }
    },
    remove(node, data) {
      var catIds = [data.catId];
      this.$confirm(`是否删除【${data.name}】菜单`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(catIds, false),
          }).then(({ data }) => {
            this.$message({
              type: "success",
              message: "删除成功!",
            });
            //删除成功后刷新数据
            this.getMenu();
            //刷新数据会导致菜单关闭，需要还原展开状态
            this.expandedKey = [node.parent.data.catId];
          });
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已取消删除",
          });
        });

      console.log(node);
      console.log(data);
    },
    addCategory() {
      console.log("将要添加的数据:", this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          type: "success",
          message: "保存成功!",
        });
        //关闭对话框
        this.dialogVisible = false;
        //刷新菜单
        this.getMenu();
        //展开添加菜单的父菜单
        this.expandedKey = [this.category.parentCid];
      });
    },
    editCategory() {
      var { catId, name, icon, productUnit } = this.category;
      console.log("将要修改的数据：" + this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        this.$message({
          type: "success",
          message: "修改成功!",
        });
        //关闭对话框
        this.dialogVisible = false;
        //刷新菜单
        this.getMenu();
        //展开添加菜单的父菜单
        this.expandedKey = [this.category.parentCid];
      });
    },
  },
  //生命周期 - 创建完成（可以访问当前this实例）
  created() {
    this.getMenu();
  },
  //生命周期 - 挂载完成（可以访问DOM元素）
  mounted() {},
  beforeCreate() {}, //生命周期 - 创建之前
  beforeMount() {}, //生命周期 - 挂载之前
  beforeUpdate() {}, //生命周期 - 更新之前
  updated() {}, //生命周期 - 更新之后
  beforeDestroy() {}, //生命周期 - 销毁之前
  destroyed() {}, //生命周期 - 销毁完成
  activated() {}, //如果页面有keep-alive缓存功能，这个函数会触发
};
</script>
<style scoped>
</style>