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

