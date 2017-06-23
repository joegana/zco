[test_status_url]:https://travis-ci.org/yyrdl/zco.svg?branch=master
[coverage_status_url]:https://coveralls.io/repos/github/yyrdl/zco/badge.svg?branch=master
[coverage_page]:https://coveralls.io/github/yyrdl/zco?branch=master

# ZCO ![build status][test_status_url] [![Coverage Status][coverage_status_url]][coverage_page]

基于Generator的协程模块,无外部依赖，支持defer和协程超时终止,可以让您愉快地书写异步代码，就像写同步代码一样，并且不依赖Promise.

推荐支持解构语法node.js版本。

 

> __什么是协程?__  协程是计算机程序组件，他允许我们写同步风格的但却是异步执行的代码。 




#  特别功能

* __调用栈跟踪__

  zco会自动为异常添加调用堆栈，方便debug异常出现的路径，解决了node里面异步方法出现异常难以debug的问题。

* __深度支持回调__

  node.js里面大部分原生接口都是基于回调实现的，比如文件模块'fs'，zco可以轻松地和这些模块协作，不需要任何包装。

* __defer__  

defer` 定义了一个在协程退出时必定会执行的操作，无论协程是否报错。可以使用defer定义一些退出时的清理工作，可类比C++的析构函数或是golang的defer关键字。


* __协程超时挂起__

 zco 提供的`co.all`和`co.timeLimit`两个方法支持设置超时时间。超时时zco将终止执行，避免无意义的资源消耗.详细见`co.timeLimit`的示例代码。简单理解就是一旦操作超时，
 后续未执行的操作都不会得到执行。
 
# 安装

	npm install zco

# 例子们

### 简单用法

```javascript

const co = require("zco");
const request = require("request")

const fake_async_func = function (callback) { //支持伪异步
	callback(undefined, "hello world");
}

co(function  * (next) {

     //请求baidu主页
	let[err, response, page] = yield request("https://www.baidu.com", next);//next作为request模块的回调函数传入,
	//err,response,page是request模块传给回调函数的参数
     //do something ....
	
	let[err2, msg] = yield fake_async_func(next);
	console.log(err2); //null
	console.log(msg); //"hello world"
})()

```



`co`方法接收一个`GeneratorFunction`,返回一个zco的`future`,`future`是一个`Function`实例，`future`接收一个函数作为他的参数，这个函数称之为zco的返回处理句柄,即`handler`。


### 调用栈跟踪

zco 默认自动为异常添加调用堆栈,可以调用`zco.__TrackCallStack(false)`全局禁用栈跟踪。

>启用栈跟踪会让zco的执行速度降低一半，时间耗在了获取堆栈上面，但基本和[co](https://github.com/tj/co)这类方案持平。性能测试结果见末尾。

示例：

```javascript
const co = require("zco");

const async_func = function (json) {
	return co(function  * (co_next) {
		yield setTimeout(co_next, 1000); //等待1秒，模拟异步操作
		return JSON.parse(json);
	})
}

const callFunc1 = function (json) {
	return async_func(json);
}
 
callFunc1("{")((err) => {
	console.log(err.stack)
})

```

打出的stack 如下：

```
SyntaxError: Unexpected end of JSON input
    at JSON.parse (<anonymous>)
    at f:\social_insurance_test\co\co.js:6:19                       //对应JSON.parse
    at callFunc1 (f:\social_insurance_test\co\co.js:11:11)          //对应 async_func 调用的地方
    at Object.<anonymous> (f:\social_insurance_test\co\co.js:14:1)  //对应callFunc1调用的地方
```

建议亲自尝试一下。

### Defer

一个实用的使用`defer`的例子:定义一个并发锁，控制访问百度首页的并发量是5，为保证锁被释放，在defer里调用`mutex.unLock` 方法.

>锁被占用以后必须在某处被释放，否则会造成死锁。然而无法确定代码是否会异常，进而会导致释放锁的操作没被执行。zco 通过defer提供了一个无论如何都会执行的保证。

 
```javascript
const reuqest = require("request");

//定义并发锁工厂
const ConcurrentLockFactory = {
	"new" : function (max_concurrent) {
		return {
			"current_running" : 0,
			"unLock" : function () {
				this.current_running--;
				this._awake();
			},
			"lock" : function () {
				this.current_running++;
			},
			"busy" : function () {
				return this.current_running > max_concurrent - 1;
			},
			"waitFree" : function (callback) {
				this._reply_pool.push(callback);
				this._awake();
			},
			"_reply_pool" : [],
			"_awake" : function () {
				if (this.current_running < max_concurrent) {
					let func = this._reply_pool.shift();
					if ("function" == typeof func) {
						func();
					}
				}
			}
		}
	}
}

//生成一个并发锁，最大并发量是5

const mutex = ConcurrentLockFactory.new(5);

 
// 下面这个方法将确保最大并发量是5，即使在同一时刻调用这个方法10000次。

const requestBaidu = function () {

	return co(function  * (next, defer) {
	
		defer(function  * (inner_next,error) {//`inner_next`的功能和`next`一致，error是`co`捕获到的错误，比如下面代码抛出的错误
		
			mutex.unLock();//释放锁
			
		});
		
		//并发控制,忙则等待
		
		if (mutex.busy()) {
			yield mutex.waitFree(next);
		}
		
		mutex.lock();//持有锁
		
		let[err, res, body] = yield request.get('https://www.baidu.com', next);
		if (err) {
			throw err;
		}
		return body;//在这个return 执行之后，或者异常抛出之后，defer里的操作将被执行
	});
}

```
### zco.timeLimit


该方法为一个操作设置最大时间限制，超时未完成则挂起并抛出超时错误.如果被挂起的协程有使用`defer`，那么被挂起时会同时运行`defer`定义的操作。

与正常退出执行`defer`定义的操作不同，如果`defer`定义的操作出现异常，那么异常将被忽略，但若是正常退出，则异常会被传递给最终的`handler`，

如果没有提供`handler`，则异常会被抛出。


```javascript

var variable = 1;
co.timeLimit(1 * 10, co(function  * (next) {
	variable = 11;
	yield setTimeout(next, 2 * 10);//wait 20ms,等待20毫秒模拟耗时的操作，由于大于10ms，超时，将在这里被挂起
	variable = 111;
}))((err) => {
	console.log(err.message); //"timeout"
})

setTimeout(function () {
	console.log(variable);
	
	//打印出11 而不是111，是因为超时后该协程被挂起了，后面的语句将不会执行

}, 5 * 10);

//more example

var variable2 = 2;

const co_func = function () {
	return co(function  * (next) {
		variable2 = 222;
		yield setTimeout(next, 20);//be suspended here because of timeout
		variable2 = 2222;
	});
}

co.timeLimit(10, co(function  * (next) {
	variable2 = 22;
	yield co_func();
}))((err) => {
	console.log(err.message); //"timeout";
})

setTimeout(function () {
	console.log(variable2); //"222"
}, 40);

```

### zco.all



并发执行一个操作集,前n个参数是要执行的操作，最后一个参数可以是数字，代表允许的最大执行时间，超时则会返回超时错误

```javascript

const co_func = function (a, b, c) {
	return co(function  * (next) {
	    yield setTimeout(next,10);//wait 10ms
		return a+b+c;
	})
}

const generic_callback = function (callback) { //第一个参数必须是回调函数
	callback(100);
}

co(function  * (next) {
	let timeout = 10 * 1000; //timeout setting
	let[err, result] = yield co.all(co_func(1, 2, 3), co_func(4, 5, 6), generic_callback, timeout); //支持设置超时时间，超时时间作为最后一个参数

	console.log(err); //null
	console.log(result) //[6,15,[100]]
})()

```

### 嵌套使用



```javascript
const co = require("zco");

const co_func=function(i){
  return co(function*(){
     return 10*i;
  })
}


co(function  * () {
	let [err1, result1] = yield co_func(1);
	if (err1) {
		throw err1;
	}

	let [err2, result2] = yield co_func(2);
    if (err2) {
   		throw err2;
   	}

	return result1+result2;
})((err, result) => {
	if (err) {
		console.log(err);
	} else {
		console.log(result);
	}
})

//或者

co.brief(function*(){

   let result1 = yield co_func(1);

   let result2 = yield co_func(2);

   return result1+result2;
})((err,result)=>{
    if (err) {//在最后处理异常
   		console.log(err);
   	} else {
   		console.log(result);
   	}
});

```

zco 假定所有的操作都可能出现异常，第一个返回值永远是error,第二个值才是正常的返回值。在我自己的工作项目中希望在异常出现的地方处理异常，因为不同地方出现的
异常有不同的意义，有的可以重试，有的需要直接终止。

在上面的第二个简明的例子中，异常在最后被处理，而不是在异常出现的地方。这是考虑到有时候并不需要细粒度的去处理异常，可以处理的粗犷一点。



### 当遇见Promise时


尽管不推荐使用Promise,zco也提供一个API来支持yield Promise.

```javascript

const promise_api = function (a, b) {
	return Promise.resolve().then(function () {
		return a + b;
	});
}

co(function  * (next) {

	let[err, data] = yield co.wrapPromise(promise_api(1, 2));
	/**
	 *  Can't yield Promise directly ,that's unsafe.Becauce some callback-style API also return a Promise at the
	 * same time,such as `pg.client.query`.
	 * */
	console.log(err); //null
	console.log(data) : //3;
})()

```

# Performance Battle

    带co前缀的都属于协程模块，在禁用栈跟踪的情况下测试zco。

    results for 20000 parallel executions, 1 ms per I/O op ,2017-06-03
     
	name                                                      timecost(ms)      memory(mb)       score(time+memory)     
    callback.js                                               96                30.23828125      46.5068
    async-neo@1.8.2.js                                        146               48.59765625      30.2967
    promise_bluebird@2.11.0.js                                509               84.8828125       10.1153
    co_zco_yyrdl@1.3.2.js                                     579               88.9609375       9.1068
    co_when_generator_cujojs@3.7.8.js                         721               117.109375       7.1949
    async_caolan@1.5.2.js                                     712               122.5859375      7.1672
    co_tj_with_bluebird_promise@4.6.0.js                      895               124.79296875     6.0711
    co_when_generator_cujojs_with_bluebird@3.7.8.js           916               131.3515625      5.8794
    async_await_es7_with_native_promise.js                    964               166.82421875     5.2861
    promise_native.js                                         949               179.29296875     5.2457
    co_tj_with_native_promise@4.6.0.js                        1107              163.2421875      4.8229
    co_when_generator_cujojs_with_native_promise@3.7.8.js     1112              173.63671875     4.719
    async_await_es7_with_bluebird_promise.js                  1183              191.41796875     4.3899
    co_coroutine_bluebird@2.11.0.js                           3695              242.4296875      2

    
    开启栈跟踪的测试结果：


	name                                                      timecost(ms)      memory(mb)       score(time+memory)     
    callback.js                                               92                31.1015625       49.8332
    async-neo@1.8.2.js                                        166               47.7109375       28.3802
    promise_bluebird@2.11.0.js                                510               85.125           10.4324
    async_caolan@1.5.2.js                                     716               122.328125       7.3841
    co_when_generator_cujojs@3.7.8.js                         789               117.17578125     6.9716
    co_tj_with_bluebird_promise@4.6.0.js                      884               126.046875       6.2992
    co_when_generator_cujojs_with_bluebird@3.7.8.js           883               131.0234375      6.231
    co_zco_yyrdl@1.3.2.js                                     1181              94.42578125      5.8436
    promise_native.js                                         999               170.3125         5.2953
    async_await_es7_with_native_promise.js                    1022              161.47265625     5.2862
    co_tj_with_native_promise@4.6.0.js                        1089              162.99609375     5.0394
    async_await_es7_with_bluebird_promise.js                  1165              188.90625        4.6036
    co_when_generator_cujojs_with_native_promise@3.7.8.js     1231              173.71875        4.5379
    co_coroutine_bluebird@2.11.0.js                           3867              242.61328125     2

    Platform info:
    Windows_NT 10.0.14393 x64
    Node.JS 7.7.3
    V8 5.5.372.41
    Intel(R) Core(TM) i5-5200U CPU @ 2.20GHz × 4
 


# License

MIT
