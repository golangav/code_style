
## python

```Python
# 时间复杂度：O(logn)
def search(array, val):
    length = len(array)
    left = 0
    right =  length - 1
    while left <= right:
        mid = (right + left) // 2 # 需整除2，获取下标
        value = array[mid]
        if value == val:
            return mid
        elif value < val: # 值在右边
            left = mid + 1
        else: # 值在左边
            right = mid - 1
    else:
        return None
li = list(range(1, 9))
print(search(li, 5))
# 结果 
# [1, 2, 3, 4, 5, 6, 7, 8]
# 4

def select_sort(array):
    for i in range(len(array) - 1):
        min_loc = i  # 无序区最小值下标
        for j in range(i, len(array)):
            if array[j] < array[min_loc]:
                min_loc = j
        array[i], array[min_loc] = array[min_loc], array[i]
    return array

select_sort([2,3,4,5,9,7,8])



def select_sort(array):
    for i in range(len(array) - 1):
        min_loc = i  # 无序区最小值下标
        for j in range(i, len(array)):
            if array[j] < array[min_loc]:
                min_loc = j
        array[i], array[min_loc] = array[min_loc], array[i]
    return array

select_sort([2,3,4,5,9,7,8])


```





## golang

```Golang
package main

import (
   "fmt"
   "time"
)

//--------------------- Task任务 ---------------------

// 定义一个任务类型 Task
type Task struct {
   f func() error     // 一个Task里面应该有一个具体业务，业务名称叫 f
}

// Task 需要一个执行的方法
func (t *Task) Execute()  {
   t.f()  // 调用任务中已经绑定好的业务方法
}

// 创建一个 Task 任务
func NewTask(arg_f func() error) *Task  {
   t := &Task{
      f: arg_f,
   }
   return t
}

//--------------------- 协程池 ---------------------

// 定义一个pool的协程池
type Pool struct {
   EntryChannel chan *Task // 对外的Task入口 管道
   JobsChannel chan *Task  // 对内的Task任务
   work_num int
}

// 创建 Pool 的函数
func NewPool(num int) *Pool {
   p := &Pool{
      EntryChannel:make(chan *Task),
      JobsChannel:make(chan *Task),
      work_num:num,
   }
   return p
}

// 协程池创建work，让work去工作
func (p *Pool) work(worker_id int)  {
   // 1. 永久从JobsChannel管道中取任务
   // 2. 渠道任务执行任务
   for task := range p.JobsChannel{
      task.Execute()
      fmt.Println("work_id",worker_id,"执行完一个任务")
   }
}

// 让协程池工作
func (p *Pool) run() {
   // 1. 根据 work_num 创建worker工作，每个worker 应该是一个goroutine
   for i:=0;i<=p.work_num;i++{
      go p.work(i)
   }

   // 2. 从EntryChannel取任务，取到任务 发送给 JobsChannel
   for task := range p.EntryChannel{
      p.JobsChannel <- task
   }
}

//--------------------- 主函数 ---------------------

func main()  {
   // 1. 创建一些任务,打印系统时间（之后任务的逻辑就写在这个函数里）
   t := NewTask(func() error {
      fmt.Println(time.Now())
      return nil
   })

   // 2. 创建一个 Pool协程池，最大的worker数量为4
   p := NewPool(4)

   // 3.将任务交给协程池 Pool处理,不断的向p中写入任务t
   go func() {
      for {
         p.EntryChannel <- t

      }
   }()

   // 启动pool，让Pool开始工作，此时Pool会创建worker，让worker工作
   p.run()

}

```









