---
title: database storage structure
date: 2022-05-09 20:09:26
tags: database
---
# Database Storage structure
本文介绍数据库在磁盘上的存储格式与层次:Table->Block->Record

## Record Structure
数据在磁盘上布局的基本单位:record;对关系型数据库而言，一条record就是对应关系中的一行，而对NoSQL数据库而言，record并不是数据存储的唯一方式。

### 定长record
&emsp;&emsp;如果一条record的长度是固定值时，数据库存储起来会比较容易，并且record的结构也会相对简单。如下面定长record图所示，record分为两大部分:header和data;
![定长record](fixed_length_record.png)
&emsp;&emsp;如图所示:record的前端为header，后部分为data部分。其中header内包含了to schema、整条record长度、时戳等信息，其中schema可以简单理解为record的是表ID属性，但其实并不准确，后续会详细介绍schema的概念，而to schema是指向该record对应的schema的指针或者其他一些访问方式，不同数据库的实现方式具有一定不同。data部分则是直接将字段值按顺序存放，而没有其他的间隔或者区分方式，这是因为schema中保存了不同字段的长度，在需要解析的时候可以根据schema把需要的字段直接定位出来。

### 变长record
&emsp;&emsp;但在实际使用中record为定长的可能性比较低，因为很可能会存在某字段需要使用不同长度的变量来描述，同时因为field为变长，record也必须是变长的。下面是一张关于变长record可能的结构图。
![变长字段的record](non_fixed_length_record.png)
&emsp;&emsp;可以看到，与定长record相似，都是分为header和data部分。header部分与定长record相同，都是一些record的信息，如to schema整条record的长度和时戳等，但是data部分与定长record却有较大的不同，变长record在存放具体字段值时与定长record相同，并没有采用特殊的分隔符等操作，但是在存放之前有一段指针数组，其中存放的时指向不同字段起始位置的指针，值得强调的是:这里说的指针并非内存中的指针，而是相对偏移量。通过这个偏移量数组即可确定不同字段的长度和内容从而做到识别字段。
 
&emsp;&emsp;除了record中有变长字段的情况，还有一种情况也会导致record为变长record——具有重复字段，如下图所示:name、address都是定长字段，而movies也是定长字段，但特殊的是在record中可能会出现多个movies的情况，因此record也无法是一个定长的record而是一条变长record来保证上述情况的保存。
![重复字段的record](repeated_field_record.png)
解决方案与具有变长字段类型，就是在data部分增加一个指针数组，指向不同的字段起始位置来区别不同字段。如果从变长字段的角度来理解的话，则movies不再是一个定长字段而是一个变长字段，这样有重复字段的处理方式其实与有变长字段的处理方式是一模一样的。

&emsp;&emsp;在变长record中还有一种情况——变字段record，即是一条record对应的字段数量、字段类型和字段含义是不相同的，而是record中具有一定的自解释能力，严格来说这已经不再属于关系型数据库的范围，因此在这里不做详细展开。

### My Record Implementation
&emsp;&emsp;该记录为数据库原理与实现课程老师给出雏形，后参照InnoDB和MySQL的实现设计而成。具体如图所示:
![My Record](my_record.png)
其中header为固定的1Byte大小，其结构为:  
<center>
<B>| T | M | x | x | x | x | x | x |</B>
</center>  
其中T为tombstone标志，用来说明这条record是否存活，M为最小记录，x均为保留位供其他使用。在使用过程中需要掩码参与进行操作。
特别说明:上图虽然header放在offset数组后面，但在代码实际实现中，header仍然位于record的最前方，而其他 相对位置不变。

## Block Structure
&emsp;&emsp;Block是数据库的基本IO单位，当进程需要调用数据库中的record的时，至少需要调入一整个Block而无法仅仅访问一个record。

### 简单Block实现
![简单Block](simple_block.png)
&emsp;&emsp;如图所示:Block一般由header和record构成，头部主要包含指向后继Block的指针，标记记录是否延续到下一个block，时戳，剩余空间大小，剩余空间起始位置等关键信息。而record区则是一个变长的数组，其中每一个元素都是一个完整的record。

### 相对复杂但更合理的Block实现
![较复杂Block](complex_block.png)
&emsp;&emsp;上图是Block的另一种可行的Block实现方案，前部分与简单Block相同，都是有着header和record区，但是在record区域后方，这里相比简单Block，添加了一个tail来为record的数组进行排序，具体排序实现在My Block Implementation中进行详细说明。在record数组和tail之间存在free space，当有新record插入时，record数组向后延伸，而tail则向前延伸，从而压缩free space空间。

### My Block Implementation
![My Block](my_block.png)
&emsp;&emsp;由上图可以看见，Block的结构为：Block Header + Data Header + Data + Free Space + Slots + Trailer。其中Slots和Trailer共同作为tail。  
&emsp;&emsp;因为Block存在不同的类型，在实现时采用了基类和派生类的方法来进行不同Block的实现，uml图如下:
![Class Structure](block_implement.png)
其中Block作为基类，派生出了SuperBlock和MetaBlock类，而DataBlock为MetaBlock的直接派生。
``` c++
struct CommonHeader
{
    unsigned int magic;       // magic number(4B)
    unsigned int spaceid;     // 表空间id(4B)
    unsigned short type;      // block类型(2B)
    unsigned short freespace; // 空闲记录链表(2B)
};
```
其中magic为魔数，编码为小端时```0x31306264```，大端时为```0x64623031```，用来判断Block是否损坏和防止缓冲区攻击。