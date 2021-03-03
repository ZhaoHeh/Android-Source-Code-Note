# Linux 进程管理相关

## wait_event_interruptible

wait_event_interruptible实际上是一个宏定义，等效代码如下，等效的时候需要注意，condition不是值传递的参数，而是一个具体的表达式：  

```c++
wait_event_interruptible(wait_queue_head_t wq, bool condition) /*等效没有考虑返回值*/
{
    if (!(condition))
    {
        wait_queue_t _ _wait;
        init_waitqueue_entry(&_ _wait, current);
        add_wait_queue(&wq, &_ _wait);
        for (;;)
        {
            set_current_state(TASK_INTERRUPTIBLE);
            if (condition)
            break;
            schedule();  /* implicit call: del_from_runqueue(current)*/
        }
        current->state = TASK_RUNNING;
        remove_wait_queue(&wq, &_ _wait);
    }
}
```

wait_queue_head_t是一个等待队列的头，对应一个循环链表：  

```c++
    static DECLARE_WAIT_QUEUE_HEAD(binder_user_error_wait);
    static int binder_stop_on_user_error;

    static int binder_set_stop_on_user_error(const char *val,
                        const struct kernel_param *kp)
    {
        int ret;

        ret = param_set_int(val, kp);
        if (binder_stop_on_user_error < 2)
            wake_up(&binder_user_error_wait);
        return ret;
    }
    module_param_call(stop_on_user_error, binder_set_stop_on_user_error,
        param_get_int, &binder_stop_on_user_error, 0644);
// 代码路径：/drivers/android/binder.c

#define DECLARE_WAIT_QUEUE_HEAD(name) \ 
wait_queue_head_t name = __WAIT_QUEUE_HEAD_INITIALIZER(name)

#define __WAIT_QUEUE_HEAD_INITIALIZER(name) { \
　　.lock                = __SPIN_LOCK_UNLOCKED(name.lock), \
　　.task_list        = { &(name).task_list, &(name).task_list } }

struct __wait_queue_head {
　　spinlock_t  lock;          //自旋锁变量，用于在对等待队列头          
　　struct list_head task_list;  // 指向等待队列的list_head
}; 
typedef struct __wait_queue_head  wait_queue_head_t;
```

参考链接：[《等待队列（一）》](https://www.cnblogs.com/zhuyp1015/archive/2012/06/09/2542882.html)  

schedule则是真正的进程调度函数，是进程平行宇宙打开的钥匙：  

参考链接：[《进程调度函数schedule()解读》](http://fuzhii.com/2014/06/29/schedule/)  