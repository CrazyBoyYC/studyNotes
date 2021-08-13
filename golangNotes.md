# 常识补充

1. 程序运行时会打开三个文件stdin stdout stderr，用于输入输出。结束时会关闭
2. 

# 基础知识点：

## GC

概念：

## 内存

1. 以32位操作系统为例（2^32为4G内存）：分为内核、用户。
2. 内核占3-4g为内核。
3. 用户分为：
   - .text(代码区) 只读
   - rodata(只读数据区)  全局常量
   - .data(数据区) 用于存放已初始化的全局变量
   - bss (未初始化数据区)
   - heap 堆 make new出来的对象
   - stack 栈 函数局部变量
4. 栈帧：用来给函数运行提供内存空间。取内存于stack上。当函数调用时，产生栈帧（连续分配）。函数调用结束，释放栈帧。
   - 栈帧存储：局部变量 形参数 （形参与局部变量等同）内存字段描述值
   - 栈帧的地址区段描述靠的是：栈顶指针 栈基指针，当出现新的栈帧时，上个栈帧存储一个叫 **内存字段描述值**的，用于记住它当时的栈基地址。

#### 内存逃逸

**简介：**golang程序变量会携带一组**校验数据**，用来证明它的整个生命周期是否在运行时**完全可知**。如果变量通过了这些校验，它就可以在**栈上分配**。否则就说它**逃逸**了，必须在**堆上分配**。

**查看逃逸命令**：go build -gcflags=-m main.go

**逃逸的典型情况：**

1. **在方法内把局部变量指针返回**
2. **发送指针或带有指针的值到channel** 在编译时，是没有办法知道哪个 goroutine 会在 channel 上接收数据。所以编译器没法知道变量什么时候才会被释放。
3. **在一个切片上存储指针或带指针的值** 一个典型的例子就是 []*string 。这会导致切片的内容逃逸。尽管其后面的数组可能是在栈上分配的，但其引用的值一定是在堆上。
4. **slice 的背后数组被重新分配了，因为 append 时可能会超出其容量( cap )。** slice 初始化的地方在编译时是可以知道的，它最开始会在栈上分配。如果切片背后的存储要基于运行时的数据进行扩充，就会在堆上分配。
5. **在 interface 类型上调用方法。** 在 interface 类型上调用方法都是动态调度的 —— 方法的真正实现只能在运行时知道。想像一个 io.Reader 类型的变量 r , 调用 r.Read(b) 会使得 r 的值和切片b 的背后存储都逃逸掉，所以会在堆上分配。

#### 内存泄漏

go是一门自己gc的语言，2分钟gc循环一次，如果内存泄漏，无非就是两种情况。

- 有goroutine泄漏，goroutine“飞了”，zombie goroutine没有结束，这个时候在这个goroutine上分配的内存对象将一直被这个僵尸goroutine引用着，进而导致gc无法回收这类对象，内存泄漏
- 有一些全局（或者生命周期和程序本身运行周期一样长的）的数据结构意外的挂住了本该释放的对象，虽然goroutine已经退出了，但是这些对象并没有从该类数据结构中删除，导致对象一直被引用，无法被回收。

## channel 

1. 双向chan可以转换为单向chan，反之不行 sendCh = ch
2. 单向 写： sendCh := make(chan <- int)
3. 是否读写成功：data, ok := <- ch

## select

1. 定义：

   - 用于监听channel上的数据流动，与select与switch类似，但每个case语句必须是一个IO操作。可读可写
   - 按顺序从头至尾评估每一个发送和接收语句。从所有可执行（未阻塞）语句中任意挑选一条来使用。如果没有可执行语句，则有default时使用default。否则被阻塞直到有一个通信可以执行下去
   - 自身不带循环，需要在外层加for

2. 基础语法

   - ```
     select {
     case <- chan1:
     case chan2 <- 1:
     default:
     	// 如果以上都没有成功，则进入default，一般不写default，避免忙轮询
     }
     ```

3. 超时使用：

   - ```
     c := make(chan int)
     o := make(chan int)
     go func() {
     	for {
     		// 每次都等5秒，没收到就退出了
     		select {
     		case v:= <-c:
     			fmt.println(v)
             case <-time.After(5 * time.Second):
             	fmt.Println("timeout")
             	o <- true
             	break
     		}
     	}
     }()
     <- o
     ```

     

4. 基本使用：

   ```go
   func send(out chan <-string)  {
   	out <- "yucheng1"
   	out <- "yucheng2"
   	out <- "yucheng3"
   	out <- "yucheng4"
   	close(out)
   }
   
   func recv(in <-chan string)  {
   	for {
   		select {
   			case tmp, has := <-in:
   				if has == false {
   					goto end
   				}
   				fmt.Println(tmp)
   			default:
   				time.Sleep(1000)
   		}
   	}
   end:
   }
   
   func main() {
   	ch := make(chan string, 10)
   	go func() {
   		send(ch)
   	}()
   	recv(ch)
   }
   ```

## 定时器

1. time.Timer

   ```
   type Timer struct {
   	C <-chan Time // 时间到时，会给此通道写入时间
   	r runtimeTimer
   }
   ```

   
   - 使用

   ```
   func main() {
   	// 在定时之间还可以做点别的事情
   	fmt.Println(time.Now())
   	mytimer := time.NewTimer(time.Second * 2)
   	nowTime := <-mytimer.C
   	fmt.Println(nowTime)
   	
   	// 类似sleep
   	nowTime := <-time.After(time.Second * 2)
   	
   	// 重置
   	mytimer.Reset(time.Second * 1)
   	// 停止
   	mytimer.Stop()
   }
   ```

2. time.ticker

   - 循环获取往ticker中的channel写入数据

   - ```
     // A Ticker holds a channel that delivers ``ticks'' of a clock
     // at intervals.
     type Ticker struct {
     	C <-chan Time // The channel on which the ticks are delivered.
     	r runtimeTimer
     }
     ```

## 锁

#### tips

互斥锁 读写锁不要与channel混用，容易产生隐性死锁

### 死锁

1. 原因：

   - 单go程自己死锁

     ```
     ch := make(chan int)
     ch <- 789 //写端阻塞
     num := <-ch
     ```

     

   - go程间channel访问顺序导致死锁

     - 使用channel一端读 一端写，有机会一同执行

   - 多go程，多channel交叉死锁

     - ```
       go fun() {
       	for {
       		select {
       			case num := <- ch1:
       				ch2 <- num
       		}
       	}
       }
       
       for {
       	select {
       		case num := <-ch2:
       			ch1 <- num
       	}
       }
       ```

### 互斥锁

1. channel完成同步

   - ```
     func f1(ch chan int) {
     	modifyShare()
     	ch <- 1
     }
     
     func f2(ch chan int) {
     	<- ch
     	modifyShare()
     }
     
     func main() {
     	go f1()
     	go f2()
     }
     ```

2. 通过锁，分享共享数据(建议锁：操作系统提供，建议你在编程时使用，不强制)

   - ```
     var mutex sync.Mutex
     
     func modifyShare() {
     	mutex.Lock()
     	doSomething()
     	mutex.unLock()
     }
     
     func f1(ch chan int) {
     	modifyShare()
     }
     
     func f2(ch chan int) {
     	modifyShare()
     }
     
     func main() {
     	go f1()
     	go f2()
     }
     ```

### 读写锁：

1. 读时共享，写时独占。写锁优先级比读锁高

2. 示例

   - ```
     // 以下代码将产生隐性死锁
     var num int
     func readGo(idx int) {
         for {
         	rwMutex.RLock()
     		fmt.Println(num)
     		rwMutex.RUnLock()
         }
     }
     
     func writeGo(out chan<- int, idx int) {
     	for {
     		rwMutex.Lock()
             num := rand.Intn(1000)
             out <- num
             time.Sleep(time.Second * 1)
             rwMutex.UnLock()
     	}
     }
     
     func main() {
     	quit := make(chan bool)
     	ch := make(chan int)
     	
     	for i:=0; i < 5; i++ {
     		go readGo(ch, i+1)
     	}
     	
     	for i:=0; i < 5; i++ {
     		go writeGo(ch, i+1)
     	}
     }
     ```

### 条件变量

1. 定义：非锁，但是常与锁一起使用。生产者、消费者写或读之前，先判断条件变量

2. 流程：判断条件变量、加锁、访问公共区、解锁、唤醒阻塞在条件变量上的对端

3. 结构体:

   - ```go
     type Cond struct {
     	noCopy noCopy
     
     	// L is held while observing or changing the condition
     	L Locker
     
     	notify  notifyList
     	checker copyChecker
     }
     ```

4. 常用函数：

   - Wait():check notifyListAdd unlock wait lock

     - ```
       //    c.L.Lock()
       //    for !condition() {
       //        c.Wait()
       //    }
       //    ... make use of condition ...
       //    c.L.Unlock()
       ```

   - Signal()

   - Broadcast()

## sync

go语言提供几个简单的原子操作

### sync/atomic

 1. Add()

 2. CAS(compare&swap)

 3. atomic.Value

    - store

    - load

    - ```
      var atomicVal atomic.Value
      str := "hello"
      // 类似小容器
      atomicValue.Store(str)
      newStr := atomicVal.Load()
      ```

### sync.WaitGroup

1. Add()
2. Done()
3. Wait

### sync.Once

 1. Once

    - ```
      // yucheng只会打印一次，无论是否多进程 真的只打一次
      func main() {
      	var once sync.Once
      	onceBody := func() {
      		fmt.Println("yucheng")
      	}
      
      	done := make(chan bool)
      	for i := 0; i < 10; i++ {
      		go func() {
      			once.Do(onceBody)
      			done <- true
      		}()
      	}
      	for i := 0; i < 10; i++ {
      		<-done
      	}
      }
      ```

## context

实现一对多的goroutine协作。当一个请求来时，会产生一个goroutine，可能会衍生出很多goroutine。context带着数据，贯穿全文。结束时ctx.Done这个channel收到信号

有两个根context：background和todo。没多大区别，常用background作为树最顶层的Context，不能被取消。

1. 生成子节点的方法：

   ```
   // 生成可撤销的Context（手动）
   func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
   // 生成可定时撤销的Context（定时撤销）
   func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
   // 也是生成可定时撤销的Context （定时撤销）
   func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
   // 不可撤销的Context,可以存一个kv的值
   func WithValue(parent Context, key, val interface{}) Context
   ```

2. 例子：

   - WithCancel()

   ```
       gen := func(ctx context.Context) <-chan int {
           dst := make(chan int)
           n := 1
           go func() {
               for {
                   select {
                   case <-ctx.Done(): //只有撤销函数被调用后，才会触发
                       return 
                   case dst <- n:
                       n++
                   }
               }
           }()
           return dst
       }
   
       ctx, cancel := context.WithCancel(context.Background())
       defer cancel()  //调用返回的cancel方法来让 context声明周期结束
   
       for n := range gen(ctx) {
           fmt.Println(n)
           if n == 5 {
               break
           }
       }
   ```

   

# 常用模型

## 生产者消费者

### 基础知识

1. 概念简述：生产者->缓冲区->消费者

2. 缓冲区：

   1. 解耦（降低生产者 和 消费者之间 耦合度）
   2. 处理并发（生产者、消费者 数量不对等时，能保持正常通信）
   3. 缓存 （生产者 消费者 数据处理速度不一致时，暂存数据）

### 实现

1. 使用channel作为缓冲区
   - 有缓冲：异步通信
   - 无缓冲：同步通信
2. 

