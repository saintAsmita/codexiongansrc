---
title: go-timer-ticker
date: 2019-12-04 17:28:37
tags: golang timer
---

本文介绍golang定时器中timer 和 ticker的区别
<!--more-->

## 一个定时任务的实现
### timer 和 ticker的表现
newTicker：给定一个时间间隔d，则每个d就会向自己的chan中发送时间
newTimer: 给定一个时间间隔d，则至少在d时间后，会向自己的chan中发送一个当前时间，仅发送一次。如果想达到newTicker相似的效果，则可以每次触发后，调用Reset,重置计时时间，从重置那一刻开始，重新计算间隔d。
例如：

```java
func testTimerAndTicker(){
	var wg sync.WaitGroup
	wg.Add(2)

	// NewTimer()，3秒后发送一个时间到chan，仅发送一次
	timer := time.NewTimer(3 * time.Second)

	//NewTicker(), 每隔2秒发送一个时间
	ticker := time.NewTicker(3 * time.Second)

	go func(t *time.Ticker) {
		defer wg.Done()
		for {
			now := <-t.C
			fmt.Println("ticker", now.Format("2006-01-02 15:04:05"))
		}
	}(ticker)

	go func(t *time.Timer) {
		defer wg.Done()
		for {
			now := <-t.C
			fmt.Println("timer", now.Format("2006-01-02 15:04:05"))
			//t.Reset(3 * time.Second)
		}
	}(timer)

	wg.Wait()

	timer.Stop()
	ticker.Stop()
```
会每3秒输出一次ticker的信息，如果把//t.Reset(3 * time.Second)打开，则每3秒，会额外输出timer的信息

### ticker中函数运行时间大于tick间隔
例如如下代码：
```java
func Run() {
	ticker := time.NewTicker(time.Second)
	defer ticker.Stop()
	for {
		<-ticker.C
		time.Sleep(5 * time.Second)
		now := time.Now().Unix()
		fmt.Println("a", now)
	}
}
```
tick间隔是1秒，但是函数运行了5秒，则tick每次滴答时，发现 ticker.C还没有被读取，写不进去新的时间，就会丢弃最新的时间戳。所以每隔5秒输出一个a。