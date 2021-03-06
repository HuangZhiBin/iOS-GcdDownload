# GCD案例分析
以Swift4实现GCD的9种经典案例
- 1.&nbsp;串行队列与并行队列
- 2.&nbsp;main队列与global队列
- 3.&nbsp;asyncAfter
- 4.&nbsp;DispatchGroup
- 5.&nbsp;barrier
- 6.&nbsp;sync与死锁
- 7.&nbsp;concurrentPerform
- 8.&nbsp;suspend与resume
- 9.&nbsp;Semaphore

### 1.&nbsp;串行队列与并行队列
#### DispatchQueueViewController
- **串行队列**(Serial Dispatch Queue)：同时只能执行一个追加的任务(Block)
- **并行队列**(Concurrent Dispatch Queue)：同时执行多个追加的任务(Block)
- 可自行创建串行队列和并行队列

#### （1）串行队列
代码：创建串行队列，执行追加的任务，分析执行顺序
```swift
        //队列名称的前缀推荐使用应用程序ID,不指定第二个DispatchQueue()参数,创建的queue将默认为serialQueue
        let serialQueue = DispatchQueue(label: "com.dianbo.serialQueue");
        
        serialQueue.async {
            self.log("串行队列中同步执行的第1个任务", Thread.current);
            sleep(4)
        }
        
        serialQueue.async {
            self.log("串行队列中同步执行的第2个任务", Thread.current);
            sleep(2)
        }
        
        serialQueue.async {
            self.log("串行队列中同步执行的第3个任务", Thread.current);
        }
```
- 1.&nbsp;串行队列同时只能执行一个追加处理，先追加的任务会先执行
- 2.&nbsp;执行结果
```swift
串行队列中同步执行的第1个任务: 当前线程的hash为105553118839552
串行队列中同步执行的第2个任务: 当前线程的hash为105553118839552
串行队列中同步执行的第3个任务: 当前线程的hash为105553118839552
```
- 3.&nbsp;可以看到执行顺序一定是1->2->3，并且为同一个线程
#### （2）并行队列
代码：创建并行队列，执行追加的任务，分析执行顺序
```swift
        let concurrentQueue = DispatchQueue(label: "com.dianbo.concurrentQueue", attributes: .concurrent)
        
        concurrentQueue.async {
            self.log("并行队列中同步执行的第1个任务", Thread.current);
            sleep(4);
        }
        
        concurrentQueue.async {
            self.log("并行队列中同步执行的第2个任务", Thread.current);
            sleep(2)
        }
        
        concurrentQueue.async {
            self.log("并行队列中同步执行的第3个任务", Thread.current);
        }
```
- 1.&nbsp;并行队列同时执行多个追加的任务
- 2.&nbsp;执行结果
```swift
并行队列中同步执行的第2个任务: 当前线程的hash为106102874632768
并行队列中同步执行的第1个任务: 当前线程的hash为105553118840576
并行队列中同步执行的第3个任务: 当前线程的hash为105553116786816
```
- 3.&nbsp;可以看到执行顺序不定(多试几次),可能是1->2->3，也可能是2->1->3，或者是其他的情况。虽然追加的任务顺序是1->2->3，但在并行队列中，追加的所有任务执行顺序不定。并且不同任务在执行时，所在的线程不为同一个
> `追加`和`执行`是两个不同的概念，在并行队列中，先追加的任务不代表先执行

### 2.&nbsp;main队列和global队列
#### DispatchGlobalViewController
- **main队列**和**global队列**是系统标准提供的队列，即全局队列
- **main队列**(Main Dispatch Queue)是在主线程RunLoop中执行的队列，属于串行队列
- **global队列**(Global Dispatch Queue)是所有应用程序都能使用的并行队列，在swift中有6个执行优先级
    - 1.**userInteractive**&nbsp;(优先级最高)
    - 2.**userInitiated**(优先级第2高)
    - 3.**default**(优先级第3高)
    - 4.**utility**(优先级第4高)
    - 5.**background**(优先级最低)
    - 6.**unspecified**&nbsp;(未指定优先级)

> 自行创建串行队列和并行队列的优先级与global队列的default优先级相同

代码： global队列指定优先级，分析任务的执行顺序

```swift
        DispatchQueue.global(qos: .background).async {
            self.log("background任务", Thread.current);
            sleep(4);
        }
        
        DispatchQueue.global(qos: .utility).async {
            self.log("utility任务", Thread.current);
            sleep(2)
        }
        
        DispatchQueue.global(qos: .default).async {
            self.log("default任务", Thread.current);
        }
        
        DispatchQueue.global(qos: .userInitiated).async {
            self.log("userInitiated任务", Thread.current);
        }
        
        DispatchQueue.global(qos: .userInteractive).async {
            self.log("userInteractive任务", Thread.current);
        }
        
        DispatchQueue.global(qos: .unspecified).async {
            self.log("unspecified任务", Thread.current);
        }
```

- 1.&nbsp;执行结果
```swift
userInteractive任务: 当前线程的hash为106102874588992
default任务: 当前线程的hash为106102874590080
userInitiated任务: 当前线程的hash为106102874590080
unspecified任务: 当前线程的hash为106102874582784
utility任务: 当前线程的hash为105553118758848
background任务: 当前线程的hash为106102874588096
```
- 2.&nbsp;global队列虽能指定任务执行的优先级，但不能保证实时性，因此执行顺序也只能是大致的判断。上面的代码执行的结果不一定是userInteractive->userInitiated->default->utility->background


### 3.&nbsp;asyncAfter
#### DispatchAfterViewController
- **asyncAfter**能完成在指定时间后执行某些处理
- **asyncAfter**在指定时间后追加处理到指定的queue，而不是在指定时间后执行处理
    - 例如下面代码的执行时间实际上会大于3秒
```swift
        DispatchQueue.main.asyncAfter(deadline: DispatchTime.now() + 3.0) {
            self.log("这是3秒后执行的语句", Thread.current);
        }
```
- **DispatchTime**和**DispatchWallTime**的区别
    - DispatchTime为毫秒级，通常用于计算相对时间，例如3秒后
    - DispatchWallTime为微毫秒级，通常用于计算绝对时间，例如2019年1月1日 00:00:000
    - DispatchWallTime的精确度明显高于DispatchTime
        

```swift
        //3秒后添加到main线程，而不是3秒后执行，实际执行时间最少3秒
        DispatchQueue.main.asyncAfter(deadline: DispatchTime.now() + 3.0) {
            // your code here
            self.log("这是3秒后执行的语句", Thread.current);
        }
        
        //比上面精确的3秒时间
        DispatchQueue.main.asyncAfter(wallDeadline: DispatchWallTime.now() + 2.0, execute: {
            // your code here
            self.log("精确的3秒后执行的语句", Thread.current);
        });
```

- 1.&nbsp;执行结果
```swift
精确的3秒后执行的语句: 当前线程的hash为105553116718016
这是3秒后执行的语句: 当前线程的hash为105553116718016
```
- 2.&nbsp;相同时间间隔内指定DispatchWallTime的任务先于DispatchTime执行

### 4.&nbsp;DispatchGroup
#### DispatchGroupViewController
- **DispatchGroup**用于检查队列的所有任务是否执行结束
```swift
        let concurrentQueue = DispatchQueue(label: "com.dianbo.concurrentQueue", attributes: .concurrent)
        
        let group = DispatchGroup.init();
        
        concurrentQueue.async(group: group) {
            self.log("并行队列中同步执行的第1个任务", Thread.current);
            sleep(4);
        }
        
        concurrentQueue.async(group: group) {
            self.log("并行队列中同步执行的第2个任务", Thread.current);
            sleep(2)
        }
        
        concurrentQueue.async(group: group) {
            self.log("并行队列中同步执行的第3个任务", Thread.current);
        }
        
        group.notify(queue: DispatchQueue.main) {
            self.log("done doing all stuff", Thread.current);
        }
```
执行结果：
```swift
并行队列中同步执行的第1个任务: 当前线程的hash为106102874590080
并行队列中同步执行的第2个任务: 当前线程的hash为106102874590464
并行队列中同步执行的第3个任务: 当前线程的hash为105553118757888
done doing all stuff: 当前线程的hash为105553116718016
```
- DispatchGroup的**wait**也可以检查队列的所有任务是否执行结束，但是会导致当前线程停止，直到所有任务执行结束
```swift
        let concurrentQueue = DispatchQueue(label: "com.dianbo.concurrentQueue", attributes: .concurrent)
        
        let group = DispatchGroup.init();
        
        concurrentQueue.async(group: group) {
            self.log("并行队列中同步执行的第1个任务", Thread.current);
            sleep(4);
        }
        
        concurrentQueue.async(group: group) {
            self.log("并行队列中同步执行的第2个任务", Thread.current);
            sleep(2)
        }
        
        concurrentQueue.async(group: group) {
            self.log("并行队列中同步执行的第3个任务", Thread.current);
        }
        
        group.wait();//同步一直等待,当前线程停止
        self.log("done doing all stuffs", Thread.current);
```    
执行结果：
```swift
并行队列中同步执行的第1个任务: 当前线程的hash为106102874590080
并行队列中同步执行的第2个任务: 当前线程的hash为106102874590464
并行队列中同步执行的第3个任务: 当前线程的hash为105553118757888
done doing all stuff: 当前线程的hash为105553116718016
``` 
- 检查1秒后队列是否执行结束
```swift
        let concurrentQueue = DispatchQueue(label: "com.dianbo.concurrentQueue", attributes: .concurrent)
        
        let group = DispatchGroup.init();
        
        concurrentQueue.async(group: group) {
            self.log("并行队列中同步执行的第1个任务", Thread.current);
            sleep(4);
        }
        
        concurrentQueue.async(group: group) {
            self.log("并行队列中同步执行的第2个任务", Thread.current);
            sleep(2)
        }
        
        concurrentQueue.async(group: group) {
            self.log("并行队列中同步执行的第3个任务", Thread.current);
        }
        
        let result = group.wait(timeout: DispatchTime.now() + 1.00);//同步一直等待
        if(result == .success){
            self.log("done doing all stuffs after 1 second", Thread.current);
        }
        else{
            self.log("still doing stuffs after 1 second", Thread.current);
        }
```   
执行结果：
```swift
并行队列中同步执行的第1个任务: 当前线程的hash为106102874590080
并行队列中同步执行的第2个任务: 当前线程的hash为106102874590464
并行队列中同步执行的第3个任务: 当前线程的hash为105553118757888
still doing stuffs after 1 second: 当前线程的hash为105553116718016
```

### 5.&nbsp;barrier
#### DispatchBarrierViewController
- 并行处理数据时，导致数据不一致的情况
```swift
        let queue = DispatchQueue.global(qos: .default);
        var a = "12345";
        queue.async {
            self.log("> 1 任务开始执行", Thread.current)
            self.log("  < 1.a的值是" + a, Thread.current)
        }
        queue.async {
            self.log("> 2 任务开始执行", Thread.current)
            self.log("  < 2.a的值是" + a, Thread.current)
        }
        queue.async {
            self.log("> 3 任务开始执行", Thread.current)
            self.log("  < 3.a的值是" + a, Thread.current)
        }
        queue.async {
            a = "54321";
            self.log("======a的值修改为" + a + "======", Thread.current)
        }
        queue.async {
            self.log("> 4 任务开始执行", Thread.current)
            self.log("  < 4.a的值是" + a, Thread.current)
        }
        queue.async {
            self.log("> 5 任务开始执行", Thread.current)
            self.log("  < 5.a的值是" + a, Thread.current)
        }
        queue.async {
            self.log("> 6 任务开始执行", Thread.current)
            self.log("  < 6.a的值是" + a, Thread.current)
        }
```
执行结果：
```swift
> 1 任务开始执行: 当前线程的hash为105553116765696
  < 1.a的值是12345: 当前线程的hash为105553116765696
> 2 任务开始执行: 当前线程的hash为106102872585920
> 3 任务开始执行: 当前线程的hash为106102872577088
======a的值修改为54321======: 当前线程的hash为105553116765696
> 4 任务开始执行: 当前线程的hash为105553116763072
  < 2.a的值是54321: 当前线程的hash为106102872585920
> 5 任务开始执行: 当前线程的hash为106102872587008
  < 3.a的值是54321: 当前线程的hash为106102872577088
> 6 任务开始执行: 当前线程的hash为106102872586624
  < 4.a的值是54321: 当前线程的hash为105553116763072
  < 5.a的值是54321: 当前线程的hash为106102872587008
  < 6.a的值是54321: 当前线程的hash为106102872586624
```
从执行结果可以看到，任务2开始后，在任务2结束之前，a的值已经修改为54321，导致任务2在访问变量a时返回的是已经被修改后的值，出现了数据不一致的情况。
- **barrier**用于解决并行处理数据时，导致数据不一致的情况
> barrier对global队列无效，只能用于自定义队列
```swift
        let queue = DispatchQueue(label: "com.dianbo.concurrentQueue", attributes: .concurrent)
        var a = "12345";
        queue.async {
            self.log("> 1 任务开始执行", Thread.current)
            self.log("  < 1.a的值是" + a, Thread.current)
        }
        queue.async {
            self.log("> 2 任务开始执行", Thread.current)
            self.log("  < 2.a的值是" + a, Thread.current)
        }
        queue.async {
            self.log("> 3 任务开始执行", Thread.current)
            self.log("  < 3.a的值是" + a, Thread.current)
        }
        queue.async(flags: .barrier){
            a = "54321";
            self.log("======a的值修改为" + a + "======", Thread.current)
        }
        queue.async {
            self.log("> 4 任务开始执行", Thread.current)
            self.log("  < 4.a的值是" + a, Thread.current)
        }
        queue.async {
            self.log("> 5 任务开始执行", Thread.current)
            self.log("  < 5.a的值是" + a, Thread.current)
        }
        queue.async {
            self.log("> 6 任务开始执行", Thread.current)
            self.log("  < 6.a的值是" + a, Thread.current)
        }
```
执行结果：
```swift
> 1 任务开始执行: 当前线程的hash为106102872585920
> 3 任务开始执行: 当前线程的hash为106102872582400
> 2 任务开始执行: 当前线程的hash为106102872586752
  < 1.a的值是12345: 当前线程的hash为106102872585920
  < 3.a的值是12345: 当前线程的hash为106102872582400
  < 2.a的值是12345: 当前线程的hash为106102872586752
======a的值修改为54321======: 当前线程的hash为106102872586752
> 5 任务开始执行: 当前线程的hash为105553116762112
> 4 任务开始执行: 当前线程的hash为106102872586752
> 6 任务开始执行: 当前线程的hash为106102872582400
  < 5.a的值是54321: 当前线程的hash为105553116762112
  < 4.a的值是54321: 当前线程的hash为106102872586752
  < 6.a的值是54321: 当前线程的hash为106102872582400
```
barrier追加到队列后，会等待并行执行的任务全部处理完成，然后才会处理barrier的任务。此时同时间只能执行barrier的任务，barrier的任务处理结束后，才会执行后面的并行执行的任务。保证了在数据修改的操作时，并行操作只有一个。

### 6.&nbsp;sync与死锁
#### DispatchSyncViewController
- GCD的**sync**将指定的block**同步**追加到指定的队列
```swift
        let globalQueue = DispatchQueue.global(qos: .default);
        globalQueue.async {
            self.log("globalQueue.async", Thread.current);
        }
        globalQueue.sync {
            self.log("globalQueue.sync", Thread.current);
        }
```
执行结果：(建议多次测试)
```swift
globalQueue.sync: 当前线程的hash为105553116670464
globalQueue.async: 当前线程的hash为106102872585920
```
或者
```swift
globalQueue.async: 当前线程的hash为105553116670464
globalQueue.sync: 当前线程的hash为106102872585920
```
从执行结果可以看到，顺序是sync->async,或者是async->sync,虽然async的追加顺序先于sync，但其执行顺序不定，可能先于sync也可能晚于sync。
- **sync同步操作引发“死锁”的情况**
```swift
        let mainQueue = DispatchQueue.main
        mainQueue.sync {
            self.log("main.sync", Thread.current);
        }
```
从执行结果可以看到，程序发生死锁而发生异常。原因在于：main队列在等待sync追加block操作的结束，但是main队列正在执行这些代码，因此sync的block始终无法追加到main队列。
> main队列属于串行队列，自定义的串行队列追加sync同步操作同样会引发“死锁”

sync同步操作即使发生在async中，也会因为程序无法继续执行，出现“死锁”：
```swift
        let mainQueue = DispatchQueue.main
        mainQueue.async {
            mainQueue.sync {
                self.log("main.sync", Thread.current);
            }
        }
```
> 避免“死锁”的原则：绝不把sync同步操作追加到串行队列中

### 7.&nbsp;concurrentPerform
#### DispatchApplyViewController
- concurrentPerform(即Objective-C的dispatch_apply函数)按指定次数的block追加到队列中，和sync操作相同，并等待全部执行结束
```swift
        let concurrentQueue = DispatchQueue(label: "com.dianbo.concurrentQueue", attributes: .concurrent)
        concurrentQueue.async {
            DispatchQueue.concurrentPerform(iterations: 10) {//执行10次
                self.log("\($0)", Thread.current);
                sleep(1);
            }
            self.log("concurrentQueue is all work done", Thread.current);
        }
```
执行结果：
```swift
2: 当前线程的hash为105553118785216
1: 当前线程的hash为105553118770048
0: 当前线程的hash为105553118770624
3: 当前线程的hash为105553118785536
5: 当前线程的hash为105553118770624
4: 当前线程的hash为105553118785216
6: 当前线程的hash为105553118770048
7: 当前线程的hash为105553118785536
10: 当前线程的hash为105553118770048
9: 当前线程的hash为105553118785216
11: 当前线程的hash为105553118785536
8: 当前线程的hash为105553118770624
14: 当前线程的hash为105553118785216
13: 当前线程的hash为105553118785536
12: 当前线程的hash为105553118770048
concurrentQueue is all work done: 当前线程的hash为105553118770624
```
结果分析
- 1.&nbsp;从执行结果可以看到，10次追加到串行队列的block的执行顺序不定的，但执行结束后一定会输出`concurrentQueue is all work done`
- 2.&nbsp;concurrentPerform会一直等待全部执行结束，因此建议在async中处理

### 8.&nbsp;suspend与resume
#### DispatchSuspendViewController
- suspend挂起指定的队列
```swift
queue.suspend()
```
- resume恢复指定的队列
```swift
queue.resume()
```
- suspend对队列已经追加的处理不受影响，挂起后追加到队列的处理会停止执行。直至resume后，这些操作才可以继续执行。
```swift
        for index in 0...5{
            concurrentQueue.async {
                self.log("\(index)", Thread.current);
                sleep(2)
            }
        }

        concurrentQueue.suspend()
        for index in 6...10{
            concurrentQueue.async {
                self.log("\(index)", Thread.current);
                sleep(2)
            }
        }
```
执行结果：
```swift
2: 当前线程的hash为105553118761216
0: 当前线程的hash为105553118759744
3: 当前线程的hash为106102874587968
1: 当前线程的hash为105553118760896
4: 当前线程的hash为106102874588544
5: 当前线程的hash为106102874587136
```
结果分析
- 1.&nbsp;suspend后追加的第6-10个处理不会执行，因为队列已经挂起，但之前的追加的第1-5个处理不受影响，正常执行
- 2.此时尝试执行下面的代码，队列通过resume()恢复了，继续执行第6-10个处理
```swift
concurrentQueue.resume()
```
执行结果：
```swift
6: 当前线程的hash为106102874588672
7: 当前线程的hash为105553118764608
8: 当前线程的hash为106102874588992
9: 当前线程的hash为106102874588224
10: 当前线程的hash为105553118758528
```

### 9.&nbsp;Semaphore
#### DispatchSemaphoreViewController
- Semaphore指旗语，表示一种计数的信号。当Semaphore的值大于0时，允许执行下面的代码。等于0则继续等待，直至Semaphore的值大于0时才“放行”。
执行代码：
```swift
        let semaphore = DispatchSemaphore(value: 1) //设置semaphore的初始量为1
        let queue = DispatchQueue(label: "com.leo.concurrentQueue", qos: .default, attributes: .concurrent)
        
        queue.async {
            //若semaphore>0继续执行，等于0则继续等待
            semaphore.wait()
            
            //此时，semaphore的值减1
            self.log("Start usb task1",Thread.current)
            sleep(3)
            self.log("End usb task1",Thread.current)
            
            semaphore.signal()
            //此时，semaphore的值加1
        }
        
        queue.async {
            semaphore.wait()
            
            self.log("Start usb task2",Thread.current)
            sleep(3)
            self.log("End usb task2",Thread.current)
            
            semaphore.signal()
        }
        
        queue.async {
            semaphore.wait()
            
            self.log("Start usb task3",Thread.current)
            sleep(3)
            self.log("End usb task3",Thread.current)
            
            semaphore.signal()
        }
```
执行结果：
```swift
Start task1: 当前线程的hash为106102874588672
End task1: 当前线程的hash为106102874588672
Start task3: 当前线程的hash为106102874573120
End task3: 当前线程的hash为106102874573120
Start task2: 当前线程的hash为105553118763456
End task2: 当前线程的hash为105553118763456
```
结果分析
- 1.&nbsp;semaphore的初始量为1，1个追加到队列的任务在semaphore.wait()时，检测到semaphore值大于0，放开执行，semaphore值减1(此时semaphore=0)。这时其他追加到队列的任务在semaphore.wait()时，由于semaphore值为0，需要继续等待，直到上面“放行”的任务执行完毕后，通过semaphore.signal()重新把semaphore值加1(此时semaphore=1)，让其他队列有机会继续执行
- 2.将semaphore的初始量设置为3,可以让同时放行的操作的数量变为3个，以此类推
- 3.semaphore能避免一次性执行多个async操作(例如10000个操作)时导致内存错误的问题

| Item      | Value |
| --------- | -----:|
| 作者  | **黄智彬** |
| 原创  | **YES** |
| 微信  | **ikrboy** |
| 邮箱  |   **ikrboy@163.com** |