---
title: Golang goroutine 使用
author: 刘Sir
date: 2024-04-17 10:10:00 +0800
categories: [技术]
tags: [Golang]
render_with_liquid: false
---  
纸上得来终觉浅，绝知此事要躬行  
最近在使用 golang 重写基于数据库的任务调度程序，为什么选择基于数据库的调度呢？
1. 数据库中 for update skip lock 可支持抢占式的任务调度，可避免重复调度。调度程序无状态，方便多实例情况下，无需第三方介入（主从模式）
2. 调度任务可通过 API 进行 CRUD 操作
3. 指标，日志，通知等监控信息都可以定制化

一个间隔 fetchIntervalSeconds 从数据库捞取任务的协程，schedule.ctx.Done() 是退出信号。
```
func (schedule dbScheduler) fetchSchedule() {
	ticker := time.NewTicker(schedule.fetchIntervalSeconds)
	defer ticker.Stop()
loop:
	for {
		select {
		case <-schedule.ctx.Done():
			// 收到退出信号，优雅关闭
			logger.Infof("fetch due goroutines: %s", schedule.ctx.Err())
			//return schedule.ctx.Err()
			break loop
		case <-ticker.C:
			st, _ := schedule.dbRepository.FetchAndLockPendingTasks()
			logger.Debugf("fetch due schedule, size:%d", len(st))
			for _, s := range st {
				go executeTask(schedule.dbRepository, s)
			}
		}
	}
}
```
需使用 ctx, cancel := context.WithCancel(context.Background()) 进行初始化，然后把 ctx 传递到需要的对象中，如此外面可通过 cancel() 通知退出。

下面是一个工作任务的协程，持续读取 taskQueue，然后执行 t.Run()。
```
func (e *Executor) worker(id int) {
	defer logger.Infof("Worker %d exited", id)
	for {
		select {
		case task, ok := <-e.taskQueue:
			if !ok {
				return
			}
			e.wg.Add(1)
			func(t scheduler.Task) {
				defer e.wg.Done()
				taskCtx, cancel := context.WithTimeout(e.ctx, e.timeout)
				defer cancel()
				t.Run()
				select {
				case <-taskCtx.Done():
					logger.Error("Task %s cancelled", t.TaskName)
					return
				}
			}(task)
		case <-e.ctx.Done():
			return
		}
	}
}
```
以上代码中配置了 taskCtx, cancel := context.WithTimeout(e.ctx, e.timeout) 一个超时的 ctx。e.wg.Add(1) 是 sync.WaitGroup 相当于一个计数器，作用在等待任务完成 （ e.wg.Wait() 会堵塞直到所有的都 Done ）。

另外一段代码 select 使用，是一个双方利用 select 和 chan 进行通信过程
```
func requestJobCtx(ctx context.Context, id uuid.UUID, ch chan jobOutRequest) *internalJob {
	resp := make(chan internalJob, 1)
    // 用 id 向调度器获取具体的任务信息
	select {
	case ch <- jobOutRequest{
		id:      id,
		outChan: resp,
	}:
	case <-ctx.Done():
		return nil
	}
    // 堵塞读取调度返回的任务信息
	var j internalJob
	select {
	case <-ctx.Done():
		return nil
	case jobReceived := <-resp:
		j = jobReceived
	}
	return &j
}
```
另外在使用 goroutine 的时候需要注意以下几点：
- goroutine 在收到 超时和退出信号后不会被系统直接杀死或者回收，正在执行的代码还是会继续执行。
```
// 此函数在 t.Run() 后才会收超时信号退出
func(t scheduler.Task) {
				defer e.wg.Done()
				taskCtx, cancel := context.WithTimeout(e.ctx, e.timeout)
				defer cancel()
				t.Run()
				select {
				case <-taskCtx.Done():
					logger.Error("Task %s cancelled", t.TaskName)
					return
				}
			}(task)
```
因此如果任务需要退出，需要在任务执行的代码的不同阶段 select 退出信号，如以下伪代码：
```
func runTask(ctx context.Context){
    step_1()
    select {
	case <-ctx.Done():
		return 
	default:

	}
    step_2()
    select {
	case <-ctx.Done():
		return 
	default:

	}
    step_3()
    select {
	case <-ctx.Done():
		return 
	default:

	}
    finished()
}
```
因此 golang 中的 goroutine 退出是协商式的，告诉代码你需要退出了，而不是直接让正在执行的代码结束。是由用户来决定在代码中**某时某处**代码可以结束执行了。

 goroutine 和 chan 使得 golang 在处理高并发的场景变得很容易。以下是一个简单示例：
 ```
 package main

import (
    "fmt"
    "sync"
)

func worker(id int, ch chan int, wg *sync.WaitGroup) {
    defer wg.Done()
    for task := range ch {
        fmt.Printf("Worker %d processing task %d\n", id, task)
    }
}

func main() {
    var wg sync.WaitGroup
    ch := make(chan int, 10) // 创建一个带缓冲区的通道

    // 启动多个 worker goroutines
    for i := 1; i <= 3; i++ {
        wg.Add(1)
        go worker(i, ch, &wg)
    }

    // 向通道发送任务
    for i := 1; i <= 5; i++ {
        ch <- i
    }

    close(ch) // 关闭通道，通知 worker 任务结束
    wg.Wait()  // 等待所有 worker 完成
}

 ```