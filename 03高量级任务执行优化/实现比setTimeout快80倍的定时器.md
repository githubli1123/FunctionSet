# 实现比 setTimeout 快 80 倍的定时器



### 起因

很多人都知道，`setTimeout` 是有最小延迟时间的，根据 MDN 文档 setTimeout：实际延时比设定值更久的原因：最小延迟时间[1] 中所说：

> 在浏览器中，setTimeout()/setInterval() 的每调用一次定时器的最小间隔是 4ms，这通常是由于函数嵌套导致（嵌套层级达到一定深度）。

在 HTML Standard[2] 规范中也有提到更具体的：

> Timers can be nested; after five such nested timers, however, the interval is forced to be at least four milliseconds.
>
> 计时器可以嵌套; 然而，这样嵌套5个定时器之后，时间间隔将被迫至少为4毫秒。

简单来说，5 层以上的定时器嵌套会导致至少 4ms 的延迟。

用如下代码做个测试：

```javascript
let a = performance.now();
setTimeout(() => {
  let b = performance.now();
  console.log(b - a);
  setTimeout(() => {
    let c = performance.now();
    console.log(c - b);
    setTimeout(() => {
      let d = performance.now();
      console.log(d - c);
      setTimeout(() => {
        let e = performance.now();
        console.log(e - d);
        setTimeout(() => {
          let f = performance.now();
          console.log(f - e);
          setTimeout(() => {
            let g = performance.now();
            console.log(g - f);
          }, 0);
        }, 0);
      }, 0);
    }, 0);
  }, 0);
}, 0);
```

在浏览器中的打印结果大概是这样的，和规范一致，第五次执行的时候延迟来到了 4ms 以上。

![图片](https://img-blog.csdnimg.cn/img_convert/1289a7f710042941400f772a922051ec.png)

### 原因

为什么会产生以上结果，浏览器采取的是什么策略？我们到 chromium 源码中来查找

```cpp
static const int maxIntervalForUserGestureForwarding = 1000; // One second matches Gecko.



static const int maxTimerNestingLevel = 5;



static const double oneMillisecond = 0.001;



// Chromium uses a minimum timer interval of 4ms. We'd like to go



// lower; however, there are poorly coded websites out there which do



// create CPU-spinning loops.  Using 4ms prevents the CPU from



// spinning too busily and provides a balance between CPU spinning and



// the smallest possible interval timer.



static const double minimumInterval = 0.004;
```

 注释中解释了原因：

Chromium 使用的最小计时器间隔为4ms。我们想再低一点;然而，也有一些编码很差的网站确实会产生 CPU 旋转循环。使用4ms可以防止 CPU 过于繁忙地旋转，并在CPU旋转和尽可能小的间隔计时器之间提供一个平衡。

其他主流浏览器比如（Safari、opera、firefox、IE）都采用了 `4ms` 的设定。 具体嵌套层级的阈值，不同浏览器实现也不一致，HTML Standard 中规定是 `>5`，chrome 实现是 `>=5`。

### 探索

假设我们就需要一个「立刻执行」的定时器呢？有什么办法绕过这个 4ms 的延迟吗，上面那篇 MDN 文档的角落里有一些线索：

> 如果想在浏览器中实现 0ms 延时的定时器，你可以参考这里[3]所说的 `window.postMessage()`。

这篇文章里的作者给出了这样一段代码，用 `postMessage` 来实现真正 0 延迟的定时器：

```javascript
(function () {



  var timeouts = [];



  var messageName = 'zero-timeout-message';



 



  // 保持 setTimeout 的形态，只接受单个函数的参数，延迟始终为 0。



  function setZeroTimeout(fn) {



    timeouts.push(fn);



    window.postMessage(messageName, '*');



  }



 



  function handleMessage(event) {



    if (event.source == window && event.data == messageName) {



      event.stopPropagation();



      if (timeouts.length > 0) {



        var fn = timeouts.shift();



        fn();



      }



    }



  }



 



  window.addEventListener('message', handleMessage, true);



 



  // 把 API 添加到 window 对象上



  window.setZeroTimeout = setZeroTimeout;



})();
```

由于 `postMessage` 的回调函数的执行时机和 `setTimeout` 类似，都属于宏任务，所以可以简单利用 `postMessage` 和 `addEventListener('message')` 的消息通知组合，来实现模拟定时器的功能。

这样，执行时机类似，但是延迟更小的定时器就完成了。

再利用上面的嵌套定时器的例子来跑一下测试：

![图片](https://img-blog.csdnimg.cn/img_convert/0bad0f9acc60bcddac4700d6a67f6b47.png)

全部在 0.1 ~ 0.3 毫秒级别，而且不会随着嵌套层数的增多而增加延迟。

### 测试

从理论上来说，由于 `postMessage` 的实现没有被浏览器引擎限制速度，一定是比 setTimeout 要快的。但空口无凭，咱们用数据说话。

作者设计了一个实验方法，就是分别用 `postMessage` 版定时器和传统定时器做一个递归执行计数函数的操作，看看同样计数到 100 分别需要花多少时间。读者也可以在这里自己跑一下测试[4]。

实验代码：

```javascript
function runtest() {



  var output = document.getElementById('output');



  var outputText = document.createTextNode('');



  output.appendChild(outputText);



  function printOutput(line) {



    outputText.data += line + '\n';



  }



 



  var i = 0;



  var startTime = Date.now();



  // 通过递归 setZeroTimeout 达到 100 计数



  // 达到 100 后切换成 setTimeout 来实验



  function test1() {



    if (++i == 100) {



      var endTime = Date.now();



      printOutput(



        '100 iterations of setZeroTimeout took ' +



          (endTime - startTime) +



          ' milliseconds.'



      );



      i = 0;



      startTime = Date.now();



      setTimeout(test2, 0);



    } else {



      setZeroTimeout(test1);



    }



  }



 



  setZeroTimeout(test1);



 



  // 通过递归 setTimeout 达到 100 计数



  function test2() {



    if (++i == 100) {



      var endTime = Date.now();



      printOutput(



        '100 iterations of setTimeout(0) took ' +



          (endTime - startTime) +



          ' milliseconds.'



      );



    } else {



      setTimeout(test2, 0);



    }



  }



}
```

实验代码很简单，先通过 `setZeroTimeout` 也就是 `postMessage` 版本来递归计数到 100，然后切换成 setTimeout 计数到 100。

直接放结论，这个差距不固定，在我的 mac 上用无痕模式排除插件等因素的干扰后，以计数到 100 为例，大概有 80 ~ 100 倍的时间差距。在我硬件更好的台式机上，甚至能到 200 倍以上。

![图片](https://img-blog.csdnimg.cn/img_convert/340676976038cc97b063bb2a4b4f5bcb.png)

### 作用

也许有同学会问，有什么场景需要无延迟的定时器？其实在 React 的源码中，做时间切片的部分就用到了。

借用 React Scheduler 为什么使用 MessageChannel 实现[5] 这篇文章中的一段伪代码：

```javascript
const channel = new MessageChannel();



const port = channel.port2;



 



// 每次 port.postMessage() 调用就会添加一个宏任务



// 该宏任务为调用 scheduler.scheduleTask 方法



channel.port1.onmessage = scheduler.scheduleTask;



 



const scheduler = {



  scheduleTask() {



    // 挑选一个任务并执行



    const task = pickTask();



    const continuousTask = task();



 



    // 如果当前任务未完成，则在下个宏任务继续执行



    if (continuousTask) {



      port.postMessage(null);



    }



  },



};
```

React 把任务切分成很多片段，这样就可以通过把任务交给 `postMessage` 的回调函数，来让浏览器主线程拿回控制权，进行一些更优先的渲染任务（比如用户输入）。

为什么不用执行时机更靠前的微任务呢？参考我的这篇对 EventLoop 规范的解读 [深入解析 EventLoop 和浏览器渲染、帧动画、空闲回调的关系](https://mp.weixin.qq.com/s?__biz=MzI3NTM5NDgzOA==&mid=2247484039&idx=1&sn=e70e5b6473917dcf71bfd3f60ddb2a7d&chksm=eb043afedc73b3e8fb3ac90613d52d14cd165d358912e519e13f25bbd236c3591386fb2e349a&token=1983269989&lang=zh_CN&scene=21#wechat_redirect)，关键的原因在于微任务会在渲染之前执行，这样就算浏览器有紧急的渲染任务，也得等微任务执行完才能渲染。

### 总结

通过本文，你大概可以了解如下几个知识点：

1. `setTimeout` 的 4ms 延迟历史原因，具体表现。
2. 如何通过 `postMessage` 实现一个真正 0 延迟的定时器。
3. `postMessage` 定时器在 React 时间切片中的运用。
4. 为什么时间切片需要用宏任务，而不是微任务。

#### 参考资料

[1]

MDN 文档 setTimeout：实际延时比设定值更久的原因：最小延迟时间: *https://developer.mozilla.org/zh-CN/docs/Web/API/WindowOrWorkerGlobalScope/setTimeout#%E5%AE%9E%E9%99%85%E5%BB%B6%E6%97%B6%E6%AF%94%E8%AE%BE%E5%AE%9A%E5%80%BC%E6%9B%B4%E4%B9%85%E7%9A%84%E5%8E%9F%E5%9B%A0%EF%BC%9A%E6%9C%80%E5%B0%8F%E5%BB%B6%E8%BF%9F%E6%97%B6%E9%97%B4*

[2]

HTML Standard: *https://html.spec.whatwg.org/multipage/timers-and-user-prompts.html#timers*

[3]

这里: *https://dbaron.org/log/20100309-faster-timeouts*

[4]

这里自己跑一下测试: *https://dbaron.org/mozilla/zero-timeout*

[5]

React Scheduler 为什么使用 MessageChannel 实现: *https://juejin.cn/post/6953804914715803678*