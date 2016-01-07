## JavaScript 测试与持续集成

据统计，在开发过程中，大部分的时间并非在设计系统和编码上，而是在最后的调试环节。而且在系统构建完成之后，不管是出于重构的目的，还是修复漏洞，一旦修改了系统里某一部分代码，我们就很难保证不会出现一些 regression BUG。『所谓的 regression BUG 指的是修改代码后引入的 BUG。』

《重构》一书说的很好：『有效的单元测试会大幅提升开发的效率』。为何？要知道，额外的单元测试编写本来就会浪费我们大量的时间。但是，当你听说过「磨刀不误砍柴工」的道理时，你就不会对上面的话表示怀疑了。毕竟代码的维护时间要远大于开发时间，一旦出现新功能，为保证老功能的可用性，又得重新人肉般地把所有条件分支都测一遍是得有多累人。有了测试，直接跑一遍测试啥结果都出来了。

有同学说，编写单元测试很难。这一点不可否认，但需要注意的是，我这里所说的难，并不是指代码本身是有多难测试，而是指覆盖率等可度量的条件很难保证完全。如果说你的代码本身很难写好测试，那就说明你的代码耦合度高，模块化不清楚，不可独立测试。也许，你得反思自己写的代码模块是否设计得合理。

下面介绍 JavaScript 中测试相关的内容，是自己日常使用过程中的一些笔记，所以可能会持续更新。

### Mocha

[Mocha](https://mochajs.org/)（抹茶）是一款优秀的测试框架，可完美的运行于浏览器和 Node 环境中。而且它可以自定义输出报告，与其它的断言库（如 `Should.js`、`Chai` 等）集成使用。

通过 `npm install -g mocha` 安装之后，其使用方法非常简单，在 test 文件夹下写好测试代码以后，直接运行 `mocha` 命令就会自动进行测试了。test 是其默认的查找测试代码目录，也可以通过指定文件夹名来自定义测试文件夹。

```bash
mocha "custom-test/**/*.js"
```

下面使用 `should.js` 断言库配合 Mocha 使用，为方便理解，我们将测试下面的代码：

```javascript
// src/util.js
function sync(value) {
	return '' + value;
}

function async(value, callback) {
	setTimeout(function () {
		callback(sync(value));
	}, 1000);
}

function promise(value) {
	return new Promise(function (resolve, reject) {
		setTimeout(function () {
			resolve(sync(value));
		}, 500);
	});
}

module.exports = {
	sync: sync,
	async: async,
	promise: promise
};
```

**同步代码的测试**：

```javascript
// snippet-1
it('#sync', function () {
	util.sync(1).should.equal('1');
	util.sync(1).should.not.equal(1);
});
```

**异步代码的测试**：向 it 的回调中传入 done 回调参数，在异步代码完成后，调用该回调函数 done，即可完成异步的测试
	
```javascript
// snippet-2
describe('#async', function () {
	it('should pass for case 1', function (done) {
		util.async(1, function (value) {
			done();
		});
	});
	it('should throw error for case 2', function (done) {
		util.async(1, function (value) {
			if (+value === 2) {
				throw new Error('error');
				return
			}
			done();
		});
	});
});
```
	
**Promise的测试**：mocha 对于 Promise 的支持非常地友好，它可以直接用于测试而无需加一个 done　参数。

```javascript
// snippet-3
it('#promise', function () {
	return util.promise(1).then(function (value) {
		value.should.equal('1');	
	})	
});
```

**钩子函数**：

```javascript
describe('util toolkits', function () {
	before(function () {
		// console.log('before function will be called only once');
	});
	after(function () {
		// console.log('after function will be called only once');
	});
	beforeEach(function () {
		// console.log('beforeEach function will be called each test');
	});
	afterEach(function () {
		// console.log('afterEach function will be called each test');
	});
	
	// snippet-1
	// snippet-2
	// snippet-3		
});
```
将会输出：

```bash
  util toolkits
before function will be called only once
beforeEach function will be called each test
    ✓ #sync
afterEach function will be called each test
beforeEach function will be called each test
    ✓ #promise (506ms)
afterEach function will be called each test
    #async
beforeEach function will be called each test
      ✓ should pass for case 1 (1001ms)
afterEach function will be called each test
beforeEach function will be called each test
      ✓ should throw error for case 2 (1002ms)
afterEach function will be called each test
after function will be called only once


  4 passing (3s)
```

显然，上面的钩子函数是一个同步的过程，不能完全满足于随处可见的异步情形，因此 Mocha 提供了一种异步的钩子函数，只需要在原来的 `before` 回调中提供一个 `done` 参数，在异步结束的地方调用 `done`，即可在异步执行完成以后再进行测试。

**注意点**：

1. Mocha 中大量使用了匿名函数，但不要使用箭头函数，这有可能会破坏 Mocha 已有的 this 上下文。
2. 完成了基本功能的了解以后，就是使用了。在使用过程中，可以参考官方文档 <https://mochajs.org/>。

*TODO：与 Mocha 类似的另一个著名测试框架 Jest，值得关注。*

### Karma

### Istanbul

包含覆盖率测试： `istanbul cover _mocha  -- --recursive`

### 持续集成

「[travis-ci](https://travis-ci.org/getting_started)」作为一个优秀的集成工具，可以很好的与 Github 配合使用。
