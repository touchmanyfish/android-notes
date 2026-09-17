# Handler消息队列分析

## 队列结构

一个消息队列可以关联多个handler。msg在target所在线程执行。

### msg插入规则

1. 队列为空，msg成为head；否则
2. msg.when == 0，msg会被插到队列的最前面；否则
3. 将msg插到最后一个when <= msg.when的消息的后面

对于插入规则2，我猜是为了实现“whe == 0表示执行优先级最高”的语义。

### 例子

```text
// 插入msg1(100)
msg1(100)

// 插入msg2(0)
msg2(0) -> msg1(100)

// 插入msg3(0)
msg3(0) -> msg2(0) -> msg1(100)

// 插入msg4(200)
msg3(0) -> msg2(0) -> msg1(100) -> msg4(200)

// 插入msg5(100)
msg3(0) -> msg2(0) -> msg1(100) -> msg5(100) -> msg4(200)
```

## 消息执行
(先讨论没有idleHandlers的情况)

### 队列中消息分类
1. barrier:msg(target == null)
2. sync_msg:msg(没有有FLAG_ASYNCHRONOUS)
3. async_msg:msg(带有FLAG_ASYNCHRONOUS)

### 执行规则:head为非barrier

从队列中取msg时执行时，如果head为可执行的(sync/async.now >= when)，则将它从链上取下来执行它，next成为新head；如果head不可执行则队列进入等待状态.

### 执行规则:head为barrier

从队列中取msg时执行时，寻找barrier后第一个可执行(的async_msg.now >= when)的async_msg(无视当前head后所有的barrier和sync_msg)，如果找到则将其从链上取下执行它；如果找不到则队列进入等待状态。

### 举例

```text
now(100)

sync(200) -> async(300) -> barrier(400) -> sync(500) -> barrier(550) -> async(600)
```

执行流程：
1. now(150)时取msg，没有可用，队列进入等待
2. now(220)时取msg，执行sync(200)
3. now(310)时取msg，执行async(300)
4. now(510)时取msg，没有可用，队列进入等待
5. now(610)时取msg，执行async(600)

值得注意的是，barrier(400)在查找下一个可执行的async(600)时，跳过了barrier(550)。

## 等待与唤醒策略
(先讨论没有idleHandlers的情况)

## nativePollOnce(ptr,nextPollTimeoutMillis)

```text
// 最多等300ms，或者300ms到达之前被唤醒
nativePollOnce(ptr, 300);

// 不等
nativePollOnce(ptr, 0);

// 一直等，直到被唤醒
nativePollOnce(ptr, -1);
```

## nativeWake(mPtr)

将对队列从nativePollOnce()导致的等待状态中唤醒


### 等待策略

在取msg时，如果当前没有可执行的msg，队列会调用nativePollOnce()进入等待状态；等待状态结束(等待时间到/被唤醒)结束以后会再次尝试获取msg。

```text
// 没有可执行的msg的情况举例

// 当前不可执行，但未来可执行
// head 非barrier,now(100) < head.when
1. sync(200) ->...
2. async(200) ->..
// head为barrier,now(100)
barrier(200) -> .. async(500)...

// 啥也没有
1. null
2. barrier -> ...（没有任何async）
```

### 唤醒策略

1. 如果发现必须唤醒/可能需要唤醒，调用nativeWake()进行唤醒
2. 如果发现明确不用唤醒，则跳过唤醒操作

#### 唤醒分析1

```java
//enqueueMessage()中节选
if (p == null || when == 0 || when < p.when) {
    // New head, wake up the event queue if blocked.
    msg.next = p;
    mMessages = msg;
    needWake = mBlocked;
}
```

p(null) + mBlocked(true):表示当前列表在等待，列表为null，这时候我插入的msg.when可能在当前时间之前也可能之后，如果是前者，则需要唤醒当前队列。如果是后者则可以让队列继续等待。

对于这种可能需要唤醒的场景，enqueueMessage()方法选择不做这个判断，而是直接唤醒队列。

可能需要唤醒的场景还有：
1. when < p.when + mBlocked(true)
2. removeSyncBarrier()中的唤醒场景
3. quit()方法中的唤醒场景

when(0) + mBlocked(true)：enqueueMessage()选择直接唤醒，我猜测是为了满足语义“when(0)的msg应该立即执行”，我将这种场景理解为“必须唤醒”

#### 唤醒分析2

```java
//enqueueMessage()中节选
// A
needWake = mBlocked && p.target == null && msg.isAsynchronous();
Message prev;

for (;;) {
    prev = p;
    p = p.next;
    if (p == null || when < p.when) {
        break;
    }
    if (needWake && p.isAsynchronous()) {
        // B
        needWake = false;
    }
}

msg.next = p;

// invariant: p == prev.next
prev.next = msg;
```
A + B的场景为：
```text
// 当前队列处于等待状态
// now(200)
mMessages
|
V
barriers .... ->async1(300) ...
```

现在你要插一个async2(400)到队列里去。显然插入队列以后不用打破等待状态。于是在这种情况下enqueueMessage()选择不唤醒等待中的队列。我将这种情况理解为“明确判断出不用唤醒”

#### 唤醒机制进一步猜测

对于那种实际不用唤醒但是却唤醒了情况，next()后续执行会再次进入等待。 我有如下进一步猜测：

nativeWake()的调用点期望做更少的唤醒判断，漏判的情况让next()来恢复等待状态。这样nativeWake()的调用点处就可以有更少的判断逻辑

## idleHandler

```java
// next()
if (pendingIdleHandlerCount < 0
        && (mMessages == null || now < mMessages.when)) {
    // A
    pendingIdleHandlerCount = mIdleHandlers.size();
}
```

idleHandler部分的逻辑如下：
1. 获取msg时发现没有可执行的msg，队列需要进入等待状态
2. 第一次进入等待+mIdleHandlers不为空，执行，然后立即尝试再次获取msg
3. 不执行，进入等待状态；
4. 只要进入过A处，后续在获取到msg以前不会再有idleHandler执行了(无论此时是否有idleHandler)

步骤3中“立即再次获取msg”的原因如下:
1. idleHandler执行过程中可能有新的可执行的msg插入
2. idleHandler执行时间可能超过原本需要等待的时间

### 判定规则
当消息队列即将进入等待状态时说明在接下来一段时间没有消息执行，idleHandler可以在一定条件下利用这个空余时间执行。

在判定时当前有如下情况:
```text
1. head == null
2. now < head(非barrier).when
3. now < head(barrier).when

// async为barrier后第一个async
4. head(barrier).when <= now < async.when
```

判定通过的条件为：
```text
第一次进入等待状态+1/2/3
```

在情况4中，async.when - now的时间窗口可以用于执行idleHandler，为什么消息队列不用呢？

#### head(barrier).when <= now < async.when分析

假设我们使用这个窗口来执行idleHandler，可能出现如下情况：
```text
// 即将进入等待状态
now(200)
barrier(100) -> async(500)

//执行
idleHandler(1000)

//在200+1000以后执行
async(500)
```
这里的问题是，如果在这种情况下使用这个窗口，可能导致barrier逻辑启动时的async延后执行。

我的猜测如下：

```text
这么实现是因为barrier逻辑优先级高于idleHandler的执行。
```

