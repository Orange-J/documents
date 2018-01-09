# Promises/A+

**为实现者提供一个健全的、可互操作的 JavaScript promise 的开放标准。**

原文地址: https://promisesaplus.com/

一个 *promise* 代表了一个异步操作的最终结果。与一个 promise 交互的主要方式是通过它的 `then` 方法向 promise 注册回调函数(callbacks)，回调函数则用来接收 promise 的最终值(eventual value)或者promise失败(cannot be fulfilled)的原因。

本规范详细描述了 `then` 方法的行为，提供了一个可互操作(interoperable)的基础，所有遵循 **Promises/A+** 规范的promise实现都可以依赖这个基础。因此，可以认为本规范是非常稳定的。尽管 Promises/A+ 组织可能会偶尔对本规范进行小幅的且向后兼容的修订以处理一些新发现的极端情况，我们只会在谨慎考虑、讨论和测试后才会向规范中加入较大的或向后不兼容的改动。

从历史上来讲，**Promises/A+** 规范阐明了更早出现的 **Promises/A** 提议，并扩展了它以涵盖 promise 的实际行为，也去掉了一些不足之处或有问题的地方。

最终，核心 **Promises/A+** 规范并不涉及如何`创建、fulfill 或者 reject ` promises，而是选择专注于提供一个可互操作的 `then` 方法。在将来的一些伴随规范中可能会涉及如何`创建、fulfill 或者 reject ` promises。

## 1. 术语
----
1.1. "promise"指一个拥有 `then` 方法的对象或函数，且该 `then` 方法的行为符合本规范。

1.2. "thenable"指一个拥有 `then` 方法的对象或函数。

1.3. "value"指任意合法的JavaScript值(包括 `undefined`，一个thenable或一个promise)。

1.4. "exception"指用 `throw` 抛出的异常。

1.5. "reason"指表明promise为何被rejected的一个值。

## 2. 规范要求
----

### 2.1 Promise状态

一个promise必须处于以下三种状态之一：**pending(未决议状态)**，**fulfilled(成功状态)** 或 **rejected(失败状态)**。

2.1.1. 当处于 `pending` 状态时，一个promise：

&nbsp;&nbsp;&nbsp;&nbsp;2.1.1.1. 可以转变为 `fulfilled` 状态或 `rejected` 状态。

2.1.2. 当处于 `fulfilled` 状态时，一个promise：

&nbsp;&nbsp;&nbsp;&nbsp;2.1.2.1. 不可再转变为任何其他状态。

&nbsp;&nbsp;&nbsp;&nbsp;2.1.2.2. 必须有一个 `value`，且值不可变。

2.1.3. 当处于 `rejected` 状态时，一个promise：

&nbsp;&nbsp;&nbsp;&nbsp;2.1.3.1. 不可再转变为任何其他状态。

&nbsp;&nbsp;&nbsp;&nbsp;2.1.3.2. 必须有一个 `reason` ，且值不可变。

这里的“不可变”对于对象等值来说是指向的对象不可变，而这并不意味着此对象的属性不可变（译者注：例如一个promise的值为{a：1}，若将其属性a的值改为2，也是允许的）。

### 2.2 `then`方法

一个promise必须提供一个 `then` 方法用于读取此promise的当前或最终值(value或reason)。

一个promise的 `then` 方法接受两个参数：

```
promise.then(onFulfilled, onRejected)
```

2.2.1. `onFulFilled` 和 `onRejected` 都是可选参数:

&nbsp;&nbsp;&nbsp;&nbsp;2.2.1.1. 若 `onFulFilled` 不是函数，则必须忽略它。

&nbsp;&nbsp;&nbsp;&nbsp;2.2.1.2. 若 `onRejected` 不是函数，则必须忽略它。

2.2.2. 若 `onFulFilled` 是一个函数，则：

&nbsp;&nbsp;&nbsp;&nbsp;2.2.2.1. 当 `promise` 变为 `fulfilled` 之后此函数必须被调用，且 `promise` 的 `value` 应作为此函数的第一个参数。

&nbsp;&nbsp;&nbsp;&nbsp;2.2.2.2. 在 `promise` 变为 `fulfilled` 之前此函数必须不被调用。

&nbsp;&nbsp;&nbsp;&nbsp;2.2.2.3. 此函数被调用的次数不可超过一次。

2.2.3. 若 `onRejected` 是一个函数，则：

&nbsp;&nbsp;&nbsp;&nbsp;2.2.3.1. 当 `promise` 变为 `rejected` 之后此函数必须被调用，且 `promise` 的 `reason` 应作为此函数的第一个参数。

&nbsp;&nbsp;&nbsp;&nbsp;2.2.3.2. 在 `promise` 变为 `rejected` 之前此函数必须不被调用。

&nbsp;&nbsp;&nbsp;&nbsp;2.2.3.3. 此函数被调用的次数不可超过一次。

2.2.4. 只有当 **执行上下文栈** 中只包含平台代码(platform code)时 `onFulfilled` 或 `onRejected` 才可以被调用。[详情见3.1]

2.2.5. `onFulfilled` 和 `onRejected` 必须以函数方式被调用，且没有`this`值(译者注: 以函数方式调用是指`onFulfilled(v)`这种方式，不可用 `new onFulfilled()` 方式或 `onFulfilled.call()` / `onFulfilled.apply()` 方式)。[详情见3.2]

2.2.6. 在同一个promise上可以多次调用 `then` 方法。

&nbsp;&nbsp;&nbsp;&nbsp;2.2.6.1. 当 `promise` 变为或已处于 `fulfilled` 时，所有 `onFulfilled` 回调必须按照其通过 `then` 注册的顺序被执行。

&nbsp;&nbsp;&nbsp;&nbsp;2.2.6.2. 当 `promise` 变为或已处于 `rejected` 时，所有 `onRejected` 回调必须按照其通过 `then` 注册的顺序被执行。

2.2.7. `then` 方法必须返回一个promise。[详情见3.3]

```
promise2 = promise1.then(onFulfilled, onRejected);
```

&nbsp;&nbsp;&nbsp;&nbsp;2.2.7.1. 如果 `onFulfilled` 或 `onRejected` 返回了一个值 `x`, 则执行Promise Resolution Procedure(Promise决议程序) `[[Resolve]](promise2, x)`。

&nbsp;&nbsp;&nbsp;&nbsp;2.2.7.2. 如果 `onFulfilled` 或 `onRejected` 抛出了一个异常 `e`，则 `promise2` 必须被rejected，且以 `e` 作为reason。

&nbsp;&nbsp;&nbsp;&nbsp;2.2.7.3. 如果 `onRejected` 不是函数且 `promise1` 已处于 `rejected` 状态，则  `promise2` 必须以 `promise1` 的 `reason` 被 `rejected`。

### 2.3 Promise Resolution Procedure(Promise决议程序)

**Promise Resolution Procedure** 是以一个promise和一个value为输入的抽象操作，我们将其表示为`[[Resolve]](promise, x)`。如果 `x` 是一个 `thenable` 且行为有几分像promise时，**Promise Resolution Procedure** 会试图让 `promise` 变为与 `x` 相同的状态。否则，`promise` 会被决议程序以 `x` 为值 `fulfilled`。

这种处理 `thenable` 的方式可以使不同的promise实现之间可以互操作，只要这些实现对外暴露了遵循 `Promises/A+规范` 的 `then` 方法即可。这种方式也使得遵循 `Promises/A+规范` 的实现可以“同化”那些没有遵循 `Promises/A+规范` 但具备合理的 `then` 方法的promise实现。

执行 `[[Resolve]](promise, x)` 需运行以下步骤：

2.3.1. 如果 `promise` 和 `x` 指向同一个对象，则  ***reject*** `promise` 并且以 `TypeError` 作为 `reason`。

2.3.2. 如果 `x` 是一个promise，则采用 `x` 的状态 [详情见3.4]:

&nbsp;&nbsp;&nbsp;&nbsp;2.3.2.1. 如果 `x` 处于 `pending` 状态，则 `promise` 必须保持 `pending` 状态直到 `x` 被 `fulfilled` 或 `rejected`。

&nbsp;&nbsp;&nbsp;&nbsp;2.3.2.2. 当 `x` 变为或已处于 `fulfilled` 时，用相同的 `value` 来 ***fulfill*** `promise`。

&nbsp;&nbsp;&nbsp;&nbsp;2.3.2.3. 当 `x` 变为或已处于 `rejected` 时，用相同的 `reason` 来 ***reject*** `promise`。

2.3.3. 如果 `x` 是一个对象或函数，

&nbsp;&nbsp;&nbsp;&nbsp;2.3.3.1. 令 `then` 指向 `x.then`。[详情见3.5]

&nbsp;&nbsp;&nbsp;&nbsp;2.3.3.2. 如果访问 `x.then` 属性导致抛出异常 `e`，则以 `e` 为 `reason` 来 ***reject*** `promise`。

&nbsp;&nbsp;&nbsp;&nbsp;2.3.3.3. 如果 `then` 是一个函数，则调用它，并且以 `x` 作为 `this`，以 `resolvePromise` 作为第一个参数，以 `rejectPromise` 作为第二个参数，且：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2.3.3.3.1. 如果或当 `resolvePromise` 以一个值 `y` 被调用了，则执行 `[[Resolve]](promise, y)`。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2.3.3.3.2. 如果或当 `rejectPromise` 以一个值 `r` 被调用了，则以 `r` 来 ***reject*** `promise`。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2.3.3.3.3. 如果 `resolvePromise` 和 `rejectPromise` 都被调用了，或者某个已被被多次调用了，则首次发生的调用生效，其余所有调用都被忽略。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2.3.3.3.4. 如果调用 `then` 抛出异常 `e`：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2.3.3.3.4.1 如果 `resolvePromise` 或 `rejectPromise` 已经被调用过了，则忽略这个异常。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2.3.3.3.4.2 否则，以 `e` 来 ***reject*** `promise`。

&nbsp;&nbsp;&nbsp;&nbsp;2.3.3.4. 如果 `then` 不是函数，则以 `x` 来 ***fulfill*** `promise`。

2.3.4. 如果 `x` 既不是对象也不是函数，则以 `x` 来 ***fulfill*** `promise`。

如果一个promise被一个 `thenable` 决议，且此 `thenable` 在一个循环的thenable链中，`[[Resolve]](promise, thenable)` 的递归特性会导致 `[[Resolve]](promise, thenable)` 再次被调用，根据上面的算法，这会导致无限递归。因此我们鼓励(但不强制要求)promise实现时能检测无限递归，并在出现无限递归时以一个 `TypeError` 来 ***reject*** `promise`。

## 3. 说明
----

3.1. 这里的“平台代码(platform code)”意为JS引擎、JS执行环境和实现promise的代码。实际上，这条要求是为了确保在 `then` 被调用的那次事件循环之后，并且是在一个新的调用栈里异步地执行 `onFulfilled` 和 `onRejected`。这种方式可以通过 “宏任务(macro-task)” 原理实现，比如 `setTimeout` 或 `setImmediate`，或通过 “微任务(micro-task)” 原理实现，比如 `MutationObserver` 或 `process.nextTick`。既然promise的实现被认为是平台代码，那么它本身可能包含一个任务调度队列或者一个用于调用函数的“trampoline”。

3.2. 这里指的是在 **严格模式(strict mode)** 下，`this` 在 `onFulfilled` 和 `onRejected` 中值为 `undefined`；在 **非严格模式** 下，`this` 指向全局对象(global object)。

3.3. 即使一个promise实现满足本规范所有要求，这个实现也可能允许出现 `promise2 === promise1`。每个promise实现应当在文档中说明该实现中是否会出现`promise2 === promise1` 的情况以及在什么情况下会出现。

3.4. 一般来说，只有当 `x` 来自当前的实现时，才能知道 `x` 是一个真正的promise。对于一个已知其符合当前实现的promise，这个条款允许使用特定于实现的方式来 **采用(adopt)** 该promise的状态。

3.5. 这种首先保存 `x.then` 的引用，然后测试此引用，再调用此引用的处理，避免了对 `x.then` 属性的多次访问。这种预防措施在面对一些特殊访问器时十分重要，因为有的访问器每次访问都会导致其值的改变。
