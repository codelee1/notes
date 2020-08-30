### 一、什么是内存对齐

内存对齐应该是编译器的"管辖范围"，编译器为程序中的每个"数据单元"安排到适当的位置上，以期望程序内存的更合理高效使用。

对大多数程序员来说，内存对齐对他们来说都应该是"透明的"，也就是可以不用知道的，有些编程语言的编译器还会自动的进行内存对齐优化。

但是如果你想了解底层的秘密，内存对齐就应该是你必须了解的一个知识点。

这里我们先看一个例子（本人机子是64位）：

````
type A1 struct {
//字段名  类型  所占字节
	a  bool    // 1
	b  int32   // 4
	c  byte    // 1
	d  int64   // 8
	e  int8    // 1
}
````

让我们来估算一下这个A1结构体的占用大小，应该是：1+4+1+8+1=15字节？看上去应该是这样的，但是我们可以通过unsafe.SizeOf(x ArbitraryType)方法来查看结构体大小：

````
fmt.Printf("A1 size is : %d\n", unsafe.Sizeof(A1{}))

output:A1 size is : 32
````

为什么是32个字节呢？

这就涉及到内存对齐，其实也的规则也很简单，就两条，一个是关于结构体各自字段之间的，另一个是关于结构体本身的，两条规则共同作用，导致结构体A1所占用字节为32位。

#### 对齐规则：

讲规则之前，必须要了解的一个名词就是"默认对齐系数"，"默认对齐系数"和你的系统有关，简而言之，比如你的电脑是64位的，那么对齐就是64/8=8，32位的就是32/8=4。

不管结构体字段或者整个结构体多大，最大对齐系数都以默认对齐系数为准，比如我本机是64位，默认对齐系数就是8。

还有一个前提知识点是，每个结构体的第一个字段的偏移量是从零开始

规则一：结构体内第一个字段的偏移量(offset)是0，以后每个数据成员的对齐系数为（offset）：
对齐系数 = min(默认对齐系数, 字段类型的大小)。也就是说取默认对齐系数和结构体当前需要对齐字段的大小之间的最小值。

规则二：结构体每个字段对齐后，整个结构体也需要对齐，结构体对齐系数（结构体大小起始值）= min(默认对齐系数，结构体字段最大类型的值)，

怎么理解这两个规则了，我们先理解清楚对齐系数
对于字段来说，对齐系数的整数倍就是该字段在结构体的起始偏移量， 也就是说，起始偏移量需要以对其系数的整数倍递增，但是需累加之前已使用的地址。
比如A1结构体的b字段，对齐系数=min(8,4) = 4，字段大小4，那么b字段的偏移量为4，此时需往a字段与b字段中填充3个字节，以达到补齐效果。

对于结构体来说，更好理解了，对齐系数使用规则也是一样，结构体大小为结构体对齐系数的最小整数倍，即：SizeOf(A1) = (结构体对齐系数 * n)，但需大于当前结构体字段大小的累加值。

如下：
````
type A1 struct {
//    字段名  类型  所占字节（对齐系数） 起始偏移量     当前大小
	 a  bool   // 1                0            1=1
//padding   -         3                -            1+3=4
	 b  int32  // 4              4*1=4          1+3+4=8
	 c  byte   // 1             1*0+8=8         1+3+4+1=9
//padding   -         7                -            1+3+4+1+7=16
	 d  int64  // 8             8*2=16          1+3+4+1+7+8=24
	 e  int8   // 1             1*0+24=24       1+3+4+1+7+8+1=25
//padding   -         7                -            1+3+4+1+7+8+1+7=32
}
````
SizeOf(A1) = 8 * 4 = 32 ,32 > 25；

go的unsafe包也提供了Alignof(x ArbitraryType)来获取字段或结构体的对齐值，Offsetof(x ArbitraryType)来获取字段的偏移量，我们也可以用这两个方法验证我们的想法。


### 二、为什么需要内存对齐

认真的同学应该可以从上述例子中看到为什么需要内存对齐的端倪，A1结构体的a和b字段之间、c和d字段以及最后的结构体之间的，
存在的padding(填充)，其实是多于的，如果我们能尽可能的减少这些填充，我们的结构体就可以更加紧凑，内存占用更小。但是怎么弄呢？

我们只需要按照内存对齐规则，然后简单调整一下A1结构体之间的字段即可:
````
type A1 struct {
	d int64 // 8
	b int32 // 4
	a bool  // 1
	c byte  // 1
	e int8  // 1
}
````

我们可以来按照规则来算一下，改变字段顺序后的结构体A1的大小是多少：
````
type A1Rearrange struct {
//    字段名  类型  所占字节（对齐系数） 起始偏移量     当前大小
	 d  int64  // 8                 0            8
	 b  int32  // 4              4*0+8=8       8+4=12
	 a  bool   // 1              1*0+12=12     8+4+1=13
	 c  byte   // 1              1*0+13=13     8+4+1+1=14
	 e  int8   // 1              1*0+13=14     8+4+1+1+1=15
//padding   -         1                -           8+4+1+1+1+1=16
}
````
SizeOf(A1Rearrange) = 8 * 2 = 16 ,16 > 15；

只是简单调换了一下字段顺序，对结构体A来说，就可以减少一半的大小。

这也间接回答了为什么需要内存对齐的一个原因，内存对齐可以使我们的结构体更加紧凑，程序更加高效，当我们需要编写在性能（cpu、memory）有要求的程序时，
或者需要优化一下代码时，内存对齐是我们需要考虑的一个点。

还有一些大多数参考解释的原因是：

1、平台原因(移植原因)：不是所有的硬件平台都能访问任意地址上的任意数据的；某些硬件平台只能在某些地址处取某些特定类型的数据，否则抛出硬件异常。

2、性能原因：数据结构(尤其是栈)应该尽可能地在自然边界上对齐。原因在于，为了访问未对齐的内存，处理器需要作两次内存访问；而对齐的内存访问仅需要一次访问。

![twice_read](https://raw.githubusercontent.com/codelee1/uploads/master/memory_alignment/twice_read.jpg "twice_read")

在上图中，假设从 index = 1 开始读取，将会出现很崩溃的问题。因为它的内存访问边界是不对齐的。因此 CPU 会做一些额外的处理工作。如下：

cpu 首次读取未对齐地址的第一个内存块，读取0 - 3字节。并移除不需要的字节 0。

cpu 再次读取未对齐地址的第二个内存块，读取4-7字节。并移除不需要的字节5、6、7字节。

合并 1-4 字节的数据，合并后放入寄存器。

从上述流程可得出，不做“内存对齐”是一件有点"麻烦"的事。因为它会增加许多耗费时间的动作。

而假设做了内存对齐，从index = 0开始读取4个字节，只需要读取一次，也不需要额外的运算。


### 三、go的内存对齐

除了知道常用的int，byte，bool类型的的大小是多少字节，那么go的map，切片，接口等类型的值是多少呢？以及为什么呢？

先看例子：
````
type A2 struct {
	arr [2]int8  // 2
	sl  []int32  // 24 切片结构体字段为：uintptr的Data，int类型的Len，int类型的Cap，所占字节为24
	ptr *int64   // 8
	st  struct { // 16，和结构体内的字段类型有关
		str string // string类型字段为：uintptr的Data，int类型的Len，所占字节为16
	}
	m map[string]int32 // 8 map结构有很多字段的，但是只使用map的的指针值
	i interface{}      //16
}

a := A2{}
fmt.Printf("A2.arr size(array): %d\n", unsafe.Sizeof(a.arr))
fmt.Printf("A2.sl size(slice): %d\n", unsafe.Sizeof(a.sl))
fmt.Printf("A2.ptr size(pointer): %d\n", unsafe.Sizeof(a.ptr))
fmt.Printf("A2.st size(struct): %d\n", unsafe.Sizeof(a.st))
fmt.Printf("A2.m size(map): %d\n", unsafe.Sizeof(a.m))
fmt.Printf("A2.i size(interface): %d\n", unsafe.Sizeof(a.i))
fmt.Printf("A2 size(struct): %d\n", unsafe.Sizeof(a))

output:

A2.arr size(array): 2
A2.sl size(slice): 24
A2.ptr size(pointer): 8
A2.st size(struct): 16
A2.m size(map): 8
A2.i size(interface): 16
A2 size(struct): 80
````

通过输出我们可以知道各个类型占用多少字节，但是为什么呢？
````
type A2 struct {
	arr [2]int8  // 2 数组大小与当前数组元素类型和数量有关，如当前int8，大小为1字节，数量为2，所占字节为2
	sl  []int32  // 24 切片结构体字段为：uintptr的Data，int类型的Len，int类型的Cap，所占字节为24
	ptr *int64   // 8 我的机子为64位，所以指针类型为8个字节
	st  struct { // 16，和结构体内的字段类型有关
		str string // string类型字段为：uintptr的Data，int类型的Len，所占字节为16
	}
	m map[string]int32 // 8 map结构有很多字段的，但是只使用map的的指针值，所以只占8字节
	i interface{}      //16 interface结构一个为iface的结构体，有一个tap指针和data指针；一个为eface结构体，有一个_type指针和一个data指针，所以大小为18
}

````
可以从源码中看到端倪：

string源码为(reflect/value.go)：
````
type StringHeader struct {
	Data uintptr
	Len  int
}
````

slice源码为(reflect/value.go)：
````
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
````

map源码为(runtime/map.go)：
````
type hmap struct {
	count     int 
	flags     uint8
	B         uint8
	noverflow uint16
	hash0     uint32
	buckets    unsafe.Pointer
	oldbuckets unsafe.Pointer
	nevacuate  uintptr
	extra *mapextra 
}
````

interface源码为(runtime/runtime2.go)：
````
type iface struct {
	tab  *itab
	data unsafe.Pointer
}

type eface struct {
	_type *_type
	data  unsafe.Pointer
}
````
到此，go基本上的类型值的大小我们都知道了，也知道结构体嵌套后的大小、数组的大小怎么计算，切片、字符串等大小是多少也知道了，
还知道map虽然源码多，但是只是用它的指针值作为字段大小使用。

### 四、go内存对齐分析与排版建议工具

1、内存分析工具
````
go get -u honnef.co/go/tools
go install honnef.co/go/tools/cmd/structlayout
go install honnef.co/go/tools/cmd/structlayout-pretty
````

2、svg图生成工具：

````
go get github.com/ajstarks/svgo/structlayout-svg
````

3、字段排版建议工具
````
go install honnef.co/go/tools/cmd/structlayout-optimize
````


如对上字段A2的分析与排版建议命令

分析：
````
structlayout -json teststh A1|structlayout-svg -t "a1-padding" >a1.svg
````
命令会在当前文件夹下生成A2结构体的svg内存分析图:

![a1.avg](https://raw.githubusercontent.com/codelee1/uploads/master/memory_alignment/a1.svg "a1.avg")
![a1.avg](https://raw.githubusercontent.com/codelee1/uploads/master/memory_alignment/a1Rearrange.svg "a1.avg")

排版建议(粗暴方式，按照对齐系数的递减来重排字段)：
````
structlayout -json teststh A1|structlayout-optimize -r

output:
A1.d int64: 0-8 (size 8, align 8)
A1.b int32: 8-12 (size 4, align 4)
A1.a bool: 12-13 (size 1, align 1)
A1.c byte: 13-14 (size 1, align 1)
A1.e int8: 14-15 (size 1, align 1)
padding: 15-16 (size 1, align 0)
````

当然，除了一个个检查类型外，还有可以批量检查内存对齐的工具

golangci-lint
````
官网：https://golangci-lint.run
mac安装： brew install golangci/tap/golangci-lint

````
命令：
````
golangci-lint run --disable-all --enable maligned struct_padding.go 
struct_padding.go:3:9: struct of size 32 bytes could be of size 16 bytes (maligned)
type A1 struct {
        ^
````
报错信息可以告诉你当前结构体A1的大小应该为16，但是实际上却是32，你可以优化一下。

详细使用请参考官网文档。


### 五、one more thing

int64字段类型在32位系统的原子操作

实现方式一：sync.WaitGroup
````
type WaitGroup struct {
	noCopy noCopy

	state1 [3]uint32
}

func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
	if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
		return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
	} else {
		return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
	}
}
````

sync.WaitGroup的状态使用state1字段实现，其中state1是3个字节的数组，我们可以看一下它源码即可知道，
它根据当前的系统的位数，来返回数组中不同元素，32位返回数组最后一个元素，64位返回数组前两个元素。

当然还有一种更简单的实现方式，或者说比较折中的方式，只需要对想要同步的字段加锁即可。

