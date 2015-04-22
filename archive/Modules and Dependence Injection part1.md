 
模块和依赖注入

依赖注入是Angular一个决定性的功能，也是他的卖点之一。对于Angular应用的开发者来说，依赖注入是把所有东西整合到一起的关键。所有其他的功能其实是(半)独立的工具，但却是DI把所有东西整合到一起形成一个一致的框架。

因为DI这么核心，所以知道其是如何运作的就非常的重要。不幸的是，他貌似是更难掌握的一部分，我们只要看看网上每天提的问题就知道是这样的了。这也许是该部分归咎于文档和代码实现中所使用的术语，但事实上他很难掌握是因为他解决了一个不平凡的问题：把应用的不同的部分都连接到一起。

DI同时也是Angular最具争议的一个功能。当人们批评Angular，DI通常是他们不喜欢Angular的理由。一些人批评Angular注入的实现，一些批评不该把在JS中搞DI。在上述情况中，这些批评都有有效的论据但往往却像是对Angular注入的真正概念的混淆。希望这一部分内容能够帮助你，不仅仅在应用的开发中，而且在讨论这些事情的时候，减轻人们展现出来的困惑。

我们将引入两个主要的概念 modules 和 injectors,并看看他们怎么允许注册应用的组件，然后注入到任何需要他们的地方。

在这两个概念当中， modules是最直接跟开发者接触的一个。modules是配置应用的信息集合。他是你注册你的service、controller、directive、filter和其他组件的地方。

但injector才是给应用带来生命的东西。在modules中注册组件时，并没有任何组件真正的创建直到你创建一个injector，然后实例化modules。

下面我们将看到如何创建modules，以及如何创建一个injector来加载创建的modules。

The angular Global

如果你曾使用过Angular，你可能会跟全局的angular对象打过交道。这是我们引入angular对象的地方。

我们现在就需要这个对象的理由是他是注册的Angular modules的信息的存储的地方。当我们开始创建modules和injectors，我们会用到那存储。

处理modules的框架的组件被叫做module loader, 实现是在loader.js文件中。我们将在这引入angular全局对象。但首先，也是我们通常所做的，让我们先为其新建一个测试文件：

test/loader_spec.js

describe('setupModuleLoader', function() {
     it('exposes angular on the window', function() {
         setupModuleLoader(window);
         expect(window.angular).toBeDefined();
     });
});
这个测试假设了一个全局的函数叫setupModuleLoader，用window对象可以调用。当你实现了之后，将会在window对象上拥有一个angular属性。

让我们创建loader.js并让他通过测试

src/loader.js

function setupModuleLoader(window) {
     var angular = window.angular = {};
}
我们将会从这里开始。

Intializing The Global Just Once

因为angular全局为注册了的modules提供了存储空间，本质上他是维护全局状态的持有者。这意味着我们需要为管理状态做一些处理。首先，我们要在每一个测试之前有一个干净的状态，所以我们需要清除掉所有的已存在的全局angular对象：

test/loader_spec.js

beforeEach(function() {
     delete window.angular;
});
并且，在setupModuleLoader中，我们需要小心的处理，不要覆盖掉已经存在的angular，甚至多次的调用了该函数。当你在window上两次调用setupModuleLoader，在每一个调用之后，angular全局对象应该指向同一个对象。

test/loader_spec.js

it('creates angular just once', function() {
     setupModuleLoader(window);
     var ng = window.angular;
     setupModuleLoader(window);
     expect(window.angular).toBe(ng);
});
这可以用一个简单的对存在的window.angular检验来修复

src/loader.js

function setupModuleLoader(window) {
     var angular = (window.angular = window.angular || {});
}
我们很快将会再次使用这个load once模式，所以让我们将其抽象成一个函数ensure，输入时一个对象，一个属性名称，一个工厂函数，输出是一个值。这个函数使用工厂函数来产出属性，当且仅当它不存在

src/loader.js

function setupModuleLoader(window) {
     var ensure = function(obj, name, factory) {
         return obj[name] || (obj[name] = factory());
     };
     var angular = ensure(window, 'angular', Object);
}
在这种情况下，我们会调用Object()将一个空对象赋值给window.angular，实际上，这跟调用new Object()是一样的。

The module Method

我们引入到angular的第一个方法是我们将会在之后用得很多的方法：module。让我们断言这个方法实际是存在于新创建的angular全局对象里：

test/loader_spec.js

it('exposes the angular module function', function() {
     setupModuleLoader(window);
     expect(window.angular.module).toBeDefined();
});
就像全局对象本身，module方法不应该被覆盖，当setupModuleLoader多次调用的时候：

test/loader_spec.js

it('exposes the angular module function just once', function() {
     setupModuleLoader(window);
     var module = window.angular.module;
     setupModuleLoader(window);
     expect(window.angular.module).toBe(module);
});
我们现在可以重用我们新的ensure函数去构建这一方法：

src/loader.js

function setupModuleLoader(window) {
     var ensure = function(obj, name, factory) {
         return obj[name] || (obj[name] = factory());
     };
     var angular = ensure(window, 'angular', Object);
     ensure(angular, 'module', function() {
         return function() {
         };
     });
}
Registering a Module

奠定了基础之后，我们现在着手注册modules。

所有接下来在loader_spec.js中定义的测试将要同module加载器协同工作，所以我们为他们来创建一个嵌套的describe块，把设置module加载器放到before块中。这样我们不必在每一个测试中重复：

test/loader_spec.js

describe('modules', function() {
     beforeEach(function() {
         setupModuleLoader(window);
     });
});
第一个我们将测试的行为是我们可以调用angular.module函数然后获取一个module对象。

angular.module方法的签名是使用一个module的名称（字符串）和一个module的依赖数组，可以是空的数组。这个方法构建一个module对象并将其返回。其中module对象将会包含有名称信息，在name属性中：

test/loader_spec.js

it('allows registering a module', function() {
     var myModule = window.angular.module('myModule', []);
     expect(myModule).toBeDefined();
     expect([myModule.name](http://myModule.name)).toEqual('myModule');
});
当你用同样的名称多次注册一个module的时候，新的module会替换旧的。这也说明当我们调用module两次，用同样的名称，我们将会分别得到不同的module对象：

*test/loader_spec.js

it('replaces a module when registered with same name again', function() {
     var myModule = window.angular.module('myModule', []);
     var myNewModule = window.angular.module('myModule', []);
     expect(myNewModule).not.toBe(myModule);
});
在我们的module方法中，我们将创建一个module的工作代理给了一个新的函数叫做createModule。在这个函数中，目前我们只是创建一个module对象并且返回它：

src/loader.js

function setupModuleLoader(window) {
     var ensure = function(obj, name, factory) {
         return obj[name] || (obj[name] = factory());
     };
     var angular = ensure(window, 'angular', Object);
     var createModule = function(name, requires) {
         var moduleInstance = {
             name: name
         };
         return moduleInstance;
     };
     ensure(angular, 'module', function() {
         return function(name, requires) {
             return createModule(name, requires);
         };
     });
}
除了名字以外，新的module应该有一个依赖模块数组的引用：

test/loader_spec.js

it('attaches the requires array to the registered module', function() {
     var myModule = window.angular.module('myModule', ['myOtherModule']);
     expect(myModule.requires).toEqual(['myOtherModule']);
});
我们仅需将给予的requires数组赋值到module对象上就可以满足需求了：

src/loader.js

var createModule = function(name, requires) {
     var moduleInstance = {
         name: name,
         requires: requires
     };
     return moduleInstance;
};
Getting A Registered Module

另一个angular.module提供的行为是获取一个module对象，之前注册过的对象。你可以通过省略第二个参数（requires数组）来实现。他将会给你的是跟创建时完全同样的module对象：

test/loader_spec.js

it('allows getting a module', function() {
     var myModule = window.angular.module('myModule', []);
     var gotModule = window.angular.module('myModule');
     expect(gotModule).toBeDefined();
     expect(gotModule).toBe(myModule);
});
我们引入另外一个私有的函数来获取module，叫做getModule。我们同样需要提防来存储已经注册了的module。我们将在ensure闭包调用中一个私有的对象来实现。我们会把createModule和getModule传进去：

src/loader.js

ensure(angular, 'module', function() {
     var modules = {};
     return function(name, requires) {
         if (requires) {
             return createModule(name, requires, modules);
         } else {
             return getModule(name, modules);
         }
     };
});
在createModule函数中，我们必须现在把新创建的module存到modules中：

src/loader.js

var createModule = function(name, requires, modules) {
     var moduleInstance = {
         name: name,
         requires: requires
     };
     modules[name] = moduleInstance;
     return moduleInstance;
};
在getModule中，我们可以只查下modules就可以了：

src/loader.js

var getModule = function(name, modules) {
     return modules[name];
};
我们基本上保存了所有注册过的module到modules变量中。这也就是为什么只能定义angular和angular.module一次。否则本地的变量会被抹掉。

现在，当你试图去获取一个并不存在的module时，你会获得一个undefined。但应该发生的是一个异常。Angular会make a big noise当你引用一个不存在的module时：

test/loader_spec.js

it('throws when trying to get a nonexistent module', function() {
     expect(function() {
         window.angular.module('myModule');
     }).toThrow();
});
在getModule函数中，我们应该在返回之前检测module是否存在：

src/loader.js

var getModule = function(name, modules) {
     if (modules.hasOwnProperty(name)) {
         return modules[name];
     } else {
         throw 'Module '+name+' is not available!';
     }
};
最后，因为我们现在用hasOwnProperty方法去检测一个module的存在与否，所以我们必须小心不要在module缓存中覆盖了这个方法：

test/loader_spec.js

it('does not allow a module to be called hasOwnProperty', function() {
     expect(function() {
         window.angular.module('hasOwnProperty', []);
     }).toThrow();
});
createModule方法需要这一检测：

src/loader.js

var createModule = function(name, requires, modules) {
     if (name === 'hasOwnProperty') {
         throw 'hasOwnProperty is not a valid module name';
     }
     var moduleInstance = {
         name: name,
         requires: requires
     };
     modules[name] = moduleInstance;
     return moduleInstance;
};
 