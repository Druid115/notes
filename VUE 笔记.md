### 生命周期

- beforeCreate 和 created
  - beforeCreate 组件创建之前；created 组件创建之后
  - created 中可以操作数据，并且可以实现 vue 数据到页面的影响，应用：发起 Ajax 请求

- beforeMount 和 mounted
  
- beforeMount vue 装载数据到 DOM 前； mounted vue 装载数据到 DOM 之后
  
  - 在 mounted 中的代码执行完毕后，vue才会根据实际的值进行相关 DOM操作。如果希望在某快代码中元素变更或者插入后，依赖这些元素做一些 DOM 操作，可以使用 this.$nextTick()


~~~vue
mounted: function() {
    // VUE不会执行三次
	this.isShow = true;
	this.isShow = false;
	this.isShow = true;

	this.$nextTick(function(){
		this.$refs.input.focus();
	});
}
~~~

- beforeUpdate 和 updated
  
- 当有数据改变才会触发
  
- beforeDestroy 和 destroyed
  
- 对应组件的 v-if 行为
  
- activated 和 deactivated
  
  - 对于组件需要频繁创建销毁的需求，必须使用 <keep-alive> 包裹起来，能提高性能



### API

- Vue.use(插件对象); // 该过程中会注册一些全局组件，及给 vue 或组件对象挂载属性
  - 挂载属性的方式

~~~vue
// 以router对象举例，IE9及以下无法使用
Object.defineProperty(Vue.prototype,'$router',{
	get:function(){
		return this.router对象
	}
})
~~~



### 路由

- $route 路由信息对象，只读对象
- $router 路由操作对象，只写对象
- 路由 mate 元素是对路由规则是否需要权限验证的配置，与 name 属性同级
- 路由钩子 -> 权限控制的函数执行时期，每次路由匹配后，渲染组件到 router-view 之前，使用 router.beforeEach(function(to, from, next){   })