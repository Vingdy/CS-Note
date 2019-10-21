# Context

上下文包，目的是为了让子goroutine可以被父goroutne控制

使用例子：main函数创建了两个协程，一个结束，另一个无法通知到，怎么办

main创建一个ctxcont.WithCancel(ctxcont.backGround())

然后用返回的ctxParent（第一个返回值）创建一个ctxChild传入另一个函数中

然后等到时机调用cancelParent（第二个返回值），第二协程就可以用ctx.Done()来判断了

```
package main
import (
    "context"
    "fmt"
    "time"
)
func sleepRandom_1(stopChan chan struct{}) {
    i := 0
    for {
        time.Sleep(1 * time.Second)
        fmt.Printf("This is sleep Random 1: %d\n", i)
        i++
        if i == 5 {
            fmt.Println("cancel sleep random 1")
            stopChan <- struct{}{}
            break
        }
    }
}
func sleepRandom_2(ctx context.Context) {
    i := 0
    for {
        time.Sleep(1 * time.Second)
        fmt.Printf("This is sleep Random 2: %d\n", i)
        i++
        select {
        case <-ctx.Done():
            fmt.Printf("Why? %s\n", ctx.Err())
            fmt.Println("cancel sleep random 2")
            return
        default:
        }
    }
}
func main() {
    ctxParent, cancelParent := context.WithCancel(context.Background())
    ctxChild, _ := context.WithCancel(ctxParent)
    stopChan := make(chan struct{})
    go sleepRandom_1(stopChan)
    go sleepRandom_2(ctxChild)
    select {
    case <- stopChan:
        fmt.Println("stopChan received")
    }
    cancelParent()
    for {
        time.Sleep(1 * time.Second)
        fmt.Println("Continue...")
    }
}
```

 https://www.bbsmax.com/A/1O5E3Dg3z7/ 

##### 接口源码

```
//  context 包里的方法是线程安全的，可以被多个 goroutine 使用  
type Context interface {             
    // 当Context 被 canceled 或是 times out 的时候，Done 返回一个被 closed 的channel    
    Done() <-chan struct{}      
  
    // 在 Done 的 channel被closed 后， Err 代表被关闭的原因 
    Err() error
  
    // 如果存在，Deadline 返回Context将要关闭的时间
    Deadline() (deadline time.Time, ok bool)
  
    // 如果存在，Value 返回与 key 相关了的值，不存在返回 nil
    Value(key interface{}) interface{}
}
```





