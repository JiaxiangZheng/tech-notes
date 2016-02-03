## Angular 初学笔记

> 注：这里所说的 Angular 指的是 1.x 版本。

按照其官方介绍，Angular 是一个所谓的 **MVW**(Model-View-Whatever) 框架，具有高度的灵活性。至于是否真的如它所说，还有待「考证」。这里主要记录我初学过程中碰到的概念、使用等，希望对 Angular 达到一个感观上的认识。

### 基本概念

先说一下相关的几个概念：

**MVVM**：与传统 MVC 模型最大的不同在于，传统的 MVC 会将重点放在 C 端，即控制器上；但 MVVM 模型弱化了控制器的地位，将更多的注意力集中在了视图上，并抽象出 View Model 概念。MVVM 更易于数据和视图的双向绑定。

**双向绑定**：顾名思义，即数据与视图的双向同步，数据发生改变会引起视图发生对应的变化，而视图绑定对应的数据后，其变化也相应地引起数据的变化。在 Angular 中，使用 `ng-model` 在视图中绑定数据模型。

**依赖注入**：参考其[官方文档](https://docs.angularjs.org/guide/di)，我理解的是将依赖组件（模块）注入到指定的服务中，比如需要 HTTP 服务相关的组件，可以在服务控制器中注入 `$http` 而后直接使用它即可。不过最常见的应该是 `$scope` 注入了。

### 模块

module 在 Angular 中是一个最基础的概念，模块间功能可以彼此独立，也可以依赖于其它模块。

```
// 创建模块（第二个参数表示其依赖的模块名，不可省）
angular.module('app', []);
// 获取模块
angular.module('app');
```

创建模块后，需要在视图中指定其作用域，有两种方式可以达到这个目的：

1. 直接在 HTML 中对应元素上添加 `ng-app="app"` 属性
2. 如果需要异步地绑定模块，可以使用 `angular.bootstrap(domElement, ['app'])`

使用 `ng-app` 的方式对于一个页面只能用一次，如果需要使用多个模块，可以使用 `bootstrap` 的方式手工绑定。

### 控制器

先说一下 `$scope` 这个概念，由于 Angular 中数据与视图会做双向绑定，这个桥梁就需要 `$scope` 来扮演了。此外，`$scope` 提供了数据的作用域概念。比如说，不同的 DOM，我们希望它们各自的数据不会彼此污染，`$scope` 在控制器中扮演的就是这么一个角色。下面这段代码中，`title` 就可以通过在 `demo-controller` 控制器里进行处理（`$scope.title = 'demo title'`），进而可以引起视图的修改。

```html
<div ng-controller="demo-controller">
	<h3>{{title}}</h3>
</div>
```

另外，Angular 中定义 `ng-app` 就自动提供了 `$rootScope`，它相当于一个全局的作用域。一般情况下我们更希望不同的组件数据间能够有效地隔离，这个时候使用 controller 就能有效地隔离作用域了。**对于不同作用域之间有数据的通信，不推荐直接将数据绑定到全局，尽量使用 service 的方式。**

控制器依托于模块而存在，其创建方式非常简单如下：

```javascript
angular
  .module('app', [])
  .controller('AppCtrl', function ($scope) {
  	// 控制器主体
  
  })
```

注意，由于控制器之间是可以有层次关系的，对于父控制器，其 `$scope` 数据自动能够在子控制器作用域中使用，这本质上是通过原型继承的方式实现的，当使用作用域下变量的时候，如果当前实例没有则通过原型链依次查找。

事实上，`$scope` 在控制器中扮演的角色非常类似于一个类中的 `this`，Angular 也确实允许我们如此使用，只不过除了在控制器中将 `$scope` 绑定数据的代码替换为 `this` 绑定，还需要在视图中使用 `ng-controller="appCtrl as app"`，而后使用过程中使用 `app.data` 引用数据。

> TODO: 控制器主体的参数如 $scope 是如何解析的？

### Service & Factory & Provider

在 Angular 中，这三个概念比较相似，其创建方式如下形式：

```
function fn() {}
// 向模块注册服务
module['service' | 'factory' | 'provider']('name', fn)
```

主要区别在于：

- 把 Service 作为参数注入到模块控制器中时，我们得到的实际上是传入函数的实例，即 `new fn()`
- Factory：参数得到的是 `fn()` 的结果
- TODO

下面是三者的实现，可以看到本质上调用的都是 provide：

```
provider.service = function(name, Class) {
  provider.provide(name, function() {
    this.$get = function($injector) {
      return $injector.instantiate(Class);
    };
  });
}

provider.factory = function(name, factory) {
  provider.provide(name, function() {
    this.$get = function($injector) {
      return $injector.invoke(factory);
    };
  });
}

provider.value = function(name, value) {
  provider.factory(name, function() {
    return value;
  });
};
```

> <http://iffycan.blogspot.co.uk/2013/05/angular-service-or-factory.html> 简单地分析了源码

### 命令（directive）

在视图层中使用 `{{}}` 直接引用控制器下的数据，而在花括号内，我们还可以使用表达式进行处理，如

```
<div ng-controller="demoCtrl">{{ items.length > 0 'True' : 'False' }}</div>
```

在视图层中，大量的交互及数据处理都是通过 Angular 内置的 `ng-*` 命令来实现。比如，在视图节点上添加 `ng-click` 属性会自动给它加上一个点击事件。除去前面提到的命令，一些常用的 `ng-*` 命令主要包括：

- `ng-repeat`：`<li ng-repeat="item in main.items" ng-click="process(item, $index)"></li>`
- `ng-model`：常用于 input 等表单元素中，将表单中的值与数据自动绑定起来
- `ng-show | ng-hide`： 
- `ng-if`：不同于 `ng-show`，`ng-if` 会在为 false 的情况下销毁对应的 DOM，注意，这在有些情况下是可以提升性能的（考虑把对应的 DOM 移除以后，对应的动态绑定数量就减少了）
- `ng-class`：可用于动态设置标签的 class
- `ng-bind`：`<p>{{main.value}}</p>` 等价于 `<p ng-bind="main.value"></p>`
- `ng-view`：用于 SPA 过程中加载相应 HTML 内容的容器

当然，Angular 也提供了自定义命令的方式。自定义命令包括多类：自定义的组件、

自定义的 DOM 元素（类似于组件化的概念），使用方式就非常简单了：

> TODO：自定义 directive


### 过滤器（Filter）

filter 允许我们对数据做一个处理再映射到页面上，HTML 的使用只需要

```html
<li ng-repeat="user in users | limitTo: 10 | orderBy: 'name' "></li>
```

一个自定义的 Filter 如下：

```javascript
// <div>{{item | myFilter}}</div>
angular.module('app').filter('myFilter', () => {
	return item => item.slice(0, 4)
})
```

> 注：上面这个是 module 级别的 filter，对于 controller 级别的 filter，直接在 controller `$scope` 上定义一个普通的方法，只不过使用的时候稍有不同：
> `<div>{{item | filter:myFilter}}</div>`


### 其它

`$http` 服务用于异步的网络请求，直接使用 `$http.get | $http.post` 即可发起请求，Angular 使用的是基于 `$q` 服务的 Promise 的返回方式。

在模块依赖中注入 `ngRoute` 模块后，我们就可以很方便地使用路由跳转来作单页面应用了，使用 `$routeProvider` 配置路由规则：

```javascript
angular.module('app', ['ngRoute'])
  .config(function ($routeProvider) {
    $routeProvider
    .when('/inbox', {})
    .when('/inbox/email/:id', {
      templateUrl: 'views/email.html'
    })
    .otherwise({
      redirectTo: '/inbox'
    });
  });
angular.module('app').controller('DemoCtroller', function ($routeParams) {
	var id = $routeParams.id;
})
// 同时在容器 view 中加入 ng-view 属性，Angular 会根据路由自动加载对应的模板并渲染页面。
```

### 参考资料

- [AngularJS Tutorial: A Comprehensive 10,000 Word Guide](https://www.airpair.com/angularjs)

***

如果想自己来考虑实现一个简单的 Angular，可以参考[这本书](http://teropa.info/build-your-own-angular)。另外，『The Complete Book on AngularJS』是一本比较系统介绍 Angular 的书。