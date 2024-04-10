---
title: 'C语言定时任务库的实现（一）'
date: 2024-04-10 15:13:14
tags: [timer,C]
published: true
hideInList: false
feature: 
isTop: false
---
# 目的
在项目开发过程中，我们常常需要定时的执行一些任务，通常的做法我们会创建一个线程，然后在线程里执行一个定时任务，如果有N个定时任务，则需要启动N个定时任务。这种做法有以下几个弊端。
1、频繁的创建线程可能会影响系统的整体运作性能。
2、每实现一个定时任务，则需要考虑任务的创建，销毁，任务的执行，而且大部分的内容是冗余的。
3、多任务之间的同步问题。
为了解决以上问题，我实现了一个多实例单线程多任务的定时器库。

# 实现方案
## 数据结构
 ```
typedef struct {
    struct list_head       head;                    // 链表头
    mtimer_task_callback   run;                     // 任务回调
    unsigned int        id;                         // 定时任务id
    unsigned int        last_tick;                  // 上一次执行的时间
    unsigned int        timeout;                    // 定时的超时时间，ms
    mtimer_task_mode    mode;                       // 定时任务的工作模式，两种工作模式。
    void           *user_data1;                     // 定时任务的上下文1
    void           *user_data2;                     // 定时任务的上下文2
    unsigned int        remove;                     // 定时任务是否标记删除
} mtimer_task;                                      // 定时任务节点
 
typedef struct {
    struct list_head    list;                       // 任务列表
    pthread_mutex_t  lock;                          // 锁
    unsigned int            id;                     // 待分配的定时器任务id
    mtimer_get_current_time get_current_time;       // 获取ms值的接口
} mtimer_handle;                                    // 定时器实例
 ```

## 工作模式
### 定时器有两种工作模式
 ```
typedef enum {
    MTIMER_MODE_LOOP = 0,        // 循环任务，每隔一个timeout执行一次
    MTIMER_MODE_NOT_LOOP,        // 非循环任务，仅在timeout时执行一次
} mtimer_task_mode;
 ```
## 接口
 ```
/*****************************************************************************
 函 数 名  : mtimer_create
 功能描述  : 创建定时器库实例
 输入参数  : mtimer_get_current_time func  
 输出参数  : 无
 返 回 值  : void
            NULL    : 创建失败
            非NULL   : 创建成功
*****************************************************************************/
void *mtimer_create(mtimer_get_current_time func);
 
/*****************************************************************************
 函 数 名  : mtimer_destroy
 功能描述  : 销毁定时器库实例
 输入参数  : void *handle  
 输出参数  : 无
 返 回 值  : int
            0   成功
            <0  失败
*****************************************************************************/
int mtimer_destroy(void *handle);
 
/*****************************************************************************
 函 数 名  : mtimer_task_reg
 功能描述  : 注册定时器任务
 输入参数  : void *handle                 
             mtimer_task_callback run  任务回调
             unsigned int    timeout   超时时间
             mtimer_task_mode    mode  LOOP:循环, 每隔timeout触发一次; NOT_LOOP:仅运行一次
             void       *user_data1    回调参数1
             void       *user_data2    回调参数2
 输出参数  : 无
 返 回 值  : unsigned
            返回定时器id
*****************************************************************************/
unsigned int mtimer_task_reg(
    void *handle,
    mtimer_task_callback run,
    unsigned int    timeout,
    mtimer_task_mode    mode,
    void       *user_data1,
    void       *user_data2);
 
/*****************************************************************************
 函 数 名  : mtimer_task_unreg
 功能描述  : 反注册定时器任务
 输入参数  : void *handle          
             unsigned int task_id   定时任务id  
 输出参数  : 无
 返 回 值  : 0: 反注册成功
            <0: 反注册失败
*****************************************************************************/
int mtimer_task_unreg(void *handle, unsigned int task_id);
 
/*****************************************************************************
 函 数 名  : mtimer_schedule
 功能描述  : 定时任务调度
 输入参数  : void *handle    
             unsigned int n  最多可执行的定时任务数，防止线程阻塞太长时间
 输出参数  : 无
 返 回 值  : 0  执行正常
            <0  执行异常
*****************************************************************************/
int mtimer_schedule(void *handle, unsigned int n);
 ```
### 使用范例
 ```
int func1(void *data1, void *data2)
{
    log_error("[time:%u, func1], data1:%s, data2:%s\n", Clock_GetTimeMs(),
              (char *)data1, (char *)data2);
    return 0;
}
 
int func2(void *data1, void *data2)
{
    log_error("[time:%u, func2], data1:%s, data2:%s\n", Clock_GetTimeMs(),
              (char *)data1, (char *)data2);
    return 0;
}
 
void *mtimer_debug_loop(void *handle)
{
 
    unsigned id1, id2;
 
    log_error("tick: %u------------------------------------ 1\n\n",
              Clock_GetTimeMs());
    id1 = mtimer_task_reg(handle, func1, 500, MTIMER_MODE_LOOP, "1data1", "1data2");
    id2 = mtimer_task_reg(handle, func2, 1000, MTIMER_MODE_NOT_LOOP, "2data1",
                          "2data2");
    while (1) {
        mtimer_schedule(handle, 10);
        Clock_SleepMs(100);
    }
 
    mtimer_destroy(handle);
 
    return NULL;
}
 
int mtimer_debug()
{
    void *handle;
    if (IS_NULL(handle = mtimer_create(Clock_GetTimeMs))) {
        log_error("mtimer_create failed\n");
        return -1;
    }
 
    pthread_t tid;
    pthread_create(&tid, NULL, mtimer_debug_loop, (void *)handle);
    pthread_detach(tid);
 
    return 0;
}
 ```

# 完整代码
代码还没有整理好，后续整理完之后会有更新，迫切需要的可以私聊
