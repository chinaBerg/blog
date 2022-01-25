# 异步队列实现

vue-router在每次路由跳转时都会触发一些的钩子函数,完整的导航解析流程如官网所示：

- 导航被触发
- 在失活的组件里调用 beforeRouteLeave 守卫。
- 调用全局的 beforeEach 守卫。
- 在重用的组件里调用 beforeRouteUpdate 守卫 (2.2+)。
- 在路由配置里调用 beforeEnter。
- 解析异步路由组件。
- 在被激活的组件里调用 beforeRouteEnter。
- 调用全局的 beforeResolve 守卫 (2.5+)。
- 导航被确认。
- 调用全局的 afterEach 钩子。
- 触发 DOM 更新。
- 调用 beforeRouteEnter 守卫中传给 next 的回调函数，创建好的组件实例会作为回调函数的参数传入。

所以，vue-router每次路由跳转时就必定会依次触发这些钩子函数，且每个钩子都可以是异步的。因此，必须要有一个可以异步串行执行的队列函数。

### 异步串行队列实现

```js
/**
 * 迭代队列函数，支持异步
 * @param {*} queue 待遍历的列表数据
 * @param {*} iteratorFn 迭代处理函数
 * @param {*} callback 队列迭代结束回调
 */
function runQueue(queue, iteratorFn, callback) {
  const step = index => {
    if (index < queue.length) {
      if (queue[index]) {
        iteratorFn(queue[index], () => {
          step(index + 1);
        });
      }
    } else {
      typeof callback === 'function' && callback();
    }
  }

  step(0);
}
```

实现的思路就是迭代队列，对每一项执行`iteratorFn`函数。`iteratorFn`的第一项参数是当前队列选项，只有第二项参数被调用时则迭代队列下一项。注意，`iteratorFn`的第二个参数是暴露给使用者手动调用的。

下面看下调用示例：

```js
/**
 * 使用演示
 * 最终结果会每隔1s输出一项数据
 */
const queue = [1, 2, 3, 4];
const iteratorFn = (item, next) => {
  setTimeout(() => {
    console.log(item);
    next();
  }, 1000)
}
runQueue(queue, iteratorFn);
```