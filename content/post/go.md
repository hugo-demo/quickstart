---
title: "Go语言|基于channel实现的并发安全的字节池"
date: 2020-09-16T17:19:38+08:00
toc : true
---

## A
字节切片[]byte是我们在编码中经常使用到的，比如要读取文件的内容，或者从io.Reader获取数据等，都需要[]byte做缓冲。

1
2
func ReadFull(r Reader, buf []byte) (n int, err error)
func (f *File) Read(b []byte) (n int, err error)
以上是两个使用到[]byte作为缓冲区的方法。那么现在问题来了，如果对于以上方法我们有大量的调用，那么就要声明很多个[]byte，这需要太多的内存的申请和释放，也就会有太多的GC。

MinIO 的字节池
这个时候，我们需要重用已经创建好的[]byte来提高对象的使用率，降低内存的申请和GC。这时候我们可以使用sync.Pool来实现，不过最近我在研究开源项目MinIO的时候，发现他们使用channel的方式实现字节池。

1
2
3
4
5
## B
type BytePoolCap struct {
	c    chan []byte
	w    int
	wcap int
}
BytePoolCap结构体的定义比较简单，共有三个字段：
### B.a

c是一个chan，用于充当字节缓存池
w是指使用make函数创建[]byte时候的len参数
wcap指使用make函数创建[]byte时候的cap参数
有了BytePoolCap结构体，就可以为其定义Get方法，用于获取一个缓存的[]byte了。

 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
func (bp *BytePoolCap) Get() (b []byte) {
	select {
	case b = <-bp.c:
	// reuse existing buffer
	default:
		// create new buffer
		if bp.wcap > 0 {
			b = make([]byte, bp.w, bp.wcap)
		} else {
			b = make([]byte, bp.w)
		}
	}
	return
}

### B.c

以上是采用经典的select+chan的方式，能获取到[]byte缓存则获取，获取不到就执行default分支，使用make函数生成一个[]byte。

从这里也可以看到，结构体中定义的w和wcap字段，用于make函数的len和cap参数。

有了Get方法，还要有Put方法，这样就可以把使用过的[]byte放回字节池，便于重用。

1
2
3
4
5
6
7
8
func (bp *BytePoolCap) Put(b []byte) {
	select {
	case bp.c <- b:
		// buffer went back into pool
	default:
		// buffer didn't go back into pool, just discard
	}
}
Put方法也是采用select+chan，能放则放，不能放就丢弃这个[]byte。

使用BytePoolCap
已经定义好了Get和Put就可以使用了，在使用前，BytePoolCap还定义了一个工厂函数，用于生成*BytePoolCap，比较方便。

1
2
3
4
5
6
7
func NewBytePoolCap(maxSize int, width int, capwidth int) (bp *BytePoolCap) {
	return &BytePoolCap{
		c:    make(chan []byte, maxSize),
		w:    width,
		wcap: capwidth,
	}
}
把相关的参数暴露出去，可以让调用者自己定制。这里的maxSize表示要创建的chan有多大，也就是字节池的大小，最大存放数量。

1
2
3
4
5
bp := bpool.NewBytePoolCap(500, 1024, 1024)
buf:=bp.Get()
defer bp.Put(buf)

//使用buf,不再举例
以上就是使用字节池的一般套路，使用后记得放回以便复用。

和sync.Pool对比
两者原理基本上差不多，都多协程安全。sync.Pool可以存放任何对象，BytePoolCap只能存放[]byte，不过也正因为其自定义，存放的对象类型明确，不用经过一层类型断言转换，同时也可以自己定制对象池的大小等。

关于二者的性能，我做了下Benchmark测试，整体看MinIO的BytePoolCap更好一些。

1
2
3
4
5
6
var bp = bpool.NewBytePoolCap(500, 1024, 1024)
var sp = &sync.Pool{
	New: func() interface{} {
		return make([]byte, 1024, 1024)
	},
}
模拟的两个字节池，[]byte的长度和容量都是1024。然后是两个模拟使用字节池，这里我启动500协程，模拟并发，使用不模拟并发的话，BytePoolCap完全是一个[]byte的分配，完全秒杀sync.Pool，对sync.Pool不公平。

 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
func opBytePool(bp *bpool.BytePoolCap) {
	var wg sync.WaitGroup
	wg.Add(500)
	for i := 0; i < 500; i++ {
		go func(bp *bpool.BytePoolCap) {
			buffer := bp.Get()
			defer bp.Put(buffer)
			mockReadFile(buffer)
			wg.Done()
		}(bp)
	}
	wg.Wait()
}

func opSyncPool(sp *sync.Pool) {
	var wg sync.WaitGroup
	wg.Add(500)
	for i := 0; i < 500; i++ {
		go func(sp *sync.Pool) {
			buffer := sp.Get().([]byte)
			defer sp.Put(buffer)
			mockReadFile(buffer)
			wg.Done()
		}(sp)
	}
	wg.Wait()
}
接下来就是我模拟的读取我本机文件的一个函数mockReadFile(buffer):

1
2
3
4
5
6
7
8
9
func mockReadFile(b []byte) {
	f, _ := os.Open("water")
	for {
		n, err := io.ReadFull(f, b)
		if n == 0 || err == io.EOF {
			break
		}
	}
}
然后运行go test -bench=. -benchmem -run=none 查看测试结果：

1
2
3
pkg: flysnow.org/hello
BenchmarkBytePool-8         1489            979113 ns/op           36504 B/op       1152 allocs/op
BenchmarkSyncPool-8         1008           1172429 ns/op           57788 B/op       1744 allocs/op
从测试结果看BytePoolCap在内存分配，每次操作分配字节，每次操作耗时来看，都比sync.Pool更有优势。

小结
很多优秀的开源项目，可以看到很多优秀的源代码实现，并且会根据自己的业务场景，做出更好的优化。
