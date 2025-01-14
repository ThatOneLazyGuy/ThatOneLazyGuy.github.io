---
title: Creating a job system
date: 2025-01-12 17:50:00 +0100
categories: [blog]
tags: [c++, multithreading, university]
media_subpath: /assets/2025-01-12-Creating-a-job-system/
---

## Intro
---

This blog will be about creating a simple job system that works for generic tasks in c++ 17.

### Where to start?
Well, anyone who knows threads can tell you that creating them is slow, very slow, that is why we first start with a thread pool.
We create a number of threads which will perform our jobs for us, but instead of creating them when new work is available we simply keep them in a sleeping state until they are needed.
How do we know the amount of threads we should start? When creating "worker threads" it makes sense to divide the count on the amount of cores your CPU has, since making more threads then CPU cores will cause these worker threads to fight for time on the same cores not providing any extra performance for our job system.

### Managing our threads
To manage our threads and add our jobs we'll create a manager class.
In here we store the vector of threads and we'll put our function to add jobs for the worker threads to perform.

```cpp
class JobManager
{
public:
    [[nodiscard]] size_t GetThreadCount() const { return m_workerThreads.size(); }

private:
    JobManager();
    ~JobManager();

    void RunJobs();

    std::vector<std::thread> m_workerThreads;
};

```

To get our core count we use `std::thread::hardware_concurrency()` this will tell us our cpu's logical core count.
Then in the constructor of our JobManager we create that amount of threads running our worker function: "RunJobs" (we pass in "this" as our first argument since RunJobs is a member function of our JobManager).

```cpp
const size_t threadCount = std::thread::hardware_concurrency();

for (size_t i = 0; i < threadCount; i++)
{
    m_workerThreads.emplace_back(&JobManager::RunJobs, this);
}
```

### Notify threads about work

### Grouping jobs together

## Conclusion
--- 

### Limitations
Due to our focus on simplicity and ease of use there are currently some important limitations to keep in mind when using our job system.

1. A job can add other jobs to the queue which my cause deadlock since all threads can be stuck waiting for jobs to complete which won't be picked up by other threads anymore.

### Possible improvements
There are currently a lot of ways forward with many possible additions that our job system could be benefit from:

- Work stealing to ensure all threads are working on some job while there is work available.
- Parent child job relationships.