# ZSIM

## 核模型

三种支持的核结构：

- 简单IPC1核
- 时序核
- 乱序核（OOO）

用4种分析方法来模拟程序：

- 基础块（BBL）（mov, add, ja）
- load
- store
- branch分支

没有实现的：

- 错误路径预测
  - 对于Pin来说难以实现
  - 对Westmere来说可以略过
- 细粒度消息传递
  - 需要大量修改
- TLB和SMT
  - 暂不支持

用指令驱动的方式仿真核行为，用事件驱动的方式仿真非核行为。

**扩展核模型的几个步骤：**

1. 修改4种分析方法
2. 用自己实现的硬件结构替换掉原来的
3. 修改ooo中的参数

#### 编程实例

1. 实现乱序核的分支预测

   > 1. 实现一个新的分支预测类和预测方法
   >
   >    ```c++
   >    class GShareBranchPredictor {
   >    	private:
   >        	bool lastSeen;
   >        public:
   >    		// Predicts and updates; returns false if mispredicted            
   >        	inline bool predict(Address branchPc, bool taken) {
   >                bool prediction = (taken == lastSeen);
   >                lastSeen = taken;
   >                return prediction; // always predict taken
   >            }
   >    }
   >    ```
   >
   > 2. 替换掉ooo_core.h中原本的分支预测器
   >
   >    ```C++
   >    //BranchPredictorPAg<11, 18, 14> branchPred;
   >    GSharePredictor branchPred;
   >    ```

2. 改变乱序核的类型

   > 如何模拟其他体系结构的乱序核？
   >
   > 1. 获取重要的乱序核参数
   > 2. 改写ooo_core.h/cpp中的核参数
   > 3. 验证

## 内存系统

![image-20210917092737150](D:\notes\assets\ZSIM\image-20210917092737150.png)

### 内存系统设计

访存请求的过程：

1. core发出访存请求MemReq
2. 经过滤器判断为load或store指令后，执行access()函数
3. 一路访问到公共cache，此时向另外一个核发送InvReq, 阻塞其他核的访存
4. MemReq查找cache，访存和替换cache。

![image-20210917092710782](D:\notes\assets\ZSIM\image-20210917092710782.png)

#### 重要的类

1. MemReq

   > 表示一个访存请求
   >
   > 重要成员：
   >
   > - uint64_t lineAddr		//地址
   > - AccessTypetype type            //访问类型，包括GETS, GETX, PUTS, PUTX
   > - uint64_t cycle              //请求周期
   > - MESIState* state           //相干状态，包括M, E, S, I

2. MemObject

   > 处理内存请求的通用接口
   >
   > 重要方法：
   >
   > uint64_t access(MemReq& req)			//进行访存操作并返回完成时间

3. InvReq

   > 表示一个来自相干控制器的阻塞请求
   >
   > 重要成员：
   >
   > - uint64_t lineAddr		//地址
   > - InvType type               //阻塞类型，包括INV, INVX, FWD
   > - uint64_t cycle              //请求周期

4. BaseCache

   > 类cache对象的通用接口
   >
   > 重要方法：
   >
   > - void setParents(...)		//注册上层cache
   > - void setChildren(...)		//注册下层cache
   > - uint64_t invalidate(constInvReq& req)            //在当前位置和下层cache进行阻塞

5. Cache

   > 广义cache，包括tag数组，相干控制器，淘汰策略，以及上述组件的控制逻辑
   >
   > 重要成员：
   >
   > - uint32_t accLat		//访问延迟
   > - uint32_t invLat          //阻塞延迟

6. NUCACache

   > 非一致cache访问
   >
   > 重要成员：
   >
   > - BankDir* bankDir
   > - g_vector<BaseCache*> banks
   
7. StreamPrefetcher

   > 实现流预取器
   >
   > 重要成员：
   >
   > - Entry array[16]         流访问数组
   >
   > 预取器会想父cache发送自己的MemReq

8. 其他cache

   > 狭义Cache——自解释性，需要单独的目录以保证一致性
   >
   > 时序Cache——添加weave-phase模型
   >
   > 过滤Cache——核和内存之间的中间件
   >
   > - 通过加速load和store来加速仿真
   > - 重要方法： uint64_t load/store(Address vAddr, uint64_t curCycle)
   > - 添加虚拟索引，直接映射cache来在cache层之前过滤访问请求
   > - 过滤器保持一致性并检查时序错误

9. CacheArray

   > 用不同组织来实现一个tag数组
   >
   > 重要方法：
   >
   > - int32_t lookup(...)         //数组是否存有地址？如果有在哪一行？
   > - uint32_t preinsert(...)           //找到淘汰对象
   > - void postinsert(...)                 //完成淘汰

10. ReplPolicy

    > 淘汰策略
    >
    > 重要方法：
    >
    > - void update(uint32_t id, constMemReq* req)       //命中时调用
    > - void replaced(uint32_t id)           //淘汰时调用
    > - template<class C> uint32_t rankCands(constMemReq* req, C cands)           //寻找淘汰对象

11. PartReplPolicy、CC

简单的主存模型实现：

```c++
class SimpleMemory: public MemObject{uint64_t {
	uint_t latency;
	g_string name;
    
public:
	SimpleMemory(uint64_t _latency, g_string_name)
		: latency(_latency), name(_name){};
	constchar* getName() { return name.c_str(); }
	
	uint64_t access(MemReq& req) {
		switch (req.type) {
			case PUTS: case PUTX: // write
				*req.state= I;
			case GETS:
				*req.state= req.is(MemReq::NOEXCL)? S : E;
			case GETX:*req.state= M;
		}
		return req.cycle+ latency;
	}
};
```

主存模型的三种类型：

- 简单内存：固定延迟，无冲突
- MD1内存：用M/D/1队列的冲突模型
- DDR内存和DRAMSim内存：详细的DDR时序模型

并发控制：

- 阻止访问请求上传，允许阻塞请求下传
- cache有两个锁：访问锁和阻塞锁
- 阻塞锁优先，访问需要两个锁，阻塞只需要一个锁

```c++
uint64_t Cache::access(MemReq& req) 
{
	invLock.acquire(); accLock.acquire();
    // look up address etc
    invLock.release();
    parent->access(req);
    // check if we got an invalidation!
    accLock.release();
    return completionTime;
}

uint64_t Cache::invalidate(InvReq& req) 
{
    invLock.acquire();// do invalidation
    children.invalidate(req);
    invLock.release();
    return completionTime;
}
```

![image-20210917092637887](D:\notes\assets\ZSIM\image-20210917092637887.png)

实现LRU

```C++
    template<typenameC>
    uint32_t uint32_trank(constMemReq* req, C cands) 
    {
        uint32_t bestCand= -1;
        for (auto ci = cands.begin(); ci != cands.end(); ci++) 
        {
            if (array[*ci] == 0) { return *ci; }
            else if (timestamp –array[*ci] <timestamp –array[bestCand]) 
                bestCand= *ci;
        }
        return bestCand;
    }
	DECL_RANK_BINDINGS();
};

```



# 代码

### cache（ooo_core.h)

```
声明：
FilterCache* l1i;
FilterCache* l1d;
l1d->load()
l1d->store()
```

### init.h

> cache:
> 一个cache组分为多个cache，每个cache又分为多个bank，每个bank 为一个cache对象

#### 进程线程相关

```
pid						进程号(linux)
tid						线程号
cid						上下文id
procIdx					当前进程在zsim中的id
zinfo->numProcs			总进程数
zinfo->procArray[]		所有进程的进程树节点，数组
fPtrs[]					储存每个线程的指令处理函数，包括load，store，bbl，分支，预测读，预测写等函数
gid=(pid << 16) | tid

int到ThreadInfo*的映射，储存每个gid对应的ThreadInfo。
gidMap[gid] = ThreadInfo(gid, syscall(SYS_getpid), syscall(SYS_gettid), mask);


```

1. 用pin将线程开始时间与ThreadStart绑定
2. `ThreadStart---->SimThreadStart`
3. `SimThreadStart---->zinfo->sched->start(procIdx, tid, procTreeNode->getMask())`
4. `start---->gidMap[gid]=ThreadInfo(gid,syscall(SYS_getpid),syscall(SYS_gettid),mask);`

#### main流程

1. `PIN_InitSymbols()`+`PIN_Init()`初始化PIN
2. `SimInit()`初始化仿真系统
3. `getLibzsimAddrs()`设置地址
4. 初始化`fPtrs[i] = joinPtrs; cids[i] = UNINITIALIZED_CID;`joinPtrs为初始执行函数指针，在调用时转化为特定核的执行函数指针。
5. 把指令处理函数`Trace`绑定到程序指令上
6. 把线程处理函数`ThreadStart`和`ThreadFini`绑定到线程上
7. 把进程处理函数`FollowChild`，`BeforeFork`，`AfterForkInParent`，`AfterForkInChild`绑定到进程上
8. `PIN_StartProgram`启动程序

#### 指令处理函数Trace

1. 遍历执行程序的每个BBL，对每条指令执行`Instruction(ins)`函数
2. 在`Instruction(ins)`中，用PIN的API判断指令类型，读写分支等，然后调用对应的`fPtrs[tid]`中的函数。
3. 在这些函数中首先调用`join`函数，通过pid和tid获取gid，再获取gidMap中的线程信息，获取其中的cid。再通过cid获取zinfo->cores[]中该tid对应的核，将该核的指令处理函数指针存入fPtrs。最后调用该指令的指令处理函数，执行指令。

