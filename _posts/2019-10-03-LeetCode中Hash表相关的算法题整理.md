---
layout:     post
title:      "LeetCode中Hash表相关算法题整理"
subtitle:   "Hash Usage in LeetCode"
date:       2019-10-03
author:     S.L
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Algorithm
---
    
> 关于哈希在LeetCode中出现的题目的相关解法的总结。注：所有代码来自互联网。


### Logger Rate Limiter(359)
> 使用HashSet来保存消息的唯一性，使用Queue来保证先进先出。

```java
class Logger {
    Queue<Tuple> q = new ArrayDeque<>();
    Set<String> dict = new HashSet<>();
    
    public boolean shouldPrintMessage(int timestamp, String message) {
        // 将10s之前的数据从队列中清除掉
        while (!q.isEmpty() && q.peek().t <= timestamp - 10) {
            Tuple next = q.poll();
            dict.remove(next.msg); // 同时清理message
        }
        if (!dict.contains(message)) {
            q.offer(new Tuple(timestamp, message));
            dict.add(message);
            return true;
        }
        return false;
    }
    
    private static class Tuple {
        int t;
        String msg;
    
        public Tuple(int t, String msg) {
            this.t = t;
            this.msg = msg;
        }
    }
}
```
### Design Twitter(355)
被关注者：
> Followee: One who is followed (has his/her posts monitored by another user).


```java
public class Twitter {
    // key is fans's id, value is his followees
    Map<Integer, Set<Integer>> fans = new HashMap<>();
    // key is author's id, value is his tweets order by time desc
    Map<Integer, LinkedList<Tweet>> tweets = new HashMap<>();
    int cnt = 0;

    public void postTweet(int userId, int tweetId) {
        if (!fans.containsKey(userId)) 
            fans.put(userId, new HashSet<>());
        fans.get(userId).add(userId); // one can receive his posts
        if (!tweets.containsKey(userId)) 
            tweets.put(userId, new LinkedList<>());
        tweets.get(userId).addFirst(new Tweet(cnt++, tweetId)); // add first
    }

    public List<Integer> getNewsFeed(int userId) {
        if (!fans.containsKey(userId)) 
            return new LinkedList<>(); // no tweets posted yet
            
        // big head heap
        PriorityQueue<Tweet> feed = new PriorityQueue<>((t1, t2) -> t2.time - t1.time);
        // make sure fans is initialized and tweets is initialized
        fans.get(userId).stream()
            .filter(f -> tweets.containsKey(f))
            .forEach(f -> tweets.get(f).forEach(feed::add)); // construct heap
        List<Integer> res = new LinkedList<>();
        // poll the biggest one every time
        while (feed.size() > 0 && res.size() < 10) 
            res.add(feed.poll().id);
        return res;
    }

    public void follow(int followerId, int followeeId) {
        if (!fans.containsKey(followerId)) 
            fans.put(followerId, new HashSet<>());
        fans.get(followerId).add(followeeId);
    }

    public void unfollow(int followerId, int followeeId) {
        if (fans.containsKey(followerId) && followeeId != followerId) 
            fans.get(followerId).remove(followeeId);
    }

    class Tweet {
        int time;
        int id;

        Tweet(int time, int id) {
            this.time = time;
            this.id = id;
        }
    }
}
```
## References
- https://www.leetcode.com

> 本文首次发布于 [S.L's Blog](http://elsef.com), 作者 [@stuartlau](http://github.com/stuartlau) ,
转载请保留原文链接.
