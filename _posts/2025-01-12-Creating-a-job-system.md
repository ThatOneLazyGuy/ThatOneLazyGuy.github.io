---
title: Creating a generic job system that just works
date: 2025-01-12 17:50:00 +0100
categories: [blog]
tags: [c++, multithreading, university]
media_subpath: /assets/2025-01-12-Creating-a-job-system/
image: Logo_BUas_RGB.png
---

As a second year student at Buas (Breda University of Applied Sciences) I have been working on a job system for the past 8 weeks to more easily multi-thread parts of a game engine.

In this blog we are going to create a job system which lets you easily turn any function into a job to be performed on a different thread, we'll be using many useful features of the c++17 standard and creating a simple API to be able to work with multiple threads quickly and easily.

### Where to start?

Well, anyone who knows threads can tell you that creating them is slow, very slow; that is why we first start with a thread pool.
We create a number of threads that will perform our jobs for us, but instead of creating them when new work is available, we simply keep them in a sleeping state until they are needed.
How do we know the amount of threads we should create? When creating "worker threads" it makes sense to base the thread count on the amount of logical cores your CPU has, since making more threads than local cores will cause these worker threads to fight for time on the same, cores which doesn't providing any extra performance for our job system.

### Managing our threads
To manage our threads and add our jobs we'll create a manager class.
In here we store the vector of threads and we'll put our function to add jobs for the worker threads to perform.

```cpp
class JobManager
{
public:
    JobManager();
    ~JobManager();

    [[nodiscard]] size_t GetThreadCount() const { return m_workerThreads.size(); }

private:

    void RunJobs();

    std::vector<std::thread> m_workerThreads;
};
```

To get our how many threads we should create, we use `std::thread::hardware_concurrency()` which gives us a the logical core count of our CPU (Logical cores might also be called hardware threads; since this can cause confusion between software threads, I will be calling them logical cores). Now we can create a thread for each logical core, maximizing the thread count without the threads fighting for time on the logical cores of our CPU.
Then in the constructor of our JobManager we create that amount of threads running our worker function, “RunJobs” (our thread is constructed with a function pointer and its arguments; since “RunJobs” is a member of JobManager, its first argument has to be “this”).

```cpp
JobManager::JobManager()
{
    const size_t threadCount = std::thread::hardware_concurrency();

    for (size_t i = 0; i < threadCount; i++)
    {
        m_workerThreads.emplace_back(&JobManager::RunJobs, this);
    }
}
```

Now that we have our pool of threads based on our logical core count, we might think we are done, but unfortunately there is one more problem we want to solve: \
**context switching.**\
Threads are managed by the OS to decide when and on which logical core to run them; this means that a thread and its context can be moved from one logical core to another by the OS based on which logical core has time.
Since we want our threads to most effectively take advantage of all our logical cores, we want to tie each thread to one of those logical cores. Unfortunately, there is no standardized way to do this, but since I am using Windows, my code would look something like this:

```cpp
void JobManager::SetThreadAffinities()
{
    for (size_t i = 0; i < GetThreadCount(); i++)
    {
        const auto nativeThreadHandle = m_workerThreads.at(i).native_handle();
        const DWORD_PTR dw = SetThreadAffinityMask(nativeThreadHandle, 1ll << i);

        if (dw == 0) printf("Error setting thread affinity: %lu\n", GetLastError());
    }
}
```

We then extend our class definition and constructor declaration like so:

```cpp
class JobManager
{
public:
    JobManager();
    ~JobManager();

    [[nodiscard]] size_t GetThreadCount() const { return m_workerThreads.size(); }

private:
    void RunJobs();
    void SetThreadAffinities();
    
    std::vector<std::thread> m_workerThreads;
};
```

```cpp
JobManager::JobManager()
{
    const size_t threadCount = std::thread::hardware_concurrency();

    for (size_t i = 0; i < threadCount; i++)
    {
        m_workerThreads.emplace_back(&JobManager::RunJobs, this);
    }

    SetThreadAffinities();
}
```

### Threads and their work

#### The shape of our tasks

To give out tasks the most flexibility and to allow for generic task we are going to use a lot of convenient tricks found in c++17.\
There are a few problems we have:
1. Storing functions with different return types or parameters in the same list.
2. Storing the function itself and the arguments for the function together.
3. Getting back the return value of a completed task.
4. Knowing which tasks have finished executing.

Let's start at the beginning with problems 1 and 2, where we can solve these problems simply by not storing any of this information directly but instead storing a lambda:

```cpp
std::function<void()> lambda = [function, args...]()
{
    function(args...);
}
```

We use a lambda in order to capture the function and its arguments, which will allow us to execute this later. Since the lambda doesn’t take any arguments anymore, nor does it return anything, we can simply describe it to be of the type: `std::function<void()>`, this allows us to store different tasks together in one list.

Unfortunately, we still have two problems remaining; however, we can conveniently solve these with another trick from modern C++: `std::promise<>`.
A promise works together with `std::future<>` to deliver a *promised* value in the *future*. In our case we first want to describe the type of the promised value; this will be our return value type.

```cpp
std::promise<ReturnType> ourPromise;
std::future<ReturnType> ourFuture = ourPromise.get_future();
```

We get a future of the same type back from our promise object, which we can later use to both check the state of our task and get our return value.

To combine these two solutions together, we have to keep a few things in mind, however. We need to get the future object back from the promise before the lambda captures it. The lambda needs to capture the promise in order to fulfil the promise by setting its value to the return value of the function. The only problem is that std::promise is non-movable and can’t be captured by the lambda after we created it; that is why instead of creating a new object on the stack, we create a shared_ptr of a promise that we can then capture with the lambda.

```cpp
auto promise = std::make_shared<std::promise<ReturnType>>();
std::future<ReturnType> future = promise->get_future();

std::function<void()> lambda = [promise, function, args...]()
{
    promise->set_value(function(args...));
}
```

Now we have a generic task type to create a list out of while still being able to execute any function and get its return value back.

#### Tasks in and out of a list

Since we can now store our tasks in a single list, let's do that. For our purpose, the list type `std::queue<>` will work well; this allows us to use a FIFO (First In First Out) structure where the first task we added also gets picked up by a worker thread first. To add a job to the queue, we can use our earlier code to create a function called “AddToQueue”:

```cpp
template <typename Function, typename... Args>
decltype(auto) JobManager::AddToQueue(Function function, Args... args)
{
    using ReturnType = std::invoke_result_t<Function, Args...>;

    auto promise = std::make_shared<std::promise<ReturnType>>();
    std::future<ReturnType> future = promise->get_future();

    std::function<void()> task = [promise, function, args...]()
    {
        if constexpr (std::is_same_v<ReturnType, void>)
        {
            function(args...);
            promise->set_value();
        }
        else
        {
            promise->set_value(function(args...)); 
        }
    }

    QueueTask(task);
    return future;
}
```

To fill in the missing pieces of my earlier example, I’ve made the AddToQueue function templated to allow for different function types and arguments. Template types are also used with `std::invoke_result_t<>` to get back the explicit return type of that function to use for the promise and future. Another possibly confusing addition is to the lambda function: `if constexpr (std::is_same_v<ReturnType, void>)`, but all this does is check if the ReturnType is equal to void at compile time. It does this in order to change how to call: `promise->set_value()` since a promise with type void means there is no actual value to set, and thus the function is required to have no arguments.

Since we are working with a queue being accessed by multiple threads, we want to use an `std::mutex` in the JobManager to make sure the threads do not try to access the shared queue simultaneously when they aren’t supposed to. 
The final adding of the task to the queue happens using QueueTask(), which acquires a lock on the mutex to push the task to the queue and then returns, which automatically releases the lock.

```cpp
void JobManager::QueueTask(const std::future<void()>& task)
{
    std::scopped_lock lock(m_mutex);
    m_taskQueue.push(task);
}
```
We then finally return the future so the user can use this to get information about the promise/task.

#### Executing tasks

Now that we have some tasks and some threads to perform them, we can describe how exactly they will be performing their tasks. A worker thread doesn't do much when no tasks are available; it should simply remain idle and wait for the JobManager to notify it about tasks becoming available; this way, no CPU usage is wasted polling the queue for new tasks constantly.
A convenient way to do both of these things is to use an `std::condition_variable`. A condition_variable can be used to check if a certain condition is true, continue if it is, or block the thread if it isn't true yet. To notify when the condition_variable should check its condition, we can use: `condition_variable::notify_one()` to notify a single thread or: `condition_variable::notify_all()` to notify all threads.\
\
To start off, we want to have a loop that uses the condition_variable to check if new work is available. The wait function on the condition_variable uses a lock to check if a predicate is true and continues if it is.
Our predicate will check two things:
1. If the queue isn't empty.
2. If the new variable m_shouldExit is true, if this is true, the worker thread should return.

If either is true, the condition_variable will continue.

```cpp
void JobManager::RunJobs()
{
    const auto predicate = [this]{ return !m_taskQueue.empty() || m_shouldExit; }

    while (true)
    {
        std::function<void()> task;
        {
            std::unique_lock lock(m_mutex)
            m_condition.wait(lock, predicate); // Wait to be notified and check the predicate

            if (m_shouldExit) return;

            task = m_taskQueue.front();
            task.pop();
        }
        task();
    }
}
```

We create our predicate to test as a lambda function, we enter the loop and create a task to fill with our task to execute later. We open a new scope to create the unique_lock in, this is done in order to automatically release the lock when the unique_lock goes out of scope after acquiring the task or when we return the function.
The task is taken from the front of the queue, and then the queue is popped from the front to discard the now picked-up task. The worker exits the scope, thereby releasing the lock. It then finally executes its picked-up task.

Now that the worker thread is waiting for the main thread to notify it about new work, we have to make a small edit to the `QueueTask` function:

```cpp
void JobManager::QueueTask(const std::future<void()>& task)
{
    std::scopped_lock lock(m_mutex);
    m_taskQueue.push(task);
    m_condition.notify_one();
}
```

And we define our destructor to notify all the threads they should exit and to then join them:

```cpp
JobManager::~JobManager()
{
    {
        std::scopped_lock lock(m_mutex)
        m_shouldExit = true;
        m_condition.notify_all();
    }

    for (auto& workerThread : m_workerThreads)
        workerThread.join();
}
```

With all this, here is our new JobManager declaration:

```cpp
class JobManager
{
public:
    JobManager();
    ~JobManager();

    template <typename Function, typename... Args>
    decltype(auto) AddToQueue(Function function, Args... args);

    [[nodiscard]] size_t GetThreadCount() const { return m_workerThreads.size(); }

private:
    void RunJobs();
    void QueueTask(const std::future<void()>& task);

    std::vector<std::thread> m_workerThreads;
    bool m_shouldExit = false;

    std::queue<std::function<void()>> m_taskQueue;
    std::condition_variable m_condition;
    std::mutex m_mutex;
};
```

Now that we have threads to execute the work, a way to add the work, and a way to get the data back, we essentially have a working job system!

Although there are some limitations that we might want to work around.

### Grouping work

When you want to multithread something, a lot of the time this is because you have a large number of things to process. A large number of tasks to process currently requires you to add a lot of jobs to the queue by using: `AddToQueue`, this can be quite slow since the threads have to frequently try to access the lock to grab a new task from the queue, causing a lot of contention.
That's why we probably want a function that automatically creates a few large tasks out of a large number of function calls; we dispatch tasks to the queue:

```cpp
template <typename Iterator, typename Function, typename... Args>
decltype(auto) JobManager::Dispatch(Iterator startIterator, const size_t taskCount, Function function, Args... args)
{
    if (taskCount <= 0) return;

    const size_t groupCount = GetThreadCount();
    const auto groupSize = static_cast<size_t>(std::ceil(static_cast<float>(taskCount) / static_cast<float>(groupCount)));

    std::vector<std::future<void>> futures;

    for (size_t groupIndex = 0; groupIndex < groupCount; groupIndex++)
    {
        const size_t startIndex = groupIndex * groupSize;  // Index of the first element in the current group
        if (startIndex >= taskCount) continue;  // Skip scheduling a task if we have exceeded the task count prematurely

        auto promise = std::make_shared<std::promise<void>>();
        futures.push_back(promise->get_future());

        const auto localStartIterator = startIterator + startIndex;
        const size_t maxLocalTaskCount = taskCount - startIndex;

        // Size of the current group, either groupSize or a smaller size depending on the amount of elements left
        const size_t localGroupSize = std::min<size_t>(groupSize, maxLocalTaskCount);

        auto taskGroup = [promise, localStartIterator, localGroupSize, function args...]
        {
            for (size_t localIndex = 0; localIndex < localGroupSize; localIndex++)
            {
                const auto iterator = localStartIterator + localIndex;

                function(*iterator, args...);
            }

            promise->set_value();
        };

        QueueTask(taskGroup);
    }

    return futures;
}
```

The dispatch function is designed to work on any container type (or pointer type) and will create groups of tasks with the first parameter being of the same type as the container's stored type.\
The parameters of the function are:
- The begin iterator for any container type that can be iterated through one by one.
- The amount of elements in the container.
- The function to execute.
- The extra arguments for this function.

We first calculate how many task groups we will make based on the thread count and then the size of each of those groups.

```cpp
const size_t groupCount = GetThreadCount();
const auto groupSize = static_cast<size_t>(std::ceil(static_cast<float>(taskCount) / static_cast<float>(groupCount)));

std::vector<std::future<void>> futures;
```

For each group we calculate what its first iterator will be and how large our group will be, as well as creating a promise for this task group:

```cpp
for (size_t groupIndex = 0; groupIndex < groupCount; groupIndex++)
{
    const size_t startIndex = groupIndex * groupSize;  // Index of the first element in the current group
    if (startIndex >= taskCount) continue;  // Skip scheduling a task if we have exceeded the task count prematurely

    auto promise = std::make_shared<std::promise<void>>();
    futures.push_back(promise->get_future());

    const auto localStartIterator = startIterator + startIndex;
    const size_t maxLocalTaskCount = taskCount - startIndex;

    // Size of the current group, either groupSize or a smaller size depending on the amount of elements left
    const size_t localGroupSize = std::min<size_t>(groupSize, maxLocalTaskCount);

    // more code....
}
```

A lambda function is created that loops through each of the iterators in its group and executes the function for that element and the extra arguments; after that, the lambda sets the task group's promise so the future is notified about the task having finished. The task group is then added to the queue:

```cpp
auto taskGroup = [promise, localStartIterator, localGroupSize, function args...]
    {
        for (size_t localIndex = 0; localIndex < localGroupSize; localIndex++)
        {
            const auto iterator = localStartIterator + localIndex;

            function(*iterator, args...);
        }

        promise->set_value();
    };
    
QueueTask(taskGroup);
```

Although this function is useful, it has some downsides to keep in mind as well:
1. The function provided needs to have the first parameter of the same type as the type the container holds.
2. The `std::vector` of futures returned by this function has to be of type void since the tasks are in groups and their return values can't individually be returned.

But for a large number of small tasks, this can be extremely useful to allow for performant multithreading.

### Getting information from the future

#### Get and Wait

`AddToQueue` and `Dispatch` return one or more future objects; these can be used to get the values back from out tasks.

`future.wait()`: This method can be used to block this thread to wait for the promise to be fulfilled by our task.
`future.get()`: This method also blocks this thread and waits for the promise to be fulfilled. However, it also returns back the value that the promise was set to, marking the future as invalid afterwards.

#### Finished?

While getting the values and waiting for them is useful, of course, what if we simply want to check if the task is done without blocking this thread? Unfortunately, there isn't a straightforward method in std::future to do this; luckily, we can do it with a bit of trickery.

```cpp
bool JobManager::IsFinished(const std::future& future)
{
    return (future.wait_for(std::chrono::seconds(0)) == std::future_status::ready)
}
```

In the `IsFinished` function we wait for 0 seconds, which will tell us the state of the future after we finished waiting for no time. If the state of the future is ready, that means it's finished.
Now that we know the state of our future, we can choose to only get our value from it back if it's true; otherwise, we can simply let the rest of the program run. This type of check can be useful for when something is being loaded in on a worker thread and you want to show that it is still running, or if you are streaming something in and only want to process it during the frame that it finishes but continue otherwise.

### Conclusion

The final JobManager declaration:

```cpp
class JobManager
{
public:
    JobManager();
    ~JobManager();

    template <typename Function, typename... Args>
    decltype(auto) AddToQueue(Function function, Args... args);

    template <typename Iterator, typename Function, typename... Args>
    decltype(auto) Dispatch(Iterator startIterator, const size_t taskCount, Function function, Args... args);

    [[nodiscard]] bool IsFinished(const std::future& future);
    [[nodiscard]] size_t GetThreadCount() const { return m_workerThreads.size(); }

private:
    void RunJobs();
    void QueueTask(const std::future<void()>& task);

    std::vector<std::thread> m_workerThreads;
    bool m_shouldExit = false;

    std::queue<std::function<void()>> m_taskQueue;
    std::condition_variable m_condition;
    std::mutex m_mutex;
};
```

A lot of different programs require multithreading, and a job system that can be reused in different ways can be a useful tool to improve performance or add more features without much trouble.

I hope this blog was helpful in learning more about job systems and can help you more effectively work with multiple threads!