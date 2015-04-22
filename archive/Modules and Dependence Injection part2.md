 
The Injector

让我们换个档，切换到Angular依赖注入的另一个重要的角色：Injector。

injector不是模块加载器的一部分，而是一个独立的服务，所以我们将代码和测试放到新的文件中。在测试中，我们总是假设一个新的模块加载器已经被设置好：

test/injector_spec.js

describe('injector', function() {
     beforeEach(function() {
         delete window.angular;
         setupModuleLoader(window);
     });
});
我们可以创建一个injector通过调用createInjector函数，这个函数用一个module的名称数组返回injector对象:

test/injector_spec.js

it('can be created', function() {
     var injector = createInjector([]);
     expect(injector).toBeDefined();
});
现在我们可以先把具体实现放一边，先简单的返回一个空的对象字面量：

src/injector.js

function createInjector(modulesToLoad) {
     return {};
}
Registering A Constant

我们将要实现的第一种angular应用的的部件是constants。用constant你可以注册一个简单值到angular module中，如number，object，或者function。

在我们注册constant到module并创建injector之后，我们可以使用injector的has方法去检验他是否的确知道这个constant：

test/injector_spec.js

it('has a constant that has been registered to a module', function() {
     var module = angular.module('myModule', []);
     module.constant('aConstant', 42);
     var injector = createInjector(['myModule']);
     expect(injector.has('aConstant')).toBe(true);
});
这里我们第一次看到完整的序列：定义一个module，然后定义一个injector。一个有趣的现象是，我们创建了一个injector，我们并没有直接传给他module对象的直接引用。而是我们给他module对象的名字，期望他自己去从angular.module中找。

作为一个全面的检测，我们得确认has方法返回false当东西没有被注册：

test/injector_spec.js

it('does not have a non-registered constant', function() {
     var module = angular.module('myModule', []);
     var injector = createInjector(['myModule']);
     expect(injector.has('aConstant')).toBe(false);
});
好，那constant是如何注册到一个module并可用的呢？首先，我们需要注册的方法存在于module对象中

src/loader.js

var createModule = function(name, requires, modules) {
     if (name === 'hasOwnProperty') {
         throw 'hasOwnProperty is not a valid module name';
     }
     var moduleInstance = {
         name: name,
         requires: requires,
         constant: function(key, value) {
         }
     };
     modules[name] = moduleInstance;
     return moduleInstance;
};
一个通用的规则关于module和injector是这样，module并不真正的包含任何应用的组件。他们仅仅包含创建应用组件的票据，injector才是他们具体化的地方。

那么，module应该保留的是一个任务集合，如“注册一个constant”，injector负责将这个任务执行，当injector加载module的时候。这个任务的集合叫做invoke queue。每个module拥有一个唤起队列，当injector加载module的时候，injector从唤起队列运行这些任务。

现在我们将定义一个唤起队列用数组的数组。每一个队列中的数组有两个项：应用组件的类型和注册该组件所用的参数。一个定义一个constant，在我们的单元测试中的的唤起队列，长这样：

[
     ['constant', ['aConstant', 42]]
]
唤起队列存储在module的一个属性叫_invokeQueue（下划线标明了它应该被认为是module中的私有变量）。从constant函数，我们现在将要放置一个项到队列中：

src/loader.js

var createModule = function(name, requires, modules) {
     if (name === 'hasOwnProperty') {
         throw 'hasOwnProperty is not a valid module name';
     }
     var moduleInstance = {
         name: name,
         requires: requires,
         constant: function(key, value) {
             moduleInstance._invokeQueue.push(['constant', [key, value]]);
         },
_         invokeQueue: []
     };
     modules[name] = moduleInstance;
     return moduleInstance;
};
当我们创建injector，我们应该遍历所有输入的module的名称，寻找关联的module对象，然后穷尽其唤起队列：

src/injector.js

function createInjector(modulesToLoad) {
     _.forEach(modulesToLoad, function(moduleName) {
         var module = angular.module(moduleName);
         _.forEach(module._invokeQueue, function(invokeArgs) {
         });
     });
     return {};
}
在injector中，我们将有这样的代码知道如何去处理每一个唤起队列持有的项。我们会将这些代码放到一个对象叫做$provide（原因我们之后会清楚）。当我们遍历唤起队列的项的时候，我们从$provide方法中查找一个方法，根据项数组的第一个参数，例如constant。然后我们调用这个方法，参数是项数组的第二个参数：

src/injector.js

function createInjector(modulesToLoad) {
     var $provide = {
         constant: function(key, value) {
         }
     };
     _.forEach(modulesToLoad, function(moduleName) {
         var module = angular.module(moduleName);
          _.forEach(module._invokeQueue, function(invokeArgs) {
             var method = invokeArgs[0];
             var args = invokeArgs[1];
             $provide[method].apply($provide, args);
         });
     });
     return {};
}
好，当你在module上调用一个方法如constant，这会触发createInjector中$provide对象同样参数同样名字的方法。只是并不立即发生，仅仅等到module被加载的时候才去执行。同时，唤起方法的信息同样会被保存在唤起队列中。

剩下的是注册constant的真正的逻辑。通常，所有的应用组件会被缓存到injector中。一个constant是简单值，我们可以很顺利的放到缓存中。我们可以实现injector的has方法去检验缓存中是否存在对应的键：

src/injector.js

function createInjector(modulesToLoad) {
     var cache = {};
     var $provide = {
         constant: function(key, value) {
             cache[key] = value;
         }
     };
     _.forEach(modulesToLoad, function(moduleName) {
         var module = angular.module(moduleName);
         _.forEach(module._invokeQueue, function(invokeArgs) {
             var method = invokeArgs[0];
             var args = invokeArgs[1];
             $provide[method].apply($provide, args);
         });
     });
     return {
         has: function(key) {
             return cache.hasOwnProperty(key);
         }
     };
}
我们又一次碰到我们需要保护hasOwnProperty属性的情况。不能注册一个constant叫做hasOwnProperty：

test/injector_spec.js

it('does not allow a constant called hasOwnProperty', function() {
     var module = angular.module('myModule', []);
     module.constant('hasOwnProperty', _.constant(false));
     expect(function() {
         createInjector(['myModule']);
     }).toThrow();
});
我们在$provide中的constant方法中检测这个键来做到不允许这样：

src/injector.js

constant: function(key, value) {
     if (key === 'hasOwnProperty') {
         throw 'hasOwnProperty is not a valid constant name!';
     }
     cache[key] = value;
}
此外，除了仅仅检测应用的组件是否存在，injector还提供方法去获取该部件。为了达成，我们引入一个方法叫做get：

test/injector_spec.js

it('can return a registered constant', function() {
     var module = angular.module('myModule', []);
     module.constant('aConstant', 42);
     var injector = createInjector(['myModule']);
     expect(injector.get('aConstant')).toBe(42);
});
这个方法简单的从缓存中查找键值：

src/injector.js

return {
     has: function(key) {
         return cache.hasOwnProperty(key);
     },
     get: function(key) {
         return cache[key];
     }
};
大部分angular的依赖注入功能是在**loader.js**中的module加载器和**injector.js**中的injector合作来完成的。我们会把这类测试放到**injector_spec.js**中，**loader_spec.js**则严格负责module加载器。
Requiring Other Modules

目前为止，我们创建了一个injector从一个单一的module。但创建injector加载多个module也是可能的。最直接的方式就是在module的名称数组中提供多于一个的名称，交给createInjector。提供的module中的应用部件都会被注册：

test/injector_spec.js

it('loads multiple modules', function() {
     var module1 = angular.module('myModule', []);
     var module2 = angular.module('myOtherModule', []);
     module1.constant('aConstant', 42);
     module2.constant('anotherConstant', 43);
     var injector = createInjector(['myModule', 'myOtherModule']);
     expect(injector.has('aConstant')).toBe(true);
     expect(injector.has('anotherConstant')).toBe(true);
});
这个其实我们已经覆盖了，因为我们在createInjector中遍历了modulesToLoad数组。

另外一种加载多个module的方法是利用module依赖别的module。当用angular.module注册一个module，有一个第二数组参数，我们一直留空的，但那可以维护必要的module的名称。当module被加载，必要的module也会被加载：

test/injector_spec.js

it('loads the required modules of a module', function() {
     var module1 = angular.module('myModule', []);
     var module2 = angular.module('myOtherModule', ['myModule']);
     module1.constant('aConstant', 42);
     module2.constant('anotherConstant', 43);
     var injector = createInjector(['myOtherModule']);
     expect(injector.has('aConstant')).toBe(true);
     expect(injector.has('anotherConstant')).toBe(true);
});
这还具有传递性，例如1require2，2require3，加载1时，123都被加载：

test/injector_spec.js

it('loads the transitively required modules of a module', function() {
     var module1 = angular.module('myModule', []);
     var module2 = angular.module('myOtherModule', ['myModule']);
     var module3 = angular.module('myThirdModule', ['myOtherModule']);
     module1.constant('aConstant', 42);
     module2.constant('anotherConstant', 43);
     module3.constant('aThirdConstant', 44);
     var injector = createInjector(['myThirdModule']);
     expect(injector.has('aConstant')).toBe(true);
     expect(injector.has('anotherConstant')).toBe(true);
     expect(injector.has('aThirdConstant')).toBe(true);
});
这工作原理其实挺简单的。当我们加载一个module，在遍历module的唤起队列之前我们遍历其依赖的module，递归地加载每一个。我们还需要给module加载函数一个名字这样我们可以递归的调用：

src/injector.js

_.forEach(modulesToLoad, function loadModule(moduleName) {
     var module = angular.module(moduleName);
     _.forEach(module.requires, loadModule);
     _.forEach(module._invokeQueue, function(invokeArgs) {
         var method = invokeArgs[0];
         var args = invokeArgs[1];
         $provide[method].apply($provide, args);
     });
});
当你有module依赖其他的module，就很容易产生循环依赖：

test/injector_spec.js

it('loads each module only once', function() {
     var module1 = angular.module('myModule', ['myOtherModule']);
     var module2 = angular.module('myOtherModule', ['myModule']);
     createInjector(['myModule']);
});
我们当前的实现爆栈了，当试图加载它的时候，因为他总是递归到下一个module而并没有检测它是否已经存在。

我们要对循环依赖要做的是确保每一个module只被加载一次。两个（非循环）路径指向同一个module，它不会加载两次，这也会生效。所以我们不必做附加的处理。

我们引入一个对象追踪已经被加载了的module。在我们加载一个module之前我们检测它是否已经加载：

src/injector.js

function createInjector(modulesToLoad) {
     var cache = {};
     var loadedModules = {};
     var $provide = {
         constant: function(key, value) {
             if (key === 'hasOwnProperty') {
                 throw 'hasOwnProperty is not a valid constant name!';
             }
             cache[key] = value;
         }
     };
     _.forEach(modulesToLoad, function loadModule(moduleName) {
         if (!loadedModules.hasOwnProperty(moduleName)) {
             loadedModules[moduleName] = true;
             var module = angular.module(moduleName);
             _.forEach(module.requires, loadModule);
                 _.forEach(module._invokeQueue, function(invokeArgs) {
                     var method = invokeArgs[0];
                     var args = invokeArgs[1];
                     $provide[method].apply($provide, args);
                 });
             }
         });
         return {
             has: function(key) {
                 return cache.hasOwnProperty(key);
             },
             get: function(key) {
                 return cache[key];
         }
     };
}
 