---
title: go-primer-note
date: 2019-06-16 17:20:49
tags: Go
---

# Go基础笔记摘要

本文只是介绍在基础阶段，Go语言中一些需要留意的特性。

1. 无符号整数一般用于特殊的位处理，二进制文件处理等，一般用的不多；

2. 浮点数优先选择float64, 32的误差可能比较大；

3. NaN 之间用 == ，总是不成立；
  <!--more-->

4. 考虑到编码，字符串的第i个字节，不一定是第i个字符。[i]取的是第i个字符；

5. 子串操作共用底层数据，因此开销低廉；

6. 字符串和[]byte之间可以互相转换，会有拷贝行为；

7. 无类型int转的时候，大小不确定，而float那些都是确定的；

8. go中的异常一般用作bug等情况下，不像Java那样，用异常控制业务流程，因为Go的异常会陷入一种带有错误信息控制流，而打出不好理解的堆栈信息，这个和Java的堆栈不同。	

9. Go中的错误信息可能会被频繁的串起来，所以要尽量避免换行，这样可以方便的grep到；

10. Go 的recover尽在defer函数中调用才有效；

11. 函数接受者可以是一切非指针类型或接口类型，但如下可以用的:

    ```go
		type Point struct {X,Y float64}
		func (i *Point) IsOk() {
		}
    ```
	这种就不行，因为接受者不能是指针类型
	```go
		type P *Point
		func (p P) IsNotOk{}
	```
    
12. defer实参在入栈的时候估值，gorountine实参在创建的时候估值。但是如果后面是匿名函数，则实参的估值推迟到执行的时候，例如：

	```go
		func estimateValSample(t *testing.T) {
			defer fmt.Print(x) // 1
			defer func() {
				fmt.Print(x) // 10
			}()
			x = 10
		}
	```
输出结果： 10，1

13. 对于多个int64连接成字符串，又不是在循环体内，循环很多次组成成一个字符串的，我建议直接用 + 号就行了。

```
// 直接+拼接字符串： Benchmark-8   	10000000	       205 ns/op
func plusStr() {
	var u1 int64 = rand.Int63()
	var u2 int64 = rand.Int63()
	_ = strconv.FormatInt(u1, 10) + "_" + strconv.FormatInt(u2, 10)
}

//通过Sprintf格式化字符串  Benchmark-8  10000000	       258 ns/op
func format(){
	var u1 int64 = rand.Int63()
	var u2 int64 = rand.Int63()
	_ = fmt.Sprintf("%d_%d", u1, u2)
}

// 通过Join构造数组，然后拼接，Benchmark-8  10000000	       211 ns/op
func join() {
	var u1 int64 = rand.Int63()
	var u2 int64 = rand.Int63()
	_ = strings.Join([]string{strconv.FormatInt(u1, 10), "_", strconv.FormatInt(u2, 10)}, "")
}
```
13. 待续
