+++
title = "Redis Lock"
date = "2016-10-16T14:07:41+08:00"
tags = ["redis","distribute","lock"]
categories = ["programming","cache"]
menu = ""
banner = "banners/redis.jpg"
+++
simple distributed lock on redis and how to improve it with the redlock algorithm.the pros and cons the redlock algorithm.
<!--more-->
+++
title = "redis string operations"
date = "2016-10-14T16:20:41+08:00"
tags = ["redis","string","rate limit"]
categories = ["programming","cache"]
menu = ""
banner = "banners/redis.jpg"
+++
Notes about the commands of Redis String operations.Especially some scenarios that would be much useful.
<!--more-->


<!--more-->
# Redis Lock
## Introduction/Usage

 In distributed system,use distributed lock to guarantee serial operation for avoiding race-condition conflicts. 

- Simple Solution
	- set a key value only when the key is not existed.In case of the client crash,the lock on the key will not be removed,we add expire time on the key .the whole operation can be like the following.
		
	 ```
	 SET resource_name my_random_value NX PX 30000
	 ```
	- The above command should guarantee the **value should be unique among all the requests**,because there can be senarios,like :a client may acquire the lock, get blocked in some operation for longer than the lock validity time (the time at which the key will expire), and later remove the lock, that was already acquired by some other client.so the lock can't be just removed by the key. It should be done like this lua script:
		
		```lua
		if redis.call("get",KEYS[1]) == ARGV[1] then
    	return redis.call("del",KEYS[1])
		else
    		return 0
		end
		```

- More Complicated Solution
	- The Simple solution is not ok under the condition that:
		1. the redis node crashed,the slave hasn't synced from the master to get the lock.the above solution will fail.
	- The redis official site gives an algorithm called `The Redlock algorithm` to provide a more  robust distributed-lock feature.
	- the doc and implementations can be found here [the redlock algorithm](http://redis.io/topics/distlock).
	
- Pros and Cons on the Redlock algorithm
	what are the distributed locks for:
	
	> Efficiency: Taking a lock saves you from unnecessarily doing the same work twice (e.g. some expensive computation). If the lock fails and two nodes end up doing the same piece of work, the result is a minor increase in cost (you end up paying 5 cents more to AWS than you otherwise would have) or a minor inconvenience (e.g. a user ends up getting the same email notification twice).
	
	> Correctness: Taking a lock prevents concurrent processes from stepping on each others’ toes and messing up the state of your system. If the lock fails and two nodes concurrently work on the same piece of data, the result is a corrupted file, data loss, permanent inconsistency, the wrong dose of a drug administered to a patient, or some other serious problem. 
	
- `The redlock algorithm` solves the sigal node problems ,but brings up more problems which are existed natuarally  in the distributed system.But the redlock algorithm did not solve them all.

- conclusion
	- Redis may not be a good option for distributed lock if you want it for its correctness.but for efficiency ,it's good enough.

Check the total pros and cons from the below address.

> [How to do distributed locking](http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html).
	
> [Is Redlock Safe](http://antirez.com/news/101)