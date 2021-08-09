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

### 死锁

1. 原因：
   - 单go程自己死锁
   - go程间channel访问顺序导致死锁
   - 多go程，多channel交叉死锁

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

