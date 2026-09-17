# View渲染流程2-测量

```text
测量阶段
│
├── 1. measureHierarchy() × 0..N
│   │
│   └── performMeasure() × 0..N
│
└── 2. performMeasure() × 0..N
```

## 测量流程

确定顶层(wSpec,hSpec),然后然后使用它来启动View树的测量。在这个过程中可能会出现：
1. 需要重新计算(wSpec,hSpec)(0..N)
2. 需要重新测量View树(0..N)

另外在performLayout()中也可能出现measureHierarchy()的调用(layout过程中出现了requestLayout()调用)。

## measureHierarchy()

这个方法涉及2件事：
1. 确定顶层(wSpec,hSpec)(0..N)
2. 使用(wSpec,hSpec)测量View树(0..N)

## performMeasure()

这个方法调用顶层View的measure(wSpec,hSpec)来开始整个View树的本轮测量。

## measure()

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec)
```

这个方法做了如下事情：
* 使用onMeasure()测量自己和自己的children(如果业务需要)
* 如果满足条件执行跳过测量的优化

### onMeasure()
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec)
```
#### 测量约束
当在onMeasure()方法中进行测量时面临如下几种测量约束：
* parent约束：onMeasure()的参数widthMeasureSpec,heightMeasureSpec
* 默认约束:View#mMinWidth/View#mMinHeight
* 业务约束

测量时需要尽量全满足这些约束，当不同类之间约束发生冲突时，有如下规则:
1. 必须满足 parent 约束
2. 尽量同时满足默认约束与业务约束
3. 默认约束与业务约束发生冲突时，由具体实现决定最终结果

**测量约束冲突的例子:**
```text
// 默认约束与业务约束冲突
parent约束:w(100,EXACTLY),h(100,AT_MOST
默认约束:mMinWidth(10),mMinHeight(50)
业务约束:w(30),h(40)

//result1:优先默认约束
w(100,50)

//result2:优先业务约束
w(100,40)

//result3:具体实现决定无视默认/业务约束选择只满足parent约束
w(100,100)或w(100,90)
```

#### 测量的遍历
onMeasure()方法中的可测量主体有:
1. View自己
2. View的children(如果有)

这里有一个问题：当面对[自己+children]时，如何选择测量顺序？这取决于具体的实现。

**一种可能的测量顺序举例：**

```text
// 假设View结构
              A 
             / \
            /   \
          B      C 
                / \
               /   \
              D     E

// 一种可能测量顺序
A ① measure() → onMeasure()
                      /   \
                     /     \
        B ② measure()     C ③ measure() → onMeasure()
             → onMeasure()                   /       \
                                            /         \
                              D ④ measure()       E ⑤ measure() → onMeasure() 
                                   → onMeasure()       
```

**RecyclerView中实现的测量顺序规则：**
* 如果parent的约束为[w(EXACTLY),h(EXACTLY)],RV会先确定自己的尺寸，然后再测量children
* 如果parent约束不是上述情况，RV会先测量childrent，然后根据children的测量结果来推算自己的尺寸。

### 跳过测量相关的优化

measure()中会假设"同一组parent约束(w_spec,h_spec)总是产生相同的测量结果"，基于这个假设，当measure()发现parent约束和上一一样时，会跳过测量直接使用上一次的结果。

但是onMeasure()方法可以被子类实现于是可能出现：
```text
// 同一组parent约束，但是测量结果不同
// 第一次onMeasure()
(w_spec,h_spec) -> 测量结果A

// 第二次onMeasure()
(w_spec,h_spec) -> 测量结果B
```

measure()使用forceLayout来解决这个问题：
1. 当业务端认为必须要进行测量(例如上面提到的情况)时给View打上forceLayout标记
2. 测量时就一定不会跳过onMeasure()的执行

```text
// 例如可以通过如下2个方法来打force Layout标记
View#requestLayout()
View#forceLayout()    
```

还有一种“parent约束和上次不同但是依然跳过测量”的优化情况：
* 本次parent约束[(w,EXACTLY),(h,EXACTLY)]和上次的parent约束不同
* 但是本次parent约束中的(w,h)部分和上次的测量结果一致

我猜测，View认为如果已有测量结果恰好与parent明确要求一致，同时没有谁明确要求必须进行测量，则可以跳过测量。

```text
parent:你的尺寸必须是(w,h)
child:我上一次测就是(w,h),有人反对我不测了吗？没人？我直接复用(w,h)！
```
