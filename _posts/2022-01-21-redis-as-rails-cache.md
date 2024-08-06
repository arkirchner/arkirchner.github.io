---
title: "Redis for caching in Rails applications"
date: 2022-01-21T18:53:25+09:00
categories:
  - blog
tags:
  - Rails
  - Cache
  - Redis
---

# Why use Redis for caching?

Redis is widely used in Rails as database and a cache store. This guide will
go into details about the best practices to use Redis for caching.

In a typical Rails application, Redis can serve two primary purposes:

1. **Database Use Case**: Redis stores key value data, such as session information. Losing this data can disrupt application operations.
2. **Cache Use Case**: Redis stores temporary data to improve performance. If cache data is lost, it can be regenerated during subsequent requests.

Best Practices for Redis Usage
Separate Instances for Caching and Database

It is critical that separate Redis instances for caching and database are used.
For the database usecase we need to ensure that always enough memory is 
avalible. Caching will use as much memory as is avalible. Campanies often use 
**volatile-lru** and **volatile-random** eviction strategy to avoid server 
seperation. **volatile** in this context means cache keys with an expire date.
The database use case (without expiration data) is not affected. 

This mix of concerns not recommended for use in production.

Caching is meant to scale your application. A lot cache keys with expiration 
costs valuable cache space. 


# Efficient cache memory management

Manually setting expiration keys is not recommended due to inefficient memory 
management and potential performance degradation. Instead, it's better to let 
Redis handle memory cleanup:

> It is also worth noting that setting an expire to a key costs memory, so using a policy like allkeys-lru is more memory efficient since there is no need to set an expire for the key to be evicted under memory pressure.
[source](https://redis.io/topics/lru-cache)

# Differentiating Cache-Only Redis Server

When configuring a cache-only Redis server, specific settings ensure optimal performance. The Rails Guides provide excellent recommendations:

> For a cache-only Redis server, set maxmemory-policy to one of the variants of allkeys. Redis 4+ supports least-frequently-used eviction (allkeys-lfu), an excellent default choice. Redis 3 and earlier should use least-recently-used eviction (allkeys-lru).
[source](https://guides.rubyonrails.org/caching_with_rails.html#activesupport-cache-rediscachestore)


For resources with many records, caching all records isn't necessary to improve 
performance. Most views only show a subset of records (e.g., new or popular 
records). The LRU (Least Recently Used) cache configuration ensures frequently 
requested records remain in the cache, significantly reducing database load and 
improving response times for popular requests.

# Recommended server configuration for caching


## maxmemory-policy: allkeys-lfu

This policy ensures the least frequently used cache entries are evicted first, maintaining a cache of the most frequently accessed data.

## maxmemory: xxGB

This setting limits the memory Redis can use. The LRU cache configuration 
utilizes 100% of the available cache. By default, this is set to zero, which 
can cause Redis to crash. Ensure enough memory is allocated for your server's 
system requirements.

## Relatively Low Timeouts

Set a fast timeout in your Rails application. If cache key lookups are slow, 
regenerating the cache might be more efficient. The timeout duration is 
dependend on the speed of the network connection between app and server.

```ruby
cache_servers = %w(redis://cache-01:6379/0 redis://cache-02:6379/0)
config.cache_store = :redis_cache_store, { url: cache_servers,
  read_timeout:       0.2, # Defaults to 1 second
  write_timeout:      0.2, # Defaults to 1 second
}
```
[source](https://guides.rubyonrails.org/caching_with_rails.html#activesupport-cache-rediscachestore)


## Scaling and Monitoring

- **Memory Usage**: Redis memory usage will be close to 100% for caching purposes (keys are only removes if no cache is available).
- **Total Memory**: Avoid using 100% of your server's total memory. Some memory needs to be reserved for the system opperation.
- **Cache Hit Ratio**: Monitor the cache hit ratio. A decreasing hit cache ratio, suggests a need for scaling.
- **CPU**: Redis is single threaded. This means only a single CPU core can be utilized. The single core utilization needs to be monitored.

# Conclusion

It's strongly recommended to separate Redis as a database and caching use cases to optimize performance.

By following these guidelines, you can leverage Redis to significantly improve the performance and decreasing DB load of your Rails application.
