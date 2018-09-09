title: Build a better client
date: 2018-08-26 20:36:29
tags: distributed-system
---

<div style="text-align:center">
<iframe src="//www.slideshare.net/slideshow/embed_code/key/pF9d6lGQKScGHc" width="595" height="485" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC; border-width:1px; margin-bottom:5px; max-width: 100%;" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="//www.slideshare.net/kakashiliu/build-a-better-client-111628081" title="Build a better client" target="_blank">Build a better client</a> </strong> from <strong><a href="https://www.slideshare.net/kakashiliu" target="_blank">cc liu</a></strong> </div>
</div>

# Preface

This is a talk I presented at the internal company meeting. It mainly addresses how could we create better client behavior for modern software architecutre. We all know that microservices becomes more and more popular nowadays. Everybody notices about the benefit of microservices that each team can do development, deployment, and service scaling independently. It also makes each component becomes more understandable and maintainable. However, decomposing an application into different services introduces the reliability problems. Because these services usually are connected to each other by the computer network instead of in the single machine, we need to be more careful to handle network errors and service errors.

# Fallacies of distributed computing

According to the [wiki](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing), there are 8 fallacies of distributed computing. I think everyone design a system connected by networking should get familiar with this knowledge. It's so important to understand that the computer network is not reliable and that's the reason why we have TCP protocol. Even though TCP does lots of thing for us, we still need to take care of the effect of these fallacies.

1. The network is reliable.
2. Latency is zero.
3. Bandwidth is infinite.
4. The network is secure.
5. Topology doesn't change.
6. There is one administrator.
7. Transport cost is zero.
8. The network is homogeneous.

# 7 resilience policies

I suggest everyone can adopt these resilience policies for improving the client-side program.
Actually, each policy can be a big topic, here I only provide some notes.

## Retry

Basically we use retry to overcome transient failures which might be server side errors or networking errors.
Here I found out the awesome [article](http://allyouneedisbackend.com/blog/2017/09/15/how-backend-software-should-retry-on-failures/) to address different scenarios that we should take care of.

- DNS error
- Connection error
- Timeout

If it's an application error, we can perform retrying based on following status code.
- 503 (Service unavailable)
- 429 (too many requests)
- 408 (request timeout)
- 500 (Internal server error)

It's also important to know if this call is idempotent so that retrying won't cause data duplication and other side effects.
In addition, if server returns 400, you should probably fix your data instead of performing the retry.

## Retry + Back off + jitter

Although the retry mechanism makes client become more resilient, it also can hurt the server in certain situations. Consider a server is overloading and couldn't deal with many requests at the same time, repetitive tries will make the situation worse. Back off and jitter can help server recovery from failing.

According to this [AWS article](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/):

![backoff](https://d2908q01vomqb2.cloudfront.net/fc074d501302eb2b93e2554793fcaf50b3bf7291/2017/10/03/exponential-backoff-and-jitter-blog-figure-4.png)

1. Adding backoff can slow clients down and reduce wasteful invocations. In addition, Server loading is decreased since the calls occur less and less frequently.

2. Adding jitter can resolve the issue that clusters of calls come to server and make server super busy within the certain time period. Adding randomness delay between each retry can spread out the server loading spikes.

![backoff](/img/2018-08/jitter.png)

## Circuit Breaker

The circuit breaker is used between the client and server. When a server doesn't response certain requests for a while (maybe after multiple retries with given timeout), we can adopt circuit breaker to stop client to call server wastefully.

## Timeout

Set up the timeout of every invocation properly can save client resources efficiently. Don't let an invocation without timeout occupied resources forever.

## Cache

Use cache to reduce the frequency of calls. It not only shortens latency of invocation but also reduces server-side loading. But don't forget. There are 2 hard problems in computer science: cache invalidation, naming things. Design a proper cache invalidation is not easy.

## Bulkhead isolation

This is an interesting policy which saperating client resources for different purposes. For example, we can use different thread pools for different services. Once one of the remote server crushed or became abnormal, it won't affect whole client performance. There is a famous implementation in the Netflix library Hystrix, you can refer to https://stackoverflow.com/questions/30391809/what-is-bulkhead-pattern-used-by-hystrix.

## Fallback

Return reasonable responses instead of showing error messages.

# Conclusion

Building a better client not only enhances your reliability of client-side application but also improves the stability of whole systems. It's worth considering to adopt these policies in your client-side program.

# Reference

1. https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing
2. http://allyouneedisbackend.com/blog/2017/09/15/how-backend-software-should-retry-on-failures/
3. https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/
4. https://stackoverflow.com/questions/30391809/what-is-bulkhead-pattern-used-by-hystrix
5. https://dzone.com/articles/performance-patterns-in-microservices-based-integr-1

