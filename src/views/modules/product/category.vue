<template>
<div>
  <el-switch
  v-model="draggable"
  active-text="开启拖拽"
  inactive-text="关闭拖拽">
</el-switch>
<el-button v-if="draggable" @click="batchSave">批量保存</el-button>
<el-button type="danger" @click="batchDelete">批量删除</el-button>
  <el-tree  :data="menus"  :props="defaultProps"    
        :expand-on-click-node="false" 
        show-checkbox="true" 
        node-key="catId" 
        :default-expanded-keys="expandedKey"
        :draggable="draggable"
        :allow-drop="allowDrop"
        @node-drop="handleDrop"
        ref="menuTree">
      <span class="custom-tree-node" slot-scope="{ node, data }">
          <span>{{ node.label }}</span>
          <span>
            <el-button
            v-if="node.level<=2" 
              type="text"
              size="mini"
              @click="() => append(data)">
              Append
            </el-button>
            <el-button
          
              type="text"
              size="mini"
              @click="() => edit(data)">
              Edit
            </el-button>
            <el-button
            v-if="node.childNodes.length==0"
              type="text"
              size="mini"
              @click="() => remove(node, data)">
              Delete
            </el-button>
          </span>
      </span>
  </el-tree>
  <el-dialog
    :title= "title"
    :visible.sync="dialogVisible"
    width="30%"
    :close-on-click-modal="false">
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
      <el-button type="primary" @click="submitCategory">确 定</el-button>
    </span>
  </el-dialog>
</div>
</template>

<script>
//这里可以导入其他文件（比如：组件，工具 js，第三方插件 js，json 文件，图片文件等等）
//例如：import 《组件名称》 from '《组件路径》';

export default {
//import 引入的组件需要注入到对象中才能使用
components: {},
props: {},

//计算属性 类似于 data 概念
computed: {},
//监控 data 中的数据变化
watch: {},
data() {
      return {
        pCid:[],
        draggable: false,
        updateNodes: [],
        maxLevel: 0,
        title: "",
        dialogType: "",
        category: {name:"",parentCid: 0,catLevel: 0,showStatus: 1,sort: 0,catId: null,icon: "",productUnit: ""},
        dialogVisible: false,
        menus: [],
        expandedKey: [],
        defaultProps: {
          children: 'children',
          label: 'name'
        }
      };
    },
    methods: {
    
      getMenus(){
        this.$http({
          url: this.$http.adornUrl('/product/category/list/tree'),
          method: 'get'
        }).then(({data})=>{
            console.log("呈贡获取到菜单数据。。。",data.data)
            this.menus = data.data;
        })
      },
      batchDelete(){
        let catIds=[];
        let checkNodes=this.$refs.menuTree.getCheckedNodes();
        console.info("被选中的元素",checkNodes);
        for(let i=0;i<checkNodes.length;i++){
          catIds.push(checkNodes[i].catId);
        }
        this.$confirm(`是否批量删除【${catIds}】菜单-------?`, '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(()=>{

            this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(catIds, false)
            }).then(({ data }) => { 
              this.$message({
              message: '菜单批量删除成功！',
              type: 'success'
              });   
              //刷新出新的菜单
              this.getMenus();         
            });



           
        }).catch(()=>{

        });
        
      },
      batchSave(){
        this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updateNodes, false)
        }).then(({ data }) => { 
            this.$message({
              message: '菜单顺序修改成功！',
              type: 'success'
              });
            //刷新菜单
              //刷新出新的菜单
            this.getMenus();
            //设置需要默认展开的菜单
            this.expandedKey=pCid;
            this.updateNodes=[];
            this.maxLevel=0;
            // this.pCid=0;

        });

      },
      handleDrop(draggingNode, dropNode, dropType, ev) {
        console.log('tree drop: ', dropNode.label, dropType);
        //1\当前节点的最新的父节点id
        let pCid=0;
        let siblings=null;
        if(dropType=="before" || dropType=="after"){
          pCid=dropNode.parent.data.catId==undefined?0:dropNode.parent.data.catId;
          siblings=dropNode.parent.childNodes;
        }else{
          pCid=dropNode.data.catId;
          siblings=dropNode.childNodes;
        }
        this.pCid.push(pCid);
        //2\当前拖拽节点的最新顺序
        for(let i=0;i<siblings.length;i++){
          if(siblings[i].data.catId==draggingNode.data.catId){
            //如果遍历的是当前正在拖拽的节点
            let catLevel=draggingNode.level;
            if(siblings[i].level!=draggingNode.level){
              //当前节点的层级发生变化
              catLevel=siblings[i].level;
              //修改子节点的层级
              this.updateNodeLevel(siblings[i]);
            }
            this.updateNodes.push({catId:siblings[i].data.catId,sort:i,parentCid:pCid,catLevel:catLevel});
          }else
          {
            this.updateNodes.push({catId:siblings[i].data.catId,sort:i});
          }
        }
        //3当前拖拽节点最新层级
        console.log("updateNodes",this.updateNodes);

      },
      updateNodeLevel(node){
        if(node.childNodes.length>0){
          for(let i=0;i<node.childNodes.length;i++){
            var cNode=node.childNodes[i].data;
            this.updateNodes.push({catId:cNode.catId,catLevel:node.childNodes[i].level});
            this.updateNodeLevel(node.childNodes[i]);

          }
        }
      },


      allowDrop(draggingNode, dropNode, type) {
        //1.被拖动的当前节点以及所在的父节点总层数不能大于3
        //1)被拖动的当前节点总层数
        console.log(draggingNode,dropNode,type);
        this.countNodeLevel(draggingNode);

        let deep=Math.abs(this.maxLevel-draggingNode.level)+1;
        console.log("拖动节点的深度："+deep);
        if(type=="inner"){
          return deep+dropNode.level<=3;
        }else{
          return (deep+dropNode.parent.level)<=3
        }
      },
      countNodeLevel(node){
        //找到所有自节点 求出最大深度
        if(node.childNodes!=null && node.childNodes.length>0){
          for(let i=0;i<node.childNodes.length;i++){
            if(node.childNodes[i].level>this.maxLevel){
              this.maxLevel=node.childNodes[i].level;
            }
            this.countNodeLevel(node.childNodes[i]);
          }
        }

      },
      edit(data){
        console.log("要修改得数据",data);
        this.dialogVisible=true;
        this.dialogType="edit";
        this.title="修改分类";
        //发送请求获取当前节点最新得数据
        this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: 'get',
        params: this.$http.adornParams({})
        }).then(({data}) => {
          //请求成功
          console.log("要回显得数据",data);
            this.category.name=data.data.name;
            this.category.catId=data.data.catId;
            this.category.icon=data.data.icon;
            this.category.productUnit=data.data.productUnit;
            this.category.parentCid=data.data.parentCid;
            this.category.catLevel=data.data.catLevel;
            this.category.showStatus=data.data.showStatus;
            this.category.sort=data.data.sort;
        })








      },
      submitCategory(){
        if(this.dialogType=="add"){
          this.addCategory();
        }

        if(this.dialogType=="edit"){
          this.editCategory();
        }
      },
      //修改三级分类数据
      editCategory(){
        var {catId,name,icon,productUnit} = this.category;
        
        // var data ={catId,name,icon,productUnit};

        this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData({catId,name,icon,productUnit}, false)
        }).then(({ data }) => { 
           this.$message({
              message: '菜单修改成功！',
              type: 'success'
              });

            //关闭页面
            this.dialogVisible=false;
            //刷新出新的菜单
            this.getMenus();
            //设置需要默认展开的菜单
            this.expandedKey=[this.category.parentCid];

        });

      },



      append(data) {
        console.log("append",data);
        this.dialogVisible=true;
        this.category.parentCid=data.catId;
        this.category.catLevel=data.catLevel*1+1;
        this.dialogType="add";
        this.title="添加分类";
        //清空值
        this.category.name="";
        this.category.catId=null;
        this.category.icon="";
        this.category.productUnit="";
        this.category.showStatus=1;
        this.category.sort=0;

      },
//添加3级分类的方法
      addCategory(){
        console.log("提交的三级分类数据",this.category);

        this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
        }).then(({ data }) => {
           this.$message({
              message: '菜单保存成功！',
              type: 'success'
              });

            //关闭页面
            this.dialogVisible=false;
            //刷新出新的菜单
            this.getMenus();
            //设置需要默认展开的菜单
            this.expandedKey=[this.category.parentCid];
         });


      },

      remove(node, data) {
        var ids=[data.catId]
        this.$confirm(`是否删除【${data.name}】菜单-------?`, '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(ids, false)
          }).then(({data}) => {
              this.$message({
              message: '菜单删除成功！',
              type: 'success'
              });
              //刷新出新的菜单
              this.getMenus();
              //设置需要默认展开的菜单
              this.expandedKey=[node.parent.data.catId]
           console.log("删除成功....");
          });


        }).catch(()=>{

        });
        console.log("remove",node,data);
      },
    },
//生命周期 - 创建完成（可以访问当前 this 实例）
created() {
    this.getMenus();
},
//生命周期 - 挂载完成（可以访问 DOM 元素）
mounted() {

},
beforeCreate() {}, //生命周期 - 创建之前
beforeMount() {}, //生命周期 - 挂载之前
beforeUpdate() {}, //生命周期 - 更新之前
updated() {}, //生命周期 - 更新之后
beforeDestroy() {}, //生命周期 - 销毁之前
destroyed() {}, //生命周期 - 销毁完成
activated() {}, //如果页面有 keep-alive 缓存功能，这个函数会触发 
}
</script>
<style lang='scss' scoped>
//@import url(); 引入公共 css 类

</style>