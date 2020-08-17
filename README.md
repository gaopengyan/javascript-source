# javascript-source
手撕js源码

### call的实现

```javascript
let obj = {
	name: 'gpy'
};

function fn(x, y) {
	console.log(this, x + y);
	return '@';
}
let res = fn.call(obj, 10, 20);
console.log(res);
```

call实现的基本思路：把函数作为要改变的THIS对象的一个成员，然后基于对象的成员访问执行函数即可
 * 如何让fn的this指向obj 
 * obj.fn=fn  =>obj.fn()

```js
Function.prototype.call=function call(context,...params){
	//现在的this指向fn
	// context -> 最后要改变的函数中的this指向  obj
	// params -> 最后要传递给函数的实参信息  [10,20]
	// this -> 要处理的函数  fn
	context=context==null? window:context
	
	// 必须要保证CONTEXT得是一个对象·
	let contextType=typeof context;
	if(!/^(object|function)$/i.test(contextType)){
		context=Object(context)
	}
	let result,key=Symbol('KEY')//创建唯一
	context[key]=this//obj.fn=fn
	result = context[key](...params) //obj.fn(...params)
	delete context[key] 
	return result
}
```

### bind的实现

```js
// call/apply：立即执行函数并且修改里面的THIS
// bind：利用柯理化函数的编程思想，预先把 "需要处理的函数/改变的THIS/传递的实参" 等信息存储在闭包中，
// 后期到达条件（事件触发/定时器等），先执行返回的匿名函数，在执行匿名函数的过程中，再去改变THIS等  =>THIS和参数的预处理
Function.prototype.bind = function bind(context, ...params) {
    // this -> 处理的函数  func
    // context -> 要改变的函数中的THIS指向  obj
    // params -> 最后给函数传递的实参 
    let _this = this;//func
    return function anonymous(...args) {
        // args -> 可能传递的事件对象等信息  [MouseEvent]
        // this -> 匿名函数中的THIS是由当初绑定的位置触发决定的（总之不是func要处理的函数）
        _this.call(context, ...params.concat(args));
    };
};


```

### 防抖

防抖：在用户频繁触发的时候，我们只识别一次（识别第一次/识别最后一次）

第一次点击，没有立即执行，等待500MS，看看这个时间内有没有触发第二次，
有触发第二次说明在频繁点击，不去执行我们的事情（继续看第三次和第二次间隔...）；如果没有触发第二次，则认为非频繁点击，此时去触发；
```js

/*
 * debounce：实现函数的防抖（目的是频繁触发中只执行一次）
 *  @params
 *     func:需要执行的函数
 *     wait:检测防抖的间隔频率
 *     immediate:是否是立即执行（如果为TRUE是控制第一次触发的时候就执行函数，默认FALSE是以最后一次触发为准）
 *  @return
 *     可被调用执行的函数
 */
function debounce(func, wait = 500, immediate = false) {
	let timer=null;
	return function anonymous(){
		let now = immediate && !timer;//以第一次执行为准
		clearTimeout(timer);//在wait时间内第二次处罚就清除
		timer=setTimeout(()=>{
			timer=null
			// 执行函数:注意保持THIS和参数的完整度 //默认在最后一次触发
			!immediate ? func.call(this,...params):null//在wait时间内没有出发第二次就执行
		},wait)
		//默认第一次执行
		now ? func.call(this,...params):null
	}
}

```

### 节流

```js
/*
 * throttle：实现函数的节流（目的是频繁触发中缩减频率）
 *   @params
 *      func:需要执行的函数
 *      wait:自己设定的间隔时间(频率)
 *   @return
 *      可被调用执行的函数
 */
function throttle(func, wait = 500) {
	let timer = null,
		previous = 0; //记录上一次操作时间
	return function anonymous(...params) {
		let now = new Date(), //当前操作的时间
			remaining = wait - (now - previous);
		if (remaining <= 0) {
			// 两次间隔时间超过频率：把方法执行即可
			clearTimeout(timer);
			timer = null;
			previous = now;
			func.call(this, ...params);
		} else if (!timer) {//当有定时器时，就不用在设置了
			// 两次间隔时间没有超过频率，说明还没有达到触发标准呢，设置定时器等待即可（还差多久等多久）
			timer = setTimeout(() => {
				clearTimeout(timer);
				timer = null;
				previous = new Date();
				func.call(this, ...params);
			}, remaining);//差多少时间等多少时间
		}
	};
}
```

### 柯里化思想

柯理化函数思想：利用闭包保存机制，把一些信息预先存储下来（预处理的思想）

```js
let res = fn(1,2)(3);
console.log(res); //=>6  1+2+3
```

```js
function fn(...outerArgs){
	return function (...innerArgs){
		let args=outerArgs.concat(innerArgs)
		return args.reduce((n,item)=>n+item)
	}
}
```

### 惰性思想

惰性思想：懒，执行过一遍的东西，如果第二遍执行还是一样的效果，则我们就不想让其重复执行第二遍了 

```js
function getCss(element, attr) {
	//处理兼容性
	if ('getComputedStyle' in window) {
		getCss = function (element, attr) {
			return window.getComputedStyle(element)[attr];
		};
	} else {
		getCss = function (element, attr) {
			return element.currentStyle[attr];
		};
	}
	// 为了第一次也能拿到值
	return getCss(element, attr);
}
getCss(document.body, 'margin');
getCss(document.body, 'padding');
getCss(document.body, 'width'); 
```

### compose 组合函数，把多层函数嵌套调用扁平化

```javascript
const fn1 = (x, y) => x + y + 10;
const fn2 = x => x - 10;
const fn3 = x => x * 10;
const fn4 = x => x / 10;

// let res = fn4(fn2(fn3(fn1(20))));
// console.log(res);

function compose(...funcs) {
	// FUNCS:存储按照顺序执行的函数(数组) =>[fn1, fn3, fn2, fn4]
	return function anonymous(...args) {
		// ARGS:存储第一个函数执行需要传递的实参信息(数组)  =>[20]
		if (funcs.length === 0) return args;
		if (funcs.length === 1) return funcs[0](...args);
		return funcs.reduce((N, func) => {
			// 第一次N的值:第一个函数执行的实参  func是第一个函数
			// 第二次N的值:上一次func执行的返回值，作为实参传递给下一个函数执行
			return Array.isArray(N) ? func(...N) : func(N);
		}, args);
	};
}
let res = compose(fn1, fn3, fn2, fn4)(20, 30);
```

### new及Object.create()

```js
Object.create = function create(prototype) {
	if (prototype === null || typeof prototype !== "object") {
		throw new TypeError(`Object prototype may only be an Object: ${prototype}`);
	}
	// 创建一个类,创建这个类的实例,实例.__proto__=类.prototype；而我们让类.prototype等于传递的prototype；
	function Temp() {}
	Temp.prototype = prototype;
	return new Temp;
};

//=============================================
function Dog(name) {
	this.name = name;
}
Dog.prototype.bark = function () {
	console.log('wangwang');
}
Dog.prototype.sayName = function () {
	console.log('my name is ' + this.name);
}
// Func要操作的这个类（最后要创建这个类的实例）
// ARGS存储未来传递给Func类的实参
function _new(Func, ...args) {
	// 1.创建一个FUNC的实例对象（实例.__proto__=>类.prototype）
	// 在IE浏览器中,禁止我们使用__proto__（也可以理解为IE并没有提供给我们__proto__这个属性，防止我们去改变原型指向）
	/* let obj = {};
	obj.__proto__ = Func.prototype; */
	let obj = Object.create(Func.prototype);

	// 2.把FUNC当做普通函数执行（让方法中的THIS指向创建的实例）
	let result = Func.call(obj, ...args);

	// 3.分析函数执行的返回值（没有返回值或者返回的是原始值类型则默认都返回创建的实例，否则以函数自身返回的为主）
	if (result !== null && /^(object|function)$/.test(typeof result)) {
		return result;
	}
	return obj;
} 
let sanmao = _new(Dog, '三毛');
sanmao.bark(); //=>"wangwang"
sanmao.sayName(); //=>"my name is 三毛"
console.log(sanmao instanceof Dog); //=>true
```

### 深浅克隆

#### 浅克隆

```js
// 实现对象的克隆有哪些办法（克隆：内存地址是不一样的）
// Object.assign：浅比较/浅克隆
// {...obj}：展开运算符，也只能展开第一级，也是浅克隆
```

```js
// 实现数组的克隆
// slice/concat... 这些都是浅克隆 
// forEach/map/reduce... 遍历也只是遍历第一级
```

#### 深克隆

##### 1.暴力法：把原始数据直接变为字符串，再把字符串变为对象（此时浏览器要重新开辟所有的内存空间），实现出深度克隆/深拷贝

出现的问题：正则会变为空对象/函数直接消失/日期直接变为字符串/Symbol直接消失/BigInt直接报错/undefined也会直接消失....

##### 2.

缺点：1.我们的方法会形成死递归，循环引用 2.考虑不全

```js
// 深克隆/深拷贝
function cloneDeep(obj) {
	const constructor = obj.constructor; //想克隆一个和Obj一样的数据类型 
	if (obj === null) return null;
	if (typeof obj !== "object") return obj;//如果不是对象直接返回  基本类型或函数...
	//正则和日期需要特殊处理
	if (/^(RegExp|Date)$/i.test(constructor.name)) return new constructor(obj);
	let clone = new constructor();
	for (let key in obj) {
		if (!obj.hasOwnProperty(key)) break;
		//是私有的进行克隆
		clone[key] = cloneDeep(obj[key]);
	}
	return clone;
}
```

```js

```

### 实现instanceOf

```js
//=>example：要检测的实例
//=>classFunc:要检测的类//example通过原型链查找正好知道classFunc的原型相等  则是他的实例
function instance_of(example, classFunc) {
	let classPrototype = classFunc.prototype,
		proto = Object.getPrototypeOf(example);//proto =example.__proto__(ie不兼容)
	while (true) {
		if (proto === null) {
			// 到了Object.prototype.__proto__
			return false;
		}
		if (proto === classPrototype) {
			// 在当前实例的原型链上找到了当前类
			return true;
		}
		proto = Object.getPrototypeOf(proto);//proto =proto.__proto__(ie不兼容)
	}
}
let res = instance_of([12, 23], Array);
```


### promise

```js
(function () {
    // constructor
    function MyPromise(executor) {
        // 参数合法校验
        if (typeof executor !== "function") {
            throw new TypeError('MyPromise resolver ' + executor + ' is not a function');
        }

        // 设置实例的私有属性
        var _this = this;
        this.PromiseStatus = 'pending';
        this.PromiseValue = undefined;
        this.resolveFunc = function () {};
        this.rejectFunc = function () {};

        // 修改实例的状态和value：只有当前状态为pending才能修改状态
        function change(status, value) {
            if (_this.PromiseStatus !== "pending") return;
            _this.PromiseStatus = status;
            _this.PromiseValue = value;
            // 通知基于.then注入的某个方法执行（异步的）
            var delayTimer = setTimeout(function () {
                clearTimeout(delayTimer);
                delayTimer = null;

                var status = _this.PromiseStatus,
                    value = _this.PromiseValue;
                status === "fulfilled" ?
                    _this.resolveFunc.call(_this, value) :
                    _this.rejectFunc.call(_this, value);
            }, 0);
        }

        // new MyPromise的时候会立即把executor函数执行
        // executor函数执行出现错误，也会把实例的状态改为失败，且value是失败的原因
        try {
            executor(function resolve(value) {
                change('fulfilled', value);
            }, function reject(reason) {
                change('rejected', reason);
            });
        } catch (err) {
            change('rejected', err.message);
        }
    }

    // MyPromise.prototype
    MyPromise.prototype.then = function (resolveFunc, rejectFunc) {
        // 参数不传默认值的处理：目的是实现状态的顺延
        if (typeof resolveFunc !== "function") {
            resolveFunc = function (value) {
                return MyPromise.resolve(value);
            };
        }
        if (typeof rejectFunc !== "function") {
            rejectFunc = function (reason) {
                return MyPromise.reject(reason);
            };
        }

        var _this = this;
        return new MyPromise(function (resolve, reject) {
            // 我们返回的新实例的成功和失败（执行resolve/reject）
            // 由resolveFunc/rejectFunc执行是否报错来决定（或者返回值是否为新的MyPromise实例来决定）
            _this.resolveFunc = function (value) {
                try {
                    var x = resolveFunc.call(_this, value);
                    x instanceof MyPromise ? x.then(resolve, reject) : resolve(x);
                } catch (err) {
                    reject(err.message);
                }
            };
            _this.rejectFunc = function (reason) {
                try {
                    var x = rejectFunc.call(_this, reason);
                    x instanceof MyPromise ? x.then(resolve, reject) : resolve(x);
                } catch (err) {
                    reject(err.message);
                }
            };
        });
    };
    MyPromise.prototype.catch = function (rejectFunc) {
        return this.then(null, rejectFunc);
    };

    // 把MyPromise当作对象
    MyPromise.resolve = function (value) {
        return new MyPromise(function (resolve) {
            resolve(value);
        });
    };
    MyPromise.reject = function (reason) {
        return new MyPromise(function (_, reject) {
            reject(reason);
        });
    };
    MyPromise.all = function (promiseArr) {
        return new MyPromise(function (resolve, reject) {
            var index = 0,
                values = [];
            for (var i = 0; i < promiseArr.length; i++) {
                // 利用闭包的方式保存循环的每一项索引
                (function (i) {
                    var item = promiseArr[i];
                    // 如果当前项不是Promise:直接算作当前项成功
                    !(item instanceof MyPromise) ? item = MyPromise.resolve(item): null;
                    item.then(function (value) {
                        index++;
                        values[i] = value;
                        if (index >= promiseArr.length) {
                            // 所有的实例都是成功的
                            resolve(values);
                        }
                    }).catch(function (reason) {
                        // 只要有一个失败，整体就是失败的
                        reject(reason);
                    });
                })(i);
            }
        });
    };

    window.MyPromise = MyPromise;
})();
```
