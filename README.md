
     
## Project Description
This project is a high concurrency rush buy system that is based on Springboot. It has functions for login, browsing, rush buy, create order. It also implement system cache, downgrade and current limitor for high currency situations.

## Developing Tools
IntelliJ IDEA + Navicat + Sublime Text3 + Git + Chrome

## Testing Tool For High Concurrency
JMeter


## Developing Framework and Database
Frontend ：Bootstrap + jQuery + Thymeleaf

Backend ：SpringBoot + MyBatis + MySQL

Middleware : Druid + Redis + RabbitMQ + Guava

## Optimizations

* 1. Hold the request at upstream system: if we let our requests flow into backend databse, it will cause a serious read-write conflict, and most request will time-out. However, users who successfully get the promotion deals only take up very little portion of the entire users, instead of letting them all making request to backend database, we can put all this work to front end and letting those few users access backend to protect our system from overflow.
* 2. Make full use of cache: Online promotion sale make up of mainly reads and few writes, we will make full use of cache to increase concurrency flow

## Techniques and Tips
### Twice MD5 Encryption

* 
* First Encryption = MD5_Encryption(User Input Password + Static Salt)
* Second Encryption = MD5_Encryption(First Encryption + Randomly Generated Salt)
Benefits：    
 
* 1. First Encryption: Hide Plaintext
* 2. Second Encryption: In case database is accessed by hackers, they cannot dycrypt with MD5 hash tables.

### 2. Shared Session

When username and password is verified, generate unique id as a token, then use token as key, then store user information into redis, and store token into cookie.
Benefits: In a distributed system, servers need to be synchronized. Save all session information with Redis to resolve lagging issues
### 3. Use JSR303 Bean Validation
Validate username and password, free validation from service code

### 4. Local marking + Redis preprocessing + RabbitMQ asynchronous ordering + client polling
* Description: Protect through three-level caching, 1, local marking 2, Redis preprocessing 3, RabbitMQ asynchronous ordering, and finally access the database, which is to minimize access to the database as much as possible.

Implementation:

* 1. During the rush buying stage, use local marking to mark products that users have rushed to buy. If they have been marked, return directly to repeat the rush buying, if not marked, query redis, and use local marking to reduce access to redis.
* 2. Before the rush buying starts, synchronize the product and inventory data to redis, and all rush buying operations are processed in redis. Reducing inventory through Redis reduces database access.
* 3. In order to protect the system from crashing due to high traffic, use RabbitMQ to process orders asynchronously with a queue, which actually provides a buffer protection and a window model. The window model will refresh the user's rush buying status in real time.
* 4. The client uses js to poll an interface to get the processing status.

### 5. Using RateLimiter to implement rate limiting
Description: When we rush to buy some products, the system may crash due to too much traffic. At this time, we need to use rate limiting to restrict the traffic volume. When the rate limiting threshold is reached, subsequent requests will be downgraded. The processing plan for the downgrade can be: returning to the queue page (too frequent access during peak hours, wait a moment and try again), error page, etc.

Implementation: The project uses RateLimiter to implement rate limiting. RateLimiter is a rate limiting implementation class provided by guava based on the token bucket algorithm. By adjusting the rate of generating tokens, it limits the user's frequent access to the rush buying page, thereby preventing the system from crashing due to excessive traffic. (The principle of the token bucket algorithm is that the system puts tokens into the bucket at a constant rate. If a request needs to be processed, it needs to get a token from the bucket first. If there are no tokens available in the bucket, the service is refused.


## Testing for High Concurrency
Presentation video: start at 6:55 [link](https://www.youtube.com/watch?v=mI_jZkse2JI)

-----
