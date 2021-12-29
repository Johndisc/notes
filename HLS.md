# 结构体

## graphAccelerator

```c++
CSR* csr;

subPartitionDescriptor subPartitions[MAX_PARTITIONS_NUM * SUB_PARTITION_NUM];

partitionDescriptor partitions[MAX_PARTITIONS_NUM];

gatherScatterDescriptor * gsKernel[SUB_PARTITION_NUM];

applyDescriptor * applyKernel;

cl_command_queue gsOps[SUB_PARTITION_NUM];		//clCreateCommandQueue()

cl_command_queue applyOps;						//clCreateCommandQueue()

cl_program program;

cl_platform_id platform;

cl_device_id device;

cl_context context;
```

## gatherScatterDescriptor

```
const char* name;
int partition_mem_attr;
int prop_id;
int output_id;
cl_kernel kernel;
he_mem_t  prop[2];
he_mem_t  tmpProp;
he_mem_t  bitMap;
```

```
gatherScatterDescriptor localGsKernel[] = {
        {.name = "readEdgesCU:{readEdgesCU_1}",},
        {.name = "readEdgesCU:{readEdgesCU_2}",},
        {.name = "readEdgesCU:{readEdgesCU_3}",},
        {.name = "readEdgesCU:{readEdgesCU_4}",},
        {.name = "readEdgesCU:{readEdgesCU_5}",},
        {.name = "readEdgesCU:{readEdgesCU_6}",},
        {.name = "readEdgesCU:{readEdgesCU_7}",},
        {.name = "readEdgesCU:{readEdgesCU_8}",},
};
acc->gsKernel[i] = getGatherScatter(i);
```

## applyDescriptor

```
const char* name;
cl_kernel kernel;
```

```
applyDescriptor localApplyKernel[] ={
	{.name = "vertexApply",//"vertexApply",},
};
acc->applyKernel = getApply();
```

## CSR

```
const int vertexNum;
const int edgeNum;
std::vector<int> rpao;  //offset
std::vector<int> ciao;  //neighbor
std::vector<int> rpai;	//pull offset
std::vector<int> ciai;	//pull neighbor
```

## he_mem_t

```C++
unsigned int        id;					//MEM_ID_VERTEX_PROP
const char          *name;				//"vertexProp"
unsigned int        attr;				//ATTR_HOST_ONLY
unsigned int        unit_size;			//sizeof(prop_t)
unsigned int        size_attr;			//SIZE_IN_VERTEX

unsigned int        size;				//item->unit_size * get_size_attribute(item->size_attr)
void                *data;				//clSVMAlloc()
cl_mem              device;
cl_mem_ext_ptr_t    ext_attr;
```

重要变量：local_mem

```
size_attr_ctrl_t local_size_ctrl[] = {
	{size_attr  = SIZE_IN_EDGE, scale = EDEG_MEMORY_SIZE},
	{size_attr  = SIZE_IN_VERTEX, scale = VERTEX_MEMORY_SIZE},
    {size_attr  = SIZE_USER_DEFINE, scale = 1}
};
```

## he_mem_lookup_t

```
unsigned int    id;
he_mem_t        *p_mem;   
```

重要变量：mem_list

```
he_mem_lookup_t lut_item;
lut_item.id = item->id;
lut_item.p_mem = item;
mem_list.push_back(lut_item);
```

## graphInfo

```
int vertexNum;
int compressedVertexNum;
int edgeNum;
int blkNum;
```

# 函数

1. `void *get_host_mem_pointer(int id)`

   返回mem_list中id对应的p_mem

2. 