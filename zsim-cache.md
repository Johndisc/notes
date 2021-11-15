## cache接口

zsim中所有的内存对象，包括cache和内存，都要继承自`MemObject`。该类仅有一个接口：`access()`，接受一个`MemReq`对象作为参数，包含所有访存参数。返回值为操作完成时间。

Cache对象的继承关系：Cache -> BaseCache -> MemObject。Cache定义了另一个接口`invalidate`。

## MemReq对象

两种使用情况：

1. 外部需要访问测存储系统，在调用`access()`方法前需要创建`MemReq`对象。需要传入访问地址`lineAddr`，起始周期`cycle`，访问类型`type`(GETS或GETX)
2. 上层cache请求下层cache实现一个上层的请求。例如上层需要替换脏块到下层时，需要创建一个`PUTX`类型的请求。这种请求可以递归执行直到命中cache或内存。

## cache系统架构

一个cache对象可以分为多个bank分区。子cache的访问首先通过哈希函数被映射到其中一个bank上，然后被分派到父分区上。每个分区都可作为独立的cache对象，通过`access()`访问，子分区到父分区的延迟储存在网络延迟配置文件中。

cache类的成员：

| `Cache` Field Name | Description                                                  |
| :----------------: | ------------------------------------------------------------ |
|         cc         | Coherence controller; Implements the state machine and shared vector for every cached block |
|       array        | 缓存块的地址                                                 |
|         rp         | 淘汰策略                                                     |
|      numLines      | cache对象的总块数                                            |
|       accLat       | access的延迟                                                 |
|       invLat       | invalidate的延迟                                             |

### tag数组

tag数组为一维地址数组，必须继承自`CacheArray`类，提供3个方法：

- `lookup()`

  返回某个地址的块索引，或-1

- `preinsert()`

  在插入新地址时调用，搜索空位。没有则会替换一个，返回索引，覆盖的地址会通过最后一个参数`wbLineAddr`返回。

- `postinsert()`

  给定地址和位置索引，将地址储存到目标位置。

  ### cache构建
  
  构建cache时，每一级cache对应一个`typedef vector<vector<BaseCache*>> CacheGroup`对象，即一个`BaseCache`的二维数组，调用`BuildCacheGroup`进行创建。其中，每个cache有caches行，banks列个bank，每个bank调用`BuildCacheBank`进行创建。
  
  ```c++
  CacheGroup* cgp = new CacheGroup;
  CacheGroup& cg = *cgp;
  
  uint32_t bankSize = size/banks;
  
  cg.resize(caches);
  for (vector<BaseCache*>& bg : cg) bg.resize(banks);
  
  cg[i][j] = BuildCacheBank(config, prefix, bankName, bankSize, isTerminal, domain);
  ```
  
  每个bank又分为`numLines`行，每个bank构造一个CacheArray对象，一个CC对象，最后构造一个Cache对象。
  
  ```c++
  uint32_t numLines = bankSize/lineSize;
  uint32_t numHashes = 1;
  uint32_t numSets = numLines/ways;
  ReplPolicy* rp = new LRUReplPolicy<true>(numLines);
  CacheArray* array = new SetAssocArray(numLines, ways, rp, hf);
  
  Cache* cache;
  CC* cc;
  if (isTerminal)
  	cc = new MESITerminalCC(numLines, name);
  else
  	cc = new MESICC(numLines, nonInclusiveHack, name);
  rp->setCC(cc);
  
  cache = new Cache(numLines, cc, array, rp, accLat, invLat, name);
  ```
  
  ![image-20211105101730542](D:\notes\assets\zsim-cache\image-20211105101730542.png)
  
  **连接parent和children**
  
  以每个bank为单位设置其parent，而`BaseCache`中的`setParents`其实就是CC的`setParents`。
  
  ```
  BaseCache->setParents(childId++, mems, network);
  
  void Cache::setParents(uint32_t childId, const g_vector<MemObject*>& parents, Network* network) {
      cc->setParents(childId, parents, network);
  }
  ```
  
  除了l1i和lid之外，其他级的cache的CC都是`MESICC`类型。该类有一个`MESITopCC`和一个`MESIBottomCC`对象，分别对应该级cache的children和parent。`MESITopCC`有一个`children`数组，`MESIBottomCC`有一个`parent`数组。在设置parent和children时，就构建这两个对象并初始化，将parents或children数组拷贝到`MESITopCC`或`MESIBottomCC`的对应数组中。
  
  ```
  MESITopCC* tcc;
  MESIBottomCC* bcc;
  
  void setParents(uint32_t childId, const g_vector<MemObject*>& parents, Network* network) {
  	bcc = new MESIBottomCC(numLines, childId, nonInclusiveHack);
  	bcc->init(parents, network, name.c_str());
  }
  
  void setChildren(const g_vector<BaseCache*>& children, Network* network) {
  	tcc = new MESITopCC(numLines, nonInclusiveHack);
  	tcc->init(children, network, name.c_str());
  }
  ```
  
  

