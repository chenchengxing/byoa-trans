 
Dependency Injection

我们现在有了有点用的一个应用部件的注册入口, 我们可以加载东西到里面去，也可以从其中查找东西。但injector真正的目的是做真正的依赖注入。那就是，调用函数、构造对象并且自动的查找他们需要的依赖。在本章的剩余部分，我们将集中关注injector的依赖注入功能。

基本的概念是：我们给injector一个函数，然后让它去调用这个函数。我们还希望它找到函数所需要的参数并且提供给函数。

好，那injector怎么知道函数需要什么参数呢？最简单的实现是显示的提供参数信息，通过使用一个叫$inject的绑在函数上的属性。这个属性维护了函数依赖的一组名字。injector会去查找这些依赖并用来调用函数：

test/injector.js

it('invokes an annotated function with dependency injection', function() {
     var module = angular.module('myModule', []);
     module.constant('a', 1);
     module.constant('b', 2);
     var injector = createInjector(['myModule']);
     var fn = function(one, two) { return one + two; };
     fn.$inject = ['a', 'b'];
     expect(injector.invoke(fn)).toBe(3);
});
这可以简单的通过从injector的缓存中查到$inject数组中的每一项来实现，injector的缓存维护了依赖名字到值的映射关系：

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
     function invoke(fn) {
         var args = _.map(fn.$inject, function(token) {
             return cache[token];
         });
         return fn.apply(null, args);
     }
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
         },
         invoke: invoke
     };
}
这是最基本的依赖注入标注的实现。

Rejecting Non-String DI Tokens

我们已经看到了$inject数组应该如何包含依赖的名字。如果有人在这个数组里面放了无效的值，如number，那么我们目前的实现就会映射到undefined。而我们却应该抛出一个异常，让用户知道他们做错了什么：

test/injector.js

it('does not accept non-strings as injection tokens', function() {
     var module = angular.module('myModule', []);
     module.constant('a', 1);
     var injector = createInjector(['myModule']);
     var fn = function(one, two) { return one + two; };
     fn.$inject = ['a', 2];
     expect(function() {
         injector.invoke(fn);
     }).toThrow();
});
这可以简单的在依赖映射函数中用类型检查来实现：

src/injector.js

function invoke(fn) {
     var args = _.map(fn.$inject, function(token) {
         if (_.isString(token)) {
             return cache[token];
         } else {
             throw 'Incorrect injection token! Expected a string, got '+token;
         }
     });
     return fn.apply(null, args);
}
Binding this in Injected Functions

有时候你想要注入的函数其实是绑在对象上的方法，在这些函数中，this的值是有重要意义的。当直接调用时，JS托管了this绑定，但当非直接的调用，通过injector.invoke就没有自动的绑定this。取而代之的是，我们给injector.invoke this值，作为可选的第二参数，会在调用函数的时候绑定：

test/injector.js

it('invokes a function with the given this context', function() {
     var module = angular.module('myModule', []);
     module.constant('a', 1);
     var injector = createInjector(['myModule']);
     var obj = {
         two: 2,
         fn: function(one) { return one + this.two; }
     };
     obj.fn.$inject = ['a'];
     expect(injector.invoke(obj.fn, obj)).toBe(3);
});
我们已经用了Function.apply来调用函数，我们将值传过去就可以了（之前我们是提供了null值）：

src/injector.js

function invoke(fn, self) {
     var args = _.map(fn.$inject, function(token) {
         if (_.isString(token)) {
             return cache[token];
         } else {
             throw 'Incorrect injection token! Expected a string, got '+token;
         }
     });
     return fn.apply(self, args);
}
Providing Locals to Injected Functions

最通常的情况下，你仅仅需要injector提供函数所有的参数，但也可能有你需要在调用时显式的提供一些参数的情况。这可能是因为你需要覆盖一些参数，也可能因为一些参数不应该注册到injector当中。

为了达成这个目的，injector.invoke接受一个可选的第三参数，是一个依赖名字到值的本地映射对象。如果提供了，依赖查询会主要从这个对象中完成，然后才是从injector本身。这根之前$scope.$eval的实现差不多：

test/injector.js

it('overrides dependencies with locals when invoking', function() {
     var module = angular.module('myModule', []);
     module.constant('a', 1);
     module.constant('b', 2);
     var injector = createInjector(['myModule']);
     var fn = function(one, two) { return one + two; };
     fn.$inject = ['a', 'b'];
     expect(injector.invoke(fn, undefined, {b: 3})).toBe(4);
});
在依赖映射函数，如果提供了local，我们将在本地先查找，退化到缓存中查找如果在locals中没有找到：

src/injector.js

function invoke(fn, self, locals) {
     var args = _.map(fn.$inject, function(token) {
         if (_.isString(token)) {
             return locals && locals.hasOwnProperty(token) ?
                 locals[token] :
                 cache[token];
         } else {
              throw 'Incorrect injection token! Expected a string, got '+token;
         }
     });
     return fn.apply(self, args);
}
Array-Style Dependency Annotation

你可以总是用$inject属性来标注一个注入函数，你也不想总是这样做因为这样非常繁琐。一个不是那么繁琐的提供依赖的名字的选择是提供injector.invke一个数组而不是一个函数。在数组中，你首先给出依赖的名称，最后一项是实际的要调用的函数：

['a', 'b', function(one, two) {
return one + two;
}]
因为我们现在正在介绍几种不同的标注函数的方法，需要一个可以选取任意标注类型的方法。这样的一个方法事实上是injector提供的。叫做annotate。

我们在一个新的嵌套了的describe块中指定这一方法的行为。首先，当函数有$inject属性，annotate返回属性的值：

test/injector_spec.js

describe('annotate', function() {
     it('returns the $inject annotation of a function when it has one', function() {
         var injector = createInjector([]);
         var fn = function() { };
         fn.$inject = ['a', 'b'];
         expect(injector.annotate(fn)).toEqual(['a', 'b']);
     });
});
我们在createinjector闭包中引入一个局部的函数，作为injector的一个属性暴露出去：

src/injector.js

function createInjector(modulesToLoad) {
     function annotate(fn) {
         return fn.$inject;
     }
     return {
         has: function(key) {
             return cache.hasOwnProperty(key);
         },
         get: function(key) {
             return cache[key];
         },
         annotate: annotate,
         invoke: invoke
     };
}
当给出一个数组，annotate从数组中抽取依赖的名字，根据我们数组注入的定义：

test/injector_spec.js

it('returns the array-style annotations of a function', function() {
     var injector = createInjector([]);
     var fn = ['a', 'b', function() { }];
     expect(injector.annotate(fn)).toEqual(['a', 'b']);
});
所以，当fn是一个数组annotate应该返回一个数组但不包括最后一个项：

src/injector.js

function annotate(fn) {
     if (_.isArray(fn)) {
         return fn.slice(0, fn.length - 1);
     } else {
         return fn.$inject;
     }
}
Dependency Annotation from Function Arguments

第三个也是最后一个，或许是定义一个函数依赖最有趣的方法是根本不去定义他们。当给injector传入一个没有$inject属性也没有数组包裹的函数，它会尝试从函数本身抽取出依赖的名字。

让我们先简单的处理零个参数的函数：

test/injector_spec.js

it('returns an empty array for a non-annotated 0-arg function', function() {
     var injector = createInjector([]);
     var fn = function() { };
     expect(injector.annotate(fn)).toEqual([]);
});
从annotate我们将返回一个空的数组如果函数没有被标注。这会使这个测试通过：

src/injector.js

function annotate(fn) {
     if (_.isArray(fn)) {
         return fn.slice(0, fn.length - 1);
     } else if (fn.$inject) {
         return fn.$inject;
     } else {
         return [];
     }
}
但如果函数有参数，我们需要找到抽取他们的方法来使下面的测试通过：

test/injector_spec.js

it('returns annotations parsed from function args when not annotated', function() {
     var injector = createInjector([]);
     var fn = function(a, b) { };
     expect(injector.annotate(fn)).toEqual(['a', 'b']);
});
这里的trick是用正则表达式去读函数的源码并抽取参数的声明。在js中，你可以通过调用toString方法获得一个函数的源码：

(function(a, b) { }).toString() // => "function (a, b) { }"
因为源码包含了函数的参数列表，我们可以用下列的正则抓取它，我们将它作为常量定义在injector.js的顶部：

var FN_ARGS = /^function\s*[^\(]*\(\s*([^\)]*)\)/m;
这个正则可以分解成下面几部分：

/^ 从头开始匹配
function 每一个函数以function关键字开始
\s* 接着是（可选的）空白
[^(]* 接着是（可选的）函数名- 不是‘(’的字符
( 接着是参数列表的左小括号
\s* 接着是（可选的）空白
( 接着是参数列表，我们通过捕获组来捕获
[ˆ)]* 读任意的不是‘)’的字符
) 结束捕获组
) 匹配右括号
/m 定义整个正则表达式可以匹配多行
我们在annotate中用这个正则匹配，我们会从作为匹配结果的第二项捕获组中获取参数的列表。然后用逗号分隔，我们可以得到参数名字的数组。当函数没有参数的时候（用Function.length检测），我们可以加上一个空数组的特殊情况：

src/injector.js

function annotate(fn) {
     if (_.isArray(fn)) {
         return fn.slice(0, fn.length - 1);
     } else if (fn.$inject) {
         return fn.$inject;
     } else if (!fn.length) {
         return [];
     } else {
         var argDeclaration = fn.toString().match(FN_ARGS);
         return argDeclaration[1].split(',');
     }
}
当你实现了这，你会注意到我们的测试仍然是失败了。这是因为第二个依赖名有额外的空格：‘ b’。我们的正则去除了参数列表的开头的空格，但并没有去除参数名之间的空格。为了去除，我们需要在返回前遍历参数的名字。

下面的正则会去除字符串任意的前置和后置的空白，然后在捕获组中捕获之间的非空白部分：

var FN_ARG = /^\s*(\S+)\s*$/;
通过映射参数的名字到匹配结果的第二项，我们得到了干净的参数名：

src/injector.js

function annotate(fn) {
     if (_.isArray(fn)) {
         return fn.slice(0, fn.length - 1);
     } else if (fn.$inject) {
         return fn.$inject;
     } else if (!fn.length) {
         return [];
     } else {
         var argDeclaration = fn.toString().match(FN_ARGS);
         return _.map(argDeclaration[1].split(','), function(argName) {
              return argName.match(FN_ARG)[1];
         });
     }
}
简单的‘on-the-fly’依赖标注可以工作了，但如果函数声明中参数间有注释的话会怎样呢：

test/injector_spec.js

it('strips comments from argument lists when parsing', function() {
     var injector = createInjector([]);
     var fn = function(a, /*b,*/ c) { };
     expect(injector.annotate(fn)).toEqual(['a', 'c']);
});
这里我们在不同的浏览器中遇到了不同的情况。一些浏览器通过Function.toString()返回的源码是已经剥除注释的，但一些却保留。在当前版本的PhantomJS WebKit剥除了注释，所以这个测试可能在你的环境中立即就通过了。Chrome不会去除这些注释，所以，如果你需要看到这些未通过的测试，链接Testem到Chrome直到本节结束（译者注：根据自己环境做选择）。
在抽取函数的参数之前，我们需要预处理函数的源码来去除包含的注释。这个正则是我们首先尝试做的：

var STRIP_COMMENTS = /\/\*.*\*\//;
正则匹配 /*字符，然后是任意的字符，然后是*/。用空的字符串来替换匹配的结果我们就可以去除注释了：

src/injector.js

function annotate(fn) {
     if (_.isArray(fn)) {
         return fn.slice(0, fn.length - 1);
     } else if (fn.$inject) {
         return fn.$inject;
     } else if (!fn.length) {
         return [];
     } else {
         var source = fn.toString().replace(STRIP_COMMENTS, '');
         var argDeclaration = source.match(FN_ARGS);
         return _.map(argDeclaration[1].split(','), function(argName) {
             return argName.match(FN_ARG)[1];
         });
     }
}
第一次的尝试并没有准确的切割，当参数列表有多个注释段：

test/injector_spec.js

it('strips several comments from argument lists when parsing', function() {
     var injector = createInjector([]);
     var fn = function(a, /*b,*/ c/*, d*/ ) { };
     expect(injector.annotate(fn)).toEqual(['a', 'c']);
});
正则匹配所有/与/之间的东西，所以任何在这之间的没有注释的地方丢失了。我们需要转换有效的匹配项为懒惰型，这样他会消费尽可能的少。我还需要在正则上加上g修饰，来匹配字符串中多个注释匹配项：

src/injector.js

var STRIP_COMMENTS = /\/\*.*?\*\//g;
然而，还有另一种形式的注释。参数的列表铺开到多行的时候，可能会包括// 形式的注释：

test/injector_spec.js

it('strips // comments from argument lists when parsing', function() {
     var injector = createInjector([]);
     var fn = function(a, //b, c) { };
     expect(injector.annotate(fn)).toEqual(['a', 'c']);
});
为了剥除这些注释，我们让STRIP_COMMENTS正则匹配两种不同的输入：一种是我们之前定义的，和一种以两个斜杆开始直到行尾。我们同样加上m修饰来匹配多行：

var STRIP_COMMENTS = /(\/\/.*$)|(\/\*.*?\*\/)/mg;
这样终于搞定了参数列表中的所有类型的注释！

最后一个我们解析参数名字时需要关注的功能是剥除前后的_。Angular允许你把_放置到一个参数名的前后，它会被忽略掉，这样这种获取注入参数到一个同名的局部变量变成可能：

var aVariable;
injector.invoke(function(_aVariable_) {
     aVariable = _aVariable_;
});
好，如果参数的两边都有，他们应该要从依赖的名字中被剥除。如果只有一边有，或者在中间，它应该被保留：

test/injector_spec.js

it('strips surrounding underscores from argument names when parsing', function() {
     var injector = createInjector([]);
     var fn = function(a, _b_, c_, _d, an_argument) { };
     expect(injector.annotate(fn)).toEqual(['a', 'b', 'c_', '_d', 'an_argument']);
});
下划线剥除可以在FN_ARG正则中完成，我们之前用它来剥除空白。它应该同样匹配一个可选的下划线，并且从后匹配相同的东西：

var FN_ARG = /^\s*(_?)(\S+?)\1\s*$/;
现在我们已经加了一个新的捕获组到正则中，真正的参数名会是第三个匹配结果项，而不是第二个：

src/injector.js

function annotate(fn) {
     if (_.isArray(fn)) {
         return fn.slice(0, fn.length - 1);
     } else if (fn.$inject) {
         return fn.$inject;
     } else if (!fn.length) {
         return [];
     } else {
         var source = fn.toString().replace(STRIP_COMMENTS, '');
         var argDeclaration = source.match(FN_ARGS);
         return _.map(argDeclaration[1].split(','), function(argName) {
             return argName.match(FN_ARG)[2];
         });
     }
}
Integrating Annotation with Invocation

我们现在已经可以用Angular支持的三种不同的方法抽取依赖的名字：$inject,数组包裹，以及函数源码抽取。我们还需要在injector.invoke中集成依赖名字的查找。你应该可以给它一个数组标注函数，然后希望他能做正确的事情：

test/injector_spec.js

it('invokes an array-annotated function with dependency injection', function() {
     var module = angular.module('myModule', []);
     module.constant('a', 1);
     module.constant('b', 2);
     var injector = createInjector(['myModule']);
     var fn = ['a', 'b', function(one, two) { return one + two; }];
     expect(injector.invoke(fn)).toBe(3);
});
以同样的方式，你应该可以给一个没有标注的函数并期望它从源码中解析依赖标注：

test/injector_spec.js

it('invokes a non-annotated function with dependency injection', function() {
     var module = angular.module('myModule', []);
     module.constant('a', 1);
     module.constant('b', 2);
     var injector = createInjector(['myModule']);
     var fn = function(a, b) { return a + b; };
     expect(injector.invoke(fn)).toBe(3);
});
在invoke方法中，我们将要做两件事：首先，我们需要用annotate函数查找依赖的名字而不是直接使用$inject。然后，我们需要检测函数是否包裹到一个数组里面，在尝试调用它之前需要打开它：

src/injector.js

function invoke(fn, self, locals) {
     var args = _.map(annotate(fn), function(token) {
         if (_.isString(token)) {
         return locals && locals.hasOwnProperty(token) ?
             locals[token] :
             cache[token];
         } else {
             throw 'Incorrect injection token! Expected a string, got '+token;
         }
     });
     if (_.isArray(fn)) {
         fn = _.last(fn);
     }
     return fn.apply(self, args);
}
现在我们满足了三种形式的函数调用！

Instantiating Objects with Dependency Injection

我们最后在injector中再加上一个能力然后我们结束这一章：可以注入普通函数，也可以注入构造函数。

当你有个构造函数并且想要用这个函数实例化一个对象，同时注入他的依赖，你可以用injector.instantiate.它可以处理一个显示用$inject标注的构造函数：

test/injector_spec.js

it('instantiates an annotated constructor function', function() {
     var module = angular.module('myModule', []);
     module.constant('a', 1);
     module.constant('b', 2);
     var injector = createInjector(['myModule']);
     function Type(one, two) {
         this.result = one + two;
     }
     Type.$inject = ['a', 'b'];
     var instance = injector.instantiate(Type);
     expect(instance.result).toBe(3);
});
你还可以用数组包裹形式：

test/injector_spec.js

it('instantiates an array-annotated constructor function', function() {
     var module = angular.module('myModule', []);
     module.constant('a', 1);
     module.constant('b', 2);
     var injector = createInjector(['myModule']);
     function Type(one, two) {
         this.result = one + two;
     }
     var instance = injector.instantiate(['a', 'b', Type]);
     expect(instance.result).toBe(3);
});
和普通函数一样，还可以从函数的源码中抽取依赖的名字：

test/injector_spec.js

it('instantiates a non-annotated constructor function', function() {
     var module = angular.module('myModule', []);
     module.constant('a', 1);
     module.constant('b', 2);
     var injector = createInjector(['myModule']);
     function Type(a, b) {
         this.result = a + b;
     }
     var instance = injector.instantiate(Type);
     expect(instance.result).toBe(3);
});
让我们在injector中引入新的instantiate方法。它指向一个本地的函数，我们很快会引入：

src/injector.js

return {
     has: function(key) {
         return cache.hasOwnProperty(key);
     },
     get: function(key) {
         return cache[key];
     },
     annotate: annotate,
         invoke: invoke,
         instantiate: instantiate
     };
一个最简单的实现是创建一个新的对象，调用其构造函数，同时新的对象绑定到this，然后返回一个新的对象：

src/injector.js

function instantiate(Type) {
     var instance = {};
     invoke(Type, instance);
     return instance;
}
这事实上通过了我们现有的测试。但还有一个重要的行为我们忘了：当你用new构造一个对象，你还可以构建对象的原型链。我们可以在injector.instantiate中遵循这一行为。

例如，如果我们实例化的构造函数有一个圆形，圆形上定义了一些额外的行为，这些行为应该在结果对象中通过继承的方式可用：

test/injector_spec.js

it('uses the prototype of the constructor when instantiating', function() {
     function BaseType() { }
     BaseType.prototype.getValue = _.constant(42);
     function Type() { this.v = this.getValue(); }
     Type.prototype = BaseType.prototype;
     var module = angular.module('myModule', []);
     var injector = createInjector(['myModule']);
     var instance = injector.instantiate(Type);
     expect(instance.v).toBe(42);
});
为了构建原型链，我们用ES5.1的Object.create函数而不是对象字面量构造一个对象。我们还需要记得打开构造函数，因为他可能用的是数组依赖标注：

src/injector.js

function instantiate(Type) {
     var UnwrappedType = _.isArray(Type) ? _.last(Type) : Type;
     var instance = Object.create(UnwrappedType.prototype);
     invoke(Type, instance);
     return instance;
}
Angular.js这里并没有用Object.create，因为它不支持旧浏览器但我们不关系这些。
最后，像injector.invoke支持提供一个locals对象一样，injector.instantiate也应该注意。可以给一个可选的第二参数：

test/injector_spec.js

it('supports locals when instantiating', function() {
     var module = angular.module('myModule', []);
     module.constant('a', 1);
     module.constant('b', 2);
     var injector = createInjector(['myModule']);
     function Type(a, b) {
         this.result = a + b;
     }
     var instance = injector.instantiate(Type, {b: 3});
     expect(instance.result).toBe(4);
});
我们仅需要将locals参数以第三个参数的形式传到invoke函数就可以了：

src/injector.js

function instantiate(Type, locals) {
     var UnwrappedType = _.isArray(Type) ? _.last(Type) : Type;
     var instance = Object.create(UnwrappedType.prototype);
     invoke(Type, instance, locals);
     return instance;
}
Summary

我们现在开启了Angular.js依赖注入框架全功能的旅程。目前我们已经有了一个完美的服务模块系统和一个可以注册constant的injector，还可以用它注入函数和构造函数。

本章中你学到了：

全局的angular和其moudle方法是怎么来的
module是如何这册的。
全局angular仅可以在一个window中注册一次，module可以被后面以同名注册的给覆盖。
如何查找之前被注册的module。
injector怎么来的。
injector是如何通过module的名字实例化的，通过angular全局来查找。
module中的应用部件是如何排队，并仅在injector加载module时才实例化的。
module可以依赖其他module，被依赖的module会被先加载。
injector仅加载每个module一次以防止循环依赖。
injector是如何被用来调用函数，和它是如何从$inject标注中查找参数的名字。
被注入的函数的this值是如何被injector.invoke绑定的。
函数的依赖可以被injector.invoke函数提供的本地对象覆盖。
函数包裹型注入如何工作。
函数依赖如何从函数的源码中查找。
如何在多种的依赖形式中抽取依赖。
在依赖注入中对象如何被实例化。
下一张中我们将关注Providers，angular依赖注入系统中关键的一块，很多高级的功能可以在其上构建。

 