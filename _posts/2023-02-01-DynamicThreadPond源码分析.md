---
title: 01-DynamicThreadPond源码分析
tags: C++高性能并发库Hipe源码分析
key: test
---

# 类图
![image](https://github.com/oscar0325/oscar0325.github.io/raw/master/images/class.png)

# 框架图
![image](https://github.com/oscar0325/oscar0325.github.io/raw/master/images/arch.png)
- **算法思想**
	1. 核心思想，生成消费者模型
	2. 所有线程共享公共任务队列，通过公共队列的互斥锁来竞争获取任务
	3. 通过条件变量来阻塞线程，使得任务队列无任务时，所有工作线程可自动挂起，节约CPU资源
- **主要数据结构**
	- 公共任务队列，列表实现，可存储大量任务
	- 线程队列，map<thread id, thread>
# 源码及分析
- 线程安全
	- 通过互斥锁实现主线程添加或工作线程获取任务时，公共任务队列的独占
	- 通过条件变量来阻塞或唤醒线程：
		其中wait（unique_lock <mutex>＆lck，Predicate pred）函数使用了pred条件，既只有当 pred 条件为 false 时调用 wait() 才会阻塞当前线程，并且在收到其他线程的通知后只有当 pred 为 true 时才会被解除阻塞。防止错过通知或虚假唤醒。
	-具体使用，包括三个条件变量
		1. std::condition_variable awake_cv， 任务队列有任务时，唤醒线程处理任务，否则阻塞线程；
		2. std::condition_variable thread_cv，一个线程运行完毕后发出通知，实际用与所有线程运行完毕后，通知主线程进行下一步动作
		3. std::condition_variable task_done_cv，一个任务运行完毕后发出通知
- 源码
```C++
/* 文件名 dynamic_pond.h */

#pragma once
#include "header.h"

namespace hipe {

class DynamicThreadPond{
    bool stop = {false}; //动态线程池关闭标志

    std::atomic_int running_tnumb = {0}; //运行中的任务数
    std::atomic_int expect_tnumb = {0}; //期待运行的任务数
    bool is_waiting_for_task = {false}; //等待任务完成标志
    bool is_waiting_for_thread = {false}; //等待线程结束标志

    std::atomic_int total_tasks = {0}; //总任务数
    std::queue<HipeTask> shared_tq = {}; //公共任务队列
    std::mutex shared_locker; //任务队列及线程队列的互斥锁
    std::condition_variable awake_cv = {}; //唤醒线程的条件变量
    std::condition_variable task_done_cv = {}; //任务完成提示
    std::condition_variable thread_cv = {}; //线程运行结束提示
    std::map<std::thread::id, std::thread> pond; //线程队列
    std::queue<std::thread> dead_threads; //等待回收的线程池队列
    std::atomic_int shrink_numb = {0}; //线程池缩小数目
    std::atomic_int tasks_loaded = {0}; //等待运行的任务数

public:
    // 构造函数，添加指定线程数到工作线程队列
    explicit DynamicThreadPond(int tnumb = 0){
        addThreads(tnumb);
    }
    
    ~DynamicThreadPond(){
        if(!stop){
            close();
        }
    }

public:
    //关闭线程池
    void close() {
        stop = true;
        adjustThreads(0);
        waitForThreads(); // 等待所有线程停止运行
        joinDeadThreads();
    }

    void addThreads(int tnumb = 1){
        assert(tnumb >= 0);
        expect_tnumb += tnumb;
        HipeLockGuard lock(shared_locker); //添加线程时，锁住线程池
        while (tnumb--){
            std::thread t(&DynamicThreadPond::worker, this);
            //将添加的线程id和线程实例加入线程池，emplace通过make_pair转换pair为键值对
            pond.emplace(std::make_pair<std::thread::id, std::thread>(t.get_id(), std::move(t)));
        }
    }


    void delThreads(int tnumb = 1){
        assert((tnumb <= expect_tnumb) && (tnumb >= 0));
        expect_tnumb -= tnumb;
        shrink_numb += tnumb;
        HipeLockGuard lock(shared_locker); //删除线程时，阻塞所有线程
        awake_cv.notify_all(); 
    }

    void adjustThreads(int target_tnumb){
        assert(target_tnumb >= 0);
        //调整动态池数目
        if (target_tnumb > expect_tnumb){
            addThreads(target_tnumb - expect_tnumb);
            return;
        }
        if(target_tnumb < expect_tnumb) {
            delThreads(expect_tnumb - target_tnumb);
            return;
        }
    }

    //回收线程资源
    void joinDeadThreads() {
        std::thread t;
        while(true) {
            shared_locker.lock();
            if (!dead_threads.empty()) {
                t = std::move(dead_threads.front());
                dead_threads.pop();
                shared_locker.unlock();
            } else {
                shared_locker.unlock();
                break;
            }
            t.join();
        }
    }

    //获取剩余任务数
    int getTaskRemain() {
        return total_tasks.load();
    }

    //获取正在运行的任务数
    int getTaskLoaded() {
        return tasks_loaded.load();
    }

    //重置任务数
    int resetTaskLoaded() {
        return tasks_loaded.exchange(0);
    }

    //获取运行中的线程数
    int getRunningThreadNumb() const {
        return running_tnumb.load();
    }

    //获取期望的线程数
    int getExpectThreadNumb() const {
        return expect_tnumb.load();
    }

    //等待所有线程运行完毕
    void waitForThreads() {
        is_waiting_for_thread = true;
        HipeUniqGuard locker(shared_locker);
        thread_cv.wait(locker, [this] { return !total_tasks; }); //如果任务队列不为空，阻塞
        is_waiting_for_thread = false;
    }

    //等待所有任务运行完毕
    void waitForTasks() {
        is_waiting_for_task = true;
        HipeUniqGuard locker(shared_locker);
        task_done_cv.wait(locker, [this] { return !total_tasks; }); //如果任务队列不为空，阻塞
        is_waiting_for_task = false;
    }

    //往公共任务队列提交一个任务
    template <typename Runnable>
    void submit(Runnable&& foo) {
        {
            HipeLockGuard lock(shared_locker);
            shared_tq.emplace(std::forward<Runnable>foo);
            ++total_tasks;
        }
        awake_cv.notify_one();
    }

    //提交具有返回值的任务
    template <typename Runnable>
    auto submitForReturn(Runnable&& foo) -> std::future<typename std::result_of<Runnable()>::type> {
        using RT = typename std::result_of<Runnable()>::type;
        std::packaged_task<RT()> pack(std::forward<Runnable>(foo));
        std::future<RT> fut(pack.get_future());
        {
            HipeLockGuard lock(shared_locker);
            shared_tq.emplace(std::move(pack));
            ++total_tasks;
        }
        awake_cv.notify_one();
        return fut;
    }

    //往公共队列任务同时提交多个任务
    template <typename Container_>
    void submitInBatch(Container_& cont, size_t size) {
        {
            HipeLockGuard lock(shared_locker);
            total_tasks += static_cast<int>(size);
            for (size_t i = 0; i < size; ++i) {
                shared_tq.emplace(std::move(cont[i]));
            }
        }
        awake_cv.notify_all();
    }

private:
    void notifyThreadAdjust() {
        HipeLockGuard lock(shared_locker);
        thread_cv.notify_one();
    }

    void worker() {
        HipeTask task;

        running_tnumb++;
        if (is_waiting_for_thread) {
            notifyThreadAdjust();
        }
        do {
            HipeUniqGuard locker(shared_locker);
            awake_cv.wait(locker, [this] { return !shared_tq.empty() || shrink_numb > 0; });

            //缩容操作
            if (shrink_numb) {
                shrink_numb--;
                auto id = std::this_thread::get_id(); //获取当前线程的id
                dead_threads.emplace(std::move(pond[id])); //将线程加入到待回收线程队列，不直接调用join
                pond.erase(id);
                break;
            }

            task = std::move(shared_tq.front());
            shared_tq.pop();
            locker.unlock();

            tasks_loaded++;

            util::invoke(task); //执行任务
            --total_tasks;

            if (is_waiting_for_task) {
                HipeLockGuard lock(shared_locker);
                task_done_cv.notify_one();
            }
        } while(true);

        running_tnumb--;
        if (is_waiting_for_thread) {
            notifyThreadAdjust();
        }
    }
};

}
```
## 应用
- 加入一个管理线程，管理线程每秒查询一次，通过比较当前载入的任务数量与上一时刻载入的任务数量的大小来自动管理工作线程数量，如果当前载入的任务数量大于上一时刻载入的任务数量（认为线程工作忙碌），并且当前工作线程的数量没有超过设定的最大值，则自动添加工作线程。相反，则减少线程。
- demo代码
```C++
/**
 * How to adjust the dynamic thread pool.
 * This is just a reference.
 */
#include "../hipe.h"

bool closed = false;
int g_tnumb = 8;
int tasks_numb = 15000;
int milli_per_submit = 500;  // 0.5s

void manager(hipe::DynamicThreadPond* pond) {
    enum class Action { add, del };
    Action last_act = Action::add;

    int unit = 2;
    int prev_load = 0;
    int max_thread_numb = 200;
    int min_thread_numb = 8;
    int total = 0;

    while (!closed) {
        auto new_load = pond->resetTasksLoaded();
        auto tnumb = pond->getExpectThreadNumb();
        total += new_load;

        printf("threads: %-3d remain: %-4d loaded: %d\n", tnumb, pond->getTasksRemain(), new_load);
        fflush(stdout);

        // if (!prev_load),  we still try add threads and have a look at the performance
        if (new_load > prev_load) {
            if (tnumb < max_thread_numb) {
                pond->addThreads(unit);
                pond->waitForThreads();
                last_act = Action::add;
            }

        } else if (new_load < prev_load) {
            if (last_act == Action::add && tnumb > min_thread_numb) {
                pond->delThreads(unit);
                pond->waitForThreads();
                last_act = Action::del;
            } else if (last_act == Action::del) {
                pond->addThreads(unit);
                pond->waitForThreads();
                last_act = Action::add;
            }
        } else {
            if (!pond->getTasksRemain() && tnumb > min_thread_numb) {
                pond->delThreads(unit);
                pond->waitForThreads();
                last_act = Action::del;
            }
        }
        prev_load = new_load;

        // I think that the interval of each sleep should be an order of magnitude more than the time a task cost.
        hipe::util::sleep_for_seconds(1);  // 1s
    }

    total += pond->resetTasksLoaded();
    hipe::util::print("total load ", total);
}

int main() {
    hipe::DynamicThreadPond pond(g_tnumb);

    // total 0.1s
    auto task1 = [] { hipe::util::sleep_for_milli(20); };
    auto task2 = [] { hipe::util::sleep_for_milli(30); };
    auto task3 = [] { hipe::util::sleep_for_milli(50); };

    auto tasks_per_second = 600;

    hipe::util::print("Submit ", tasks_per_second, " task per second");
    hipe::util::print("So we hope that the threads is able to load [", tasks_per_second, "] task per second");
    hipe::util::print(hipe::util::boundary('=', 65));

    // create a manager thread.
    std::thread mger(manager, &pond);

    // 600 tasks per second
    int count = (tasks_numb / 3) / 100;
    while (count--) {
        // 300 tasks
        for (int i = 0; i < 100; ++i) {
            pond.submit(task1);
            pond.submit(task2);
            pond.submit(task3);
        }
        hipe::util::sleep_for_milli(500);  // 0.5s
    }

    // wait for task done and then close the manager
    pond.waitForTasks();

    closed = true;
    mger.join();
}
```