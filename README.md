# go_concurrency_example

# refrence:
[linux limit concurrency](https://www.youtube.com/watch?v=jA7aYSRKVTQ)
# introduction

假設現在有100個 job我們希望

透過goroutine的方式 讓每次可以並行執行10個goroutine

原本的寫法是

用waitGroup來把100 job加入

使用一個 integer channel found來接收執行結果
並使用一個size 為10的buffered channel limitCh來接收每次執行的task

然後每次執行完task之後把結果寫入found
並且把waitGroup 的count 減去1, 把limitCh的task讀出

最後的for loop 則在found被close 之前 會一直不斷接收found的結果
直到found close後
才會印出所有的結果result

在這個最出版本的code

有一個問題

就是 當bufferred channel limitCh被寫滿10的時候

limitCh就會被block 無法接收其他寫入的task

那如果這時寫入第11task

就會deadlock 無法運作

```golang
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

func main() {
	const concurrencyProcesses = 10 // limit the maximum number of concurrent reading process tasks
	const jobCount = 100

	var wg sync.WaitGroup
	wg.Add(jobCount)
	found := make(chan int)
	limitCh := make(chan struct{}, concurrencyProcesses)
	for i := 0; i < concurrencyProcesses; i++ {
		limitCh <- struct{}{}
		go func(val int) {
			defer func() {
				<-limitCh
				wg.Done()
			}()
            waitTime := rand.Int31n(1000)
            fmt.Println("job:", val, "wait time:", waitTime, "millisecond")
            time.Sleep(time.Duration(waitTime) * time.Millisecond)
            found <- val
		}(i)
	}
	go func() {
		wg.Wait()
		close(found)
	}()
	var results []int
	for p := range found {
		fmt.Println("Finished job:", p)
		results = append(results, p)
	}

	fmt.Println("result:", results)
}
```
想法一：

出問題的點在於 寫入limitCh的地方有可能遭到block

所以 把 寫入 limitCh的地方 也寫成goroutine

讓limitCh <-  struct{}{} 也在背景執行

這樣就不會block

也就是:
```golang
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

func main() {
	const concurrencyProcesses = 10 // limit the maximum number of concurrent reading process tasks
	const jobCount = 100

	var wg sync.WaitGroup
	wg.Add(jobCount)
	found := make(chan int)
	limitCh := make(chan struct{}, concurrencyProcesses)
	for i := 0; i < concurrencyProcesses; i++ {
        go func () {
            limitCh <- struct{}{}
        }()
		go func(val int) {
			defer func() {
				<-limitCh
				wg.Done()
			}()
            waitTime := rand.Int31n(1000)
            fmt.Println("job:", val, "wait time:", waitTime, "millisecond")
            time.Sleep(time.Duration(waitTime) * time.Millisecond)
            found <- val
		}(i)
	}
	go func() {
		wg.Wait()
		close(found)
	}()
	var results []int
	for p := range found {
		fmt.Println("Finished job:", p)
		results = append(results, p)
	}

	fmt.Println("result:", results)
}
```
然而 這樣做

由於執行寫入limitCh跟執行task的邏輯都在背景執行

因此 limitCh這個 bufferedChannel便無法達到只限制同時只執行10個task的效果

因此需要保持同時只執行10個的效果 必須使用其他的設計方式

設計如下

寫一個 integer channel queue

首先把所有的 task 逐步寫入 queue這個channel

直到結束 才把queue close

然後透過for loop 一次讀取10個

直到所有queue被讀取完畢

實做如下：
```golang
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

func main() {
	const concurrencyProcesses = 10 // limit the maximum number of concurrent reading process tasks
	const jobCount = 100

	var wg sync.WaitGroup
	wg.Add(jobCount)
	found := make(chan int)
	queue := make(chan int)
	go func(queue chan<- int) {
		for i := 0; i < jobCount; i++ {
			queue <- i
		}
		close(queue)
	}(queue)
	for i := 0; i < concurrencyProcesses; i++ {
		go func(val int) {
			for val := range queue {
				defer wg.Done()
				waitTime := rand.Int31n(1000)
				fmt.Println("job:", val, "wait time:", waitTime, "millisecond")
				time.Sleep(time.Duration(waitTime) * time.Millisecond)
				found <- val
			}
		}(i)
	}
	go func() {
		wg.Wait()
		close(found)
	}()
	var results []int
	for p := range found {
		fmt.Println("Finished job:", p)
		results = append(results, p)
	}

	fmt.Println("result:", results)
}
```
這樣就能夠保證每次只有10個 task被同時執行

使用這樣的設計 必須要了解到 non-buffered-channel的 讀寫的block特性

還有如何使用waitGroup的概念

## my hackmd record

[go_concurrency_example](https://hackmd.io/E-7F9GYrR3CPUkRC9lIw1A)