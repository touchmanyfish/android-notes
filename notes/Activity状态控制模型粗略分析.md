# Activity状态控制模型粗略分析

## 模型
```text
请求方进程
    |
    | IActivityTaskManager
    ↓
system_server进程
    |
    | 计算状态迁移方案
    | ClientTransaction
    |
    | IApplicationThread
    ↓
App进程
    |
    | 执行生命周期操作
    ↓
Activity状态变化
```

任何角色可以通过IActivityTaskManager向系统进程发送一个activity相关的控制请求，系统进程收到请求以后会进行处理，在这个处理的过程中，会一次或者多次的通过IApplicationThread以
ClientTransaction的形式向app进程发送指令,app执行这些指令以后就会影响到activity的状态(查询指令除外)。

一句话：请求跨进程发给系统进程，系统进程计算应该执行哪些操作，然后跨进程指挥app进程执行

## 场景分析:startActivity()

```kotlin
// 在BActivity中启动AActivity
startActivity(Intent(this, AActivity::class.java))
```

接下来通过上述具体场景来分析这个模型。

### 发送请求

```java
// Instrumentation.java
// startActivity() -> execStartActivity()
// 请求发起者进程
public ActivityResult execStartActivity(
        Context who,
        IBinder contextThread,
        IBinder token,
        Activity target,
        Intent intent,
        int requestCode,
        Bundle options) {

    IApplicationThread whoThread = (IApplicationThread) contextThread;
    // 省略代码..
    try {
        // 省略代码..
        // 注释A
        int result = ActivityTaskManager.getService().startActivity(
                whoThread,// 注释B
                who.getOpPackageName(),
                who.getAttributionTag(),
                intent,
                intent.resolveTypeIfNeeded(who.getContentResolver()),
                token,
                target != null ? target.mEmbeddedID : null,
                requestCode,
                0,
                null,
                options
        );
        // 省略代码..
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }

    return null;
}
```

在注释A处通过getService()拿到IActivityTaskManager,然后通过调用它的startActivity()(Binder调用)向系统进程发出启动activity的请求。

注释B处将IApplicationThread传递给了系统进程，方便系统进程通过它来传递app进程应该执行的操作。

### 处理请求发送操作

操作的计算逻辑这里不讨论，这里只观察系统进程在这个过程中发送了哪些操作指令(ClientTransaction)

```java
// private class ApplicationThread extends IApplicationThread.Stub
// scheduleTransaction(ClientTransaction transaction)

// app进程，但是位于binder线程
public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    ActivityThread.this.scheduleTransaction(transaction);
}
```

系统进程发送的指令会传递到scheduleTransaction()方法中。

```text
// 当前场景下，系统进程发出的所有指令

// ClientTransaction1
mActivityCallbacks
->[TopResumedActivityChangeItem]
mLifecycleStateRequest
->null

// ClientTransaction2
mActivityCallbacks
->[]
mLifecycleStateRequest
->PauseActivityItem

// ClientTransaction3
mActivityCallbacks
->[LaunchActivityItem]
mLifecycleStateRequest
->ResumeActivityItem

// ClientTransaction4
mActivityCallbacks
->[TopResumedActivityChangeItem]
mLifecycleStateRequest
->null

// ClientTransaction5
mActivityCallbacks
->[]
mLifecycleStateRequest
->StopActivityItem
```


### ClientTransaction
这个类包含系统进程期望app进程执行的指令。
```java
// ClientTransaction.java

private List<ClientTransactionItem> mActivityCallbacks;

private ActivityLifecycleItem mLifecycleStateRequest;

public void preExecute(android.app.ClientTransactionHandler clientTransactionHandler)
```

#### app进程执行操作

ClientTransaction的执行有如下3个步骤

1. 执行preExecute()
2. 执行mActivityCallbacks中的ClientTransactionItem
3. 执行mLifecycleStateRequest

2/3都是在执行ClientTransactionItem，具体执行item时：
1. item.execute()
2. item.postExecute()

目前发现有2种执行上述3个步骤的调用路径，这导致1的执行线程有两种情况：
1. app进程收到系统进程发过来的ClientTransaction时所在的binder线程
2. ui线程

不过无论哪种情况，2/3总是在ui线程中执行。

####  调用路径:ui线程 -> preExecute()
```java
public void executeTransaction(ClientTransaction transaction) {
    mIsExecutingLocalTransaction = true;
    try {
        // 调用preExecute()
        transaction.preExecute(this);
        // 在当前线程执行mActivityCallbacks和mLifecycleStateRequest
        getTransactionExecutor().execute(transaction);
    } finally {
        mIsExecutingLocalTransaction = false;
        transaction.recycle();
    }
}
```
搜索代码发现executeTransaction()总是在ui线程执行！

####  调用路径:Binder线程 -> preExecute()

```java
// app进程收到系统传递ClientTransaction时所在的Binder进程
public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    ActivityThread.this.scheduleTransaction(transaction);
}

// ActivityThread.this.scheduleTransaction(transaction);
void scheduleTransaction(ClientTransaction transaction) {
    transaction.preExecute(this);
    // 将ClientTransaction传递给主线程
    sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
}

// ActivityThread.H(Handler,ui线程)
case EXECUTE_TRANSACTION:
    // ui线程收到了ClientTransaction
    final ClientTransaction transaction = (ClientTransaction) msg.obj;
    // 在当前线程执行mActivityCallbacks和mLifecycleStateRequest
    mTransactionExecutor.execute(transaction);
    if (isSystem()) {
        // Client transactions inside system process are recycled on the client side
        // instead of ClientLifecycleManager to avoid being cleared before this
        // message is handled.
        transaction.recycle();
    }

   break;
```

#### 理解mLifecycleStateRequest/mActivityCallbacks

```text
mLifecycleStateRequest表示当前ClientTransaction执行完毕以后对应activity应该到达的一个阶段性状态；mActivityCallbacks表示
在activity到达这个阶段性状态前需要执行的操作。

ClientTransaction执行的过程中activity的逻辑状态也会跟着流转，当ClientTransaction执行完毕时，activity到达最终逻辑状态。

```text
//activity逻辑状态流转
x -> y -> z

// 目前没掌握ClientTransaction的生成逻辑，如下是观察到的2种类型
// item序列可能1:
stateRequest:y -> stateRequest:z

// item序列可能2:
stateRequest:z
```
#### activity逻辑状态处理

当activity进入某个逻辑状态以后，对应的ClientTransactionHandler#handleXActivity()方法会被回调

```text
// 调用路径

某些ClientTransactionItem#execute()
       |
       |
       V
handleXActivity()

// ResumeActivityitem.java
// 例如当activity进入Resume状态时
public abstract void handleResumeActivity(@NonNull ActivityClientRecord r,
            boolean finalStateRequest, boolean isForward, boolean shouldSendCompatFakeFocus,
            String reason);
```

#### cycleToPath()

```text
// activity逻辑状态流转
当前状态   stateRequest
|         |
V         V
x -> y -> z
     ^
     |
     ? 
```
在activity状态变化的过程中，一般情况下不是每一个状态都会被设置给stateRequest。在上述例子中，app进程如何得知中间状态y并处理它？


```java
// TransactionExecutor.java
private void cycleToPath(
        ActivityClientRecord r, 
        int finish, 
        boolean excludeLastState, 
        ClientTransaction transaction
)

// activity当前状态
final int start = r.getLifecycleState();
// 阶段性目标状态
int finish
```

cycleToPath()用于计算activity的当前状态和阶段性目标状态finish之间的所有中间状态，并让activity按顺序流转这些状态；excludeLastState则控制是否流转目标状态。

cycleToPath()在执行mActivityCallbacks中的ClientTransactionItem和mLifecycleStateRequest的过程中都有调用时机。

```text
// excludeLastState:false,只流转中间状态
// activity逻辑状态流转
当前状态   stateRequest
|         |
V         V
x -> y -> z
     ^
     |
   cycleToPath()计算出的中间状态  
     
        
// handle方法调用
cycleToPath() -> handleYActivity()
item -> handleZActivity()
```

#### 启动例子具体分析

前面都说清楚了，这里对着代码debug一下就行了。







