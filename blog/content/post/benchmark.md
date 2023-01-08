---
title: benchmark
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["golang"]
tags: ["golang"]
---

# benchmark
#技术/golang

# 介绍
基准测试是测量一个程序在固定工作负载下的性能

在Golang中，基准测试函数以Benchmark为前缀并且带有一个 *testing.B 类型的参数

# 规则

1. 基准测试的代码文件必须以_test.go结尾
2. 基准测试的函数必须以Benchmark开头
3. 基准测试函数必须接受一个指向Benchmark类型的指针作为唯一参数，即比如func BenchmarkMapkeys1(b *testing.B)
4. 基准测试函数不能有返回值
5. b.ResetTimer是重置计时器，这样可以避免for循环之前的初始化代码的干扰
6. 最后的for循环很重要，被测试的代码要放到循环里
7. b.N是基准测试框架提供的，表示循环的次数，因为需要反复调用测试的代码，才可以评估性能


# 注意事项
## ResetTimer

If a benchmark needs some expensive setup before running, the timer may be reset

如果在整个 benchmark 执行前，需要一些耗时的准备工作，我们需要将这部分耗时忽略掉
```go
func BenchmarkFib(b *testing.B) {
	time.Sleep(3 * time.Second) // 模拟耗时的准备工作
	b.ResetTimer() // 重置计时器，忽略前面的准备时间
	for n := 0; n < b.N; n++ {
		fib(10)
	}
}
```

## StopTimer & StartTimer

StopTimer stops timing a test. This can be used to pause the timer while performing complex initialization that you don't want to measure.

StartTimer starts timing a test. This function is called automatically before a benchmark starts, but it can also be used to resume timing after a call to StopTimer.

如果在被测函数每次执行前，需要一些准备工作，我们可以使用 StopTimer 暂停计时，准备工作完成后，使用 StartTimer 继续计时。

```go
func BenchmarkFib(b *testing.B) {
	for n := 0; n < b.N; n++ {
		b.StopTimer()  // 暂停计时
		prepare()      // 每次函数执行前的准备工作
		b.StartTimer() // 继续计时

		funcUnderTest() // 被测函数
	}
}
```

# 参数介绍
`go test -bench=. -benchmem -count=3`

`go test adapter.go circuitbreaker.go  adapter_benchmark_test.go -bench=BenchmarkFuse -benchmem -count=10 -cpuprofile=cpu.out`

1. 参数-bench，它指明要测试的函数；点字符意思是测试当前所有以Benchmark为前缀函数

2. 参数-benchmem，性能测试的时候显示测试函数的内存分配大小，内存分配次数的统计信息

3. 参数-count n,运行测试和性能多少次，默认一次

## 更多参数
|参数|含义|
|-|-|
|-bench regexp|性能测试，支持表达式对测试函数进行筛选。|
|-bench|. 则是对所有的benchmark函数测试，指定名称则只执行具体测试方法而不是全部|
|-benchmem|性能测试的时候显示测试函数的内存分配的统计信息|
|－count n|运行测试和性能多少次，默认一次|
|-run regexp|只运行特定的测试函数， 比如-run ABC只测试函数名中包含ABC的测试函数|
|-timeout t|测试时间如果超过t, panic,默认10分钟|
|-v|显示测试的详细信息，也会把Log、Logf方法的日志显示出来|


# 测试结果分析

```sh
goos: darwin
goarch: amd64
pkg: godemo/circuit_breaker
cpu: Intel(R) Core(TM) i7-1068NG7 CPU @ 2.30GHz
BenchmarkIsAllowed-8   	38678893	        30.17 ns/op	       0 B/op	       0 allocs/op
BenchmarkIsAllowed-8   	41858949	        30.09 ns/op	       0 B/op	       0 allocs/op
BenchmarkIsAllowed-8   	41888743	        28.25 ns/op	       0 B/op	       0 allocs/op
BenchmarkSucceed-8     	18539643	        64.84 ns/op	       0 B/op	       0 allocs/op
BenchmarkSucceed-8     	18441639	        64.92 ns/op	       0 B/op	       0 allocs/op
BenchmarkSucceed-8     	18286372	        65.42 ns/op	       0 B/op	       0 allocs/op
BenchmarkFail-8        	 5031613	       247.0 ns/op	       0 B/op	       0 allocs/op
BenchmarkFail-8        	 4884055	       243.7 ns/op	       0 B/op	       0 allocs/op
BenchmarkFail-8        	 4904302	       243.1 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	godemo/circuit_breaker	11.878s
```
以测试结果的第一行为例

1. BenchmarkIsAllowed-8 表测试的函数名，-8 表示GOMAXPROCS（线程数）的值为8

2. 38678893 表示一共执行了38678893次，即B.N的值

3. 30.17 ns/op表示平均每次操作花费了30.17纳秒

4. 0 B/op 表示每次操作申请了0Byte的内存申请

5. 0 allocs/op 表每次操作申请了0次内存

# 结合pprof分析

|参数含义|命令示例|
|-|-|
|
-cpuprofile [file]输出cpu性能文件|go test -bench=. -benchmem -cpuprofile=cpu.out-memprofile|
|[file]输出mem内存性能文件go test -bench=. -benchmem -memprofile=cpu.out
生成的CPU、内存文件|

可以通过go tool pprof [file]进行查看，然后在pprof中通过list [file]方法查看CPU、内存的耗时情况

### 内存情况
```
(pprof) list BenchmarkArrayAppend
Total: 36.49GB
ROUTINE ======================== program/benchmark.BenchmarkArrayAppend in /Users/guanjian/workspace/go/program/benchmark/benchmark_test.go
   11.98GB    11.98GB (flat, cum) 32.83% of Total
         .          .      7://Array
         .          .      8:func BenchmarkArrayAppend(b *testing.B) {
         .          .      9:   for i := 0; i < b.N; i++ {
         .          .     10:           var arr []int
         .          .     11:           for i := 0; i < 10000; i++ {
   11.98GB    11.98GB     12:                   arr = append(arr, i)
         .          .     13:           }
         .          .     14:   }
         .          .     15:}
```
### CPU情况
```
(pprof) list BenchmarkArrayAppend
Total: 8.86s
ROUTINE ======================== program/benchmark.BenchmarkArrayAppend in /Users/guanjian/workspace/go/program/benchmark/benchmark_test.go
      10ms      640ms (flat, cum)  7.22% of Total
         .          .      6:
         .          .      7://Array
         .          .      8:func BenchmarkArrayAppend(b *testing.B) {
         .          .      9:   for i := 0; i < b.N; i++ {
         .          .     10:           var arr []int
      10ms       10ms     11:           for i := 0; i < 10000; i++ {
         .      630ms     12:                   arr = append(arr, i)
         .          .     13:           }
         .          .     14:   }
         .          .     15:}

```