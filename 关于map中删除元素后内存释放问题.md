2019-08-01 参加了 [Go 夜读活动第53期](https://github.com/developer-learning/reading-go/issues/441)，时间有限，杨文大佬并没有详细讲述[所有的例子](https://github.com/yangwenmai/examples/blob/master/map-example)
，后来自己试验了一下，对其中几个有点疑惑。

1. [https://github.com/yangwenmai/examples/blob/master/map-example/mem_stats_demo.go](https://github.com/yangwenmai/examples/blob/master/map-example/mem_stats_demo.go)
```go
package main

import (
	"runtime"
	"log"
)

var intMap map[int]int
var cnt = 8192

func main() {
	printMemStats()
	
	initMap()
	runtime.GC()
	printMemStats()
	
	log.Println(len(intMap))
	for i := 0; i < cnt; i++ {
		delete(intMap, i)
	}
	log.Println(len(intMap))
	
	runtime.GC()
	printMemStats()
	
	intMap = nil
	runtime.GC()
	printMemStats()
}

func initMap() {
	intMap = make(map[int]int, cnt)
	
	for i := 0; i < cnt; i++ {
		intMap[i] = i
	}
}

func printMemStats() {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	log.Printf("Alloc = %v TotalAlloc = %v Sys = %v NumGC = %v\n", m.Alloc/1024, m.TotalAlloc/1024, m.Sys/1024, m.NumGC)
}
```

2. [https://github.com/yangwenmai/examples/blob/master/map-example/memory_leak_demo.go](https://github.com/yangwenmai/examples/blob/master/map-example/memory_leak_demo.go)

```go
package main

import (
	"fmt"
	"runtime"
	"runtime/debug"
)

func main() {
	a := make(map[int]int)
	
	stats := new(runtime.MemStats)
	runtime.ReadMemStats(stats)
	fmt.Printf("Before: %5.2fk\n", float64(stats.Alloc)/1024)
	previous := stats.Alloc
	
	for i := 1000000; i > 0; i-- {
		a[i] = 0
	}
	
	for k := range a {
		delete(a, k)
	}
	
	debug.FreeOSMemory()
	fmt.Printf("len(a)=%v\n", len(a))
	
	runtime.ReadMemStats(stats)
	fmt.Printf("After: %5.2fk    Added: %5.2fk\n", float64(stats.Alloc)/1024, float64(stats.Alloc-previous)/1024)
	
	runtime.GC()
	runtime.ReadMemStats(stats)
	fmt.Printf("After: %5.2fk    Added: %5.2fk\n", float64(stats.Alloc)/1024, float64(stats.Alloc-previous)/1024)
	
	return
}
```

这两个程序都是先初始化一个含有很多个元素的map，打印memory情况，然后逐个delete所有元素，再次打印memory情况，但在再次打印时，第一个程序中的memory没有被释放，而第二个程序的memory已经被释放。

有什么区别呢？试验表明，并不是 debug.FreeMemory 导致的区别（不管调用 FreeMemory+GC 还是只调用 GC内存都会被释放）。

真正的原因是 第一个程序中的map是局部变量，而第二个程序中的map是全局变量，局部变量在delete所有元素后内存会释放，而全局变量只有在将map设置为nil后内存才会释放。


[Go 夜读](https://github.com/developer-learning/reading-go) 是个非常棒的线上分享活动，利用有限的时间参加过两次直播，每次都很有收获，赶不上直播还可以看回放。
