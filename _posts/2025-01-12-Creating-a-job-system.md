---
title: Creating a job system
date: 2025-01-12 17:50:00 +0100
categories: [blog]
tags: [c++, multithreading, university]
media_subpath: /assets/2025-01-12-Creating-a-job-system/
---

## Intro
---

The goal is to create a job system that is generic for a lot of tasks and easy to use.

## Body
---

### Creating a job pool

### Notify threads about work

### Waiting for jobs

### Grouping jobs together

### Atomic queue

## Conclusion
--- 

### Limitations
Due to our focus on simplicity and ease of use there are currently some important limitations to keep in mind when using our job system.

1. A job can add other jobs to the queue which my cause deadlock since all threads can be stuck waiting for jobs to complete which won't be picked up by other threads anymore.

### Possible improvements
There are currently a lot of ways forward with many possible additions that our job system could be benefit from:

- Work stealing to ensure all threads are working on some job while there is work available.
- Parent child job relationships