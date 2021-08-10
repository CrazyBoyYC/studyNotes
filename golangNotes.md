# 常识补充

1. 程序运行时会打开三个文件stdin stdout stderr，用于输入输出。结束时会关闭
2. 





# 基础知识点：

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

# 常用模型

## 生产者消费者

### 基础知识

1. 概念简述：生产者->缓冲区->消费者

   ![image-20210809172614456](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210809172614456.png)

2. 缓冲区：

   1. 解耦（降低生产者 和 消费者之间 耦合度）
   2. 处理并发（生产者、消费者 数量不对等时，能保持正常通信）
   3. 缓存 （生产者 消费者 数据处理速度不一致时，暂存数据）

### 实现

1. 使用channel作为缓冲区
   - 有缓冲：异步通信
   - 无缓冲：同步通信
2. 

