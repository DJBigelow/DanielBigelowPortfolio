---
layout: single
title:  "Dealing With Rate-Limiting in Node"
date:   2021-11-20 19:00:00 -0600
categories: javascript node web-dev
---

One of my projects while working at Snow College's IT department was to build a website that allowed instructors and TA's to select a series of dates and create an assignment in Canvas on each of those days. One of the technical challenges I faced was rate limiting on the Canvas API. 

The problem occured when testing posting an assignment for multiple days a week for each week in a semester. Making that many requests to the Canvas API at once would exceed the rate limit and cause some requests to fail, meaning multiple assignments simply wouldn't be posted. I had to find a way of limiting the rate requests were made to the Canvas API and to retry requests that failed. Let's start with what I did to reduce the rate of requests.

## Pooling Axios Instances

We were using Node + Express for our backend at the time and Axios for HTTP web requests. My idea for limiting the total number of requests happening at once was to make a limited size pool of Axios instances that web requests would have to be made through. I found a lightweight (library)[https://github.com/coopernurse/node-pool] for creating pools of generic resources and used it to create a service for acquiring and releasing Axios instances.   

```js
import * as GenericPool from 'generic-pool'
import axios, {AxiosInstance} from 'axios'


export default class AxiosPoolService {
  private clientFactory: GenericPool.Factory<any>
  private requestPool: GenericPool.Pool<AxiosInstance>

  constructor(maxClients = 10) {
    this.clientFactory = {
      create: async () => {
        new Promise(resolve => resolve(axios.create()))
      },
      destroy: async(_axiosInstance: AxiosInstance) => {
        //Axios instances don't need to be destroyed/disconnected
        //They still need to be released from the pool after usage
      }
    }

    const poolOptions: GenericPool.Options = {
      max: maxClients
    }

    this.requestPool = GenericPool.createPool(this.clientFactory, poolOptions)
  }

  acquireResourcePromise(): Promise<AxiosInstance> {
    return this.requestPool.acquire()
  }

  releaseResource(axiosInstance: AxiosInstance): void {
    this.requestPool.release(axiosInstance)
  }
}
```

In testing, this improved the situation but couldn't completely resolve it, even when fine-tuning the pool size. I could've reduced the pool size to 1, but at that point I may as well have made each request sequentially, which my supervisor felt would be too slow and wanted to keep as a backup in case other solutions failed. Luckily, the next part of the solution worked out well.

## Circuit Breakers

My next move was to run the Axios requests through a circuit breaker. A circuit breaker is a mechanism that will monitor the status of jobs fed through it and trip if a certain number or percentage of those jobs fail. When the circuit breaker trips, it will wait a given amount of time before resetting and allowing jobs to start running again. This solution was a great fit for the problem at hand as when requests would start failing due to rate limiting, the circuit breaker would delay them for a given time, allowing the rate limiter time to start accepting requests again. Below is my code utilizing the [Opossum](https://github.com/nodeshift/opossum) library's circuit breaker:

```js
  async submitAssignments(courseId: number, attendanceDates: Date[]) {
    const assignmentGroup = await this.getOrCreateAttendanceAssigmentGroup(courseId);

    const assignments = this.createAssignmentObjects(courseId, attendanceDates, assignmentGroup.id)

    const url = `${canvasUrl}/api/v1/courses/${courseId}/assignments`;

    logger.info(`Posting attendance assignments for course ${courseId}`)

    this.postAssignmentsWithCircuitBreaker(assignments, url)
  }

  private postAssignmentsWithCircuitBreaker(assignments: Assignment[], url: string) {
    const circuitBreakerOptions: CircuitBreaker.Options = {
      timeout: 1000,
      errorThresholdPercentage: 25,
      resetTimeout: 250
    }

    const circuitBreaker = new CircuitBreaker(this.axiosPostWithPool, circuitBreakerOptions);
    circuitBreaker.fallback(this.failedRequestFallback)

    assignments.forEach(assignment => {
      circuitBreaker.fire(url, {assignment}, webOptions).then(() => {
        logger.info(`Assignment created `, assignment.name)
      })
    })
  }

  private async axiosPostWithPool(url: string, data: any, options: AxiosRequestConfig) {
    const axiosInstance = await axiosPoolService.acquireResourcePromise();
    await this.axiosPost(axiosInstance, url, data, options)
    axiosPoolService.releaseResource(axiosInstance)
  }
```

In the end, I was able to quickly create assignments for every day of a semester without any of them not being posted due to rate limiting.