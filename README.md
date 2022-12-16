# 内存管理系统实验

## 实验内容

请编写模拟程序演示内存分配，包括：基本内存分配，虚拟内存分配。

基本内存分配包括连续内存分配和分页内存分配，需要实现内存的初始化，空闲内存管理，内存分配，内存回收等机制。

虚拟内存分配需要实现动态地为进程分配物理帧，完成相应页表映射，实现请求调页和页面置换算法。

## 实验环境

* Windows11
* Visual Studio 2022

## 程序设计

本人一共为本次实验实现了两个模拟程序，第一个为连续内存分配模拟，第二个为分页式内存管理加上虚拟内存管理，接下来分别对两个程序进行介绍。

### 连续内存分配模拟

#### 设计思路

程序随机生成一定数量的进程，对于每个进程随机生成其执行时间和花费内存空间大小，并将其交给内存管理系统。内存管理系统分别实现了首次适配、最优适配、最差适配算法以及各算法的抢占与非抢占版本。内存分配系统收到进程后，为其分配内存，并等待程序的运行时间结束将其释放；若无法分配需求的大小空间，将进程加入进程队列，等待其他进程运行结束释放内存后再分配个给进程队列中的进程。

#### 数据结构

连续内存一共有两个重要的数据结构，分别是孔大小队列以及进程队列。

孔大小由一个结构体`Hole`表示，结构中包含孔的起始地址和大小用于表示内存中的一段空间，同时重载了`<`运算符将孔大小作为比较标准，以下为`Hole`具体实现：

    struct Hole
	{
		bool operator<(const Hole& hole) const { return size < hole.size; }

		//起始地址
		int begin_index;
		//大小
		int size;
	};

孔大小队列使用三个继承自抽象类`HoleList`的类实现，分别为`FirstFitHoleList`，`BestFitHoleList`，`WorstFitHoleList`，分别实现首次适配，最优适配和最差适配。以下为`HoleList`具体实现：

    class HoleList
	{
	public:
		virtual void AddHole(const Hole& hole) = 0;

		virtual const int Allocate(const int& size) = 0;

		virtual const Hole MergeHole(const int& begin_index, const int& size) = 0;

		virtual const string ToString() = 0;
	};

* virtual void AddHole(const Hole& hole) 向孔序列中添加孔
* virtual const int Allocate(const int& size) 分配大小为`size`的空间，返回该空间的起始地址
* virtual const Hole MergeHole(const int& begin_index, const int& size) 向孔序列中添加起始地址为`begin_index`，大小为`size`的空间，孔序列会将该空间相邻的孔合并成一个更大的孔
* virtual const string ToString() 输出孔队列状态，用于程序调试

子类`FirstFitHoleList`采用无序可变数组`vector<Hole> holes_`存储孔序列；子类`BestFitHoleList`和`WorstFitHoleList`采用有序集合`multiset<Hole> holes_`存储孔序列。

进程队列由简单的可变数组`vector<Process*> process_queue_`表示。

#### 详细实现

首先程序随机生成成一定数量的进程，对于每个进程随机生成其执行时间和花费内存空间大小，并将其交给内存执行。

以下为进程交给连续内存执行的流程图：

![连续内存程序流程图](/img/%E8%BF%9E%E7%BB%AD%E5%86%85%E5%AD%98%E7%A8%8B%E5%BA%8F%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

函数`Allocate`将依据适配策略而有所不同。首次适配将在无序可变数组中从头按顺序查找第一个满足要求的孔；最优适配将在有序集合中找到第一个大于需求大小的孔；最差适配将判断最大的孔是否满足要求。

函数`TestQueue`将依据是否允许抢占而有所不同。允许抢占的情况下，将会判断整个进程队列，并将可以在合并后大孔空间内运行的进程集合取出；不允许抢占的情况下，只会判断进程队列的前缀部分，从队首开始判断，直到大孔空间不在满足更多地进程。

#### 输入输出

首先编译程序获得可执行程序`contiguous_memory.exe`

程序的输入如下：

    contiguous_memory MEMORY_SIZE PROCESS_COUNT MAX_TIME MAX_MEMORY -ff|-bf|-wf -p|-fp

* `MEMORY_SIZE`为内存空间大小，单位字节
* `PROCESS_COUNT`为同时允许在内存中的进程数量
* `MAX_TIME` 进程最大执行时间
* `-ff|-bf|-wf`分别表示首次适配，最优适配，最差适配
* `-p|-fp`分别表示允许抢占和不允许抢占

程序输出同时输出到标准输出和文件`output.txt`，输出分为三列

第一列为进程随机创建信息，格式如下：

    Create PROCESS_ID SIZE TIME

其中`PROCESS_ID`为进程唯一标识，`SIZE`为所需空间，`TIME`为执行时间

第二列为进程分配和释放空间的运行信息，格式如下

    Allocate PROCESS_ID (BEGIN_INDEX, END_INDEX) TIME
    Reclaim PROCESS_ID (BEGIN_INDEX, END_INDEX)

其中`PROCESS_ID`为进程唯一标识，`BEGIN_INDEX`和`END_INDEX`分别为分配空间的起始和结束地址，`TIME`为执行时间

第三列为内存调试信息，格式如下

    --------------------------------------------------------
    (BEGIN_INDEX1, END_INDEX1) (BEGIN_INDEX2, END_INDEX2) ...
    --------------------------------------------------------
    -> PROCESS_ID1 -> PROCESS_ID2 -> ...

其中`(BEGIN_INDEX, END_INDEX)`表示孔序列中孔起始位置和结束位置，`-> PROCESS_ID`表示进程队列中还未执行的进程。

以下为程序输出范例：

![连续内存输出范例](/img/连续内存输出范例.png)

