
     
## 系统介绍 Project Description
本系统是使用SpringBoot开发的高并发限时抢购秒杀系统，除了实现基本的登录、查看商品列表、秒杀、下单等功能，项目中还针对高并发情况实现了系统缓存、降级和限流。
This project is a high concurrency rush buy system that is based on Springboot. It has functions for login, browsing, rush buy, create order. It also implement system cache, downgrade and current limitor for high currency situations.

## 开发工具 Developing Tools
IntelliJ IDEA + Navicat + Sublime Text3 + Git + Chrome

## 压测工具 Testing Tool For High Concurrency
JMeter


## 开发技术 Developing Framework and Database
前端技术 Frontend ：Bootstrap + jQuery + Thymeleaf

后端技术 Backend ：SpringBoot + MyBatis + MySQL

中间件技术 Middleware : Druid + Redis + RabbitMQ + Guava

## 秒杀优化方向 Optimizations

* 1. 将请求尽量拦截在系统上游：传统秒杀系统之所以挂，请求都压倒了后端数据层，数据读写锁冲突严重，几乎所有请求都超时，流量虽大，下单成功的有效流量甚小，我们可以通过限流、降级等措施来最大化减少对数据库的访问，从而保护系统。
* 2. 充分利用缓存：秒杀商品是一个典型的读多写少的应用场景，充分利用缓存将大大提高并发量
* 1. Hold the request at upstream system: if we let our requests flow into backend databse, it will cause a serious read-write conflict, and most request will time-out. However, users who successfully get the promotion deals only take up very little portion of the entire users, instead of letting them all making request to backend database, we can put all this work to front end and letting those few users access backend to protect our system from overflow.
* 2. Make full use of cache: Online promotion sale make up of mainly reads and few writes, we will make full use of cache to increase concurrency flow

## 实现技术点 Techniques and Tips
### 1. 两次MD5加密 Twice MD5 Encryption

* 将用户输入的密码和固定Salt通过MD5加密生成第一次加密后的密码，再讲该密码和随机生成的Salt通过MD5进行第二次加密，最后将第二次加密后的密码和第一次的固定Salt存数据库
* First Encryption = MD5_Encryption(User Input Password + Static Salt)
* Second Encryption = MD5_Encryption(First Encryption + Randomly Generated Salt)
好处Benefits：    
     
* 1. 第一次作用：防止用户明文密码在网络进行传输
* 2. 第二次作用：防止数据库被盗，避免通过MD5反推出密码，双重保险
* 1. First Encryption: Hide Plaintext
* 2. Second Encryption: In case database is accessed by hackers, they cannot dycrypt with MD5 hash tables.

### 2. session共享 Shared Session
验证用户账号密码都正确情况下，通过UUID生成唯一id作为token，再将token作为key、用户信息作为value模拟session存储到redis，同时将token存储到cookie，保存登录状态
When username and password is verified, generate unique id as a token, then use token as key, then store user information into redis, and store token into cookie.
好处： 在分布式集群情况下，服务器间需要同步，定时同步各个服务器的session信息，会因为延迟到导致session不一致，使用redis把session数据集中存储起来，解决session不一致问题。
Benefits: In a distributed system, servers need to be synchronized. Save all session information with Redis to resolve lagging issues
### 3. JSR303自定义参数验证 Use JSR303 Bean Validation
使用JSR303自定义校验器，实现对用户账号、密码的验证，使得验证逻辑从业务代码中脱离出来。
Validate username and password, free validation from service code
### 4. 全局异常统一处理 Deal with errors in a global setting
通过拦截所有异常，对各种异常进行相应的处理，当遇到异常就逐层上抛，一直抛到最终由一个统一的、专门负责异常处理的地方处理，这有利于对异常的维护。
By intercepting all exceptions and handling them appropriately, when encountering an exception, throw it up layer by layer until it is finally handled by a unified, special place responsible for exception handling, which is beneficial for maintaining exceptions
### 5. 页面缓存 + 对象缓存 Page cache + object cache
* 1. 页面缓存：通过在手动渲染得到的html页面缓存到redis
* 2. 对象缓存：包括对用户信息、商品信息、订单信息和token等数据进行缓存，利用缓存来减少对数据库的访问，大大加快查询速度。
* 1. Page cache: Cache the html page obtained through manual rendering to redis
* 2. Object cache: Including caching user information, product information, order information, and token data, using cache to reduce access to the database and greatly speed up query speed.
### 6. 页面静态化 Page staticization
对商品详情和订单详情进行页面静态化处理，页面是存在html，动态数据是通过接口从服务端获取，实现前后端分离，静态页面无需连接数据库打开速度较动态页面会有明显提高
Process the product details and order details pages statically, the page is in html, and the dynamic data is obtained from the server through the interface, realizing the separation of front and back ends. The static page does not need to connect to the database and the opening speed is significantly faster than the dynamic page.

### 7. 本地标记 + redis预处理 + RabbitMQ异步下单 + 客户端轮询 Local marking + Redis preprocessing + RabbitMQ asynchronous ordering + client polling
* 描述：通过三级缓冲保护，1、本地标记  2、redis预处理  3、RabbitMQ异步下单，最后才会访问数据库，这样做是为了最大力度减少对数据库的访问。
* Description: Protect through three-level caching, 1, local marking 2, Redis preprocessing 3, RabbitMQ asynchronous ordering, and finally access the database, which is to minimize access to the database as much as possible.
实现：

* 1. 在秒杀阶段使用本地标记对用户秒杀过的商品做标记，若被标记过直接返回重复秒杀，未被标记才查询redis，通过本地标记来减少对redis的访问
* 2. 抢购开始前，将商品和库存数据同步到redis中，所有的抢购操作都在redis中进行处理，通过Redis预减少库存减少数据库访问
* 3. 为了保护系统不受高流量的冲击而导致系统崩溃的问题，使用RabbitMQ用异步队列处理下单，实际做了一层缓冲保护，做了一个窗口模型，窗口模型会实时的刷新用户秒杀的状态。
* 4. client端用js轮询一个接口，用来获取处理状态

Implementation:

* 1. During the rush buying stage, use local marking to mark products that users have rushed to buy. If they have been marked, return directly to repeat the rush buying, if not marked, query redis, and use local marking to reduce access to redis.
* 2. Before the rush buying starts, synchronize the product and inventory data to redis, and all rush buying operations are processed in redis. Reducing inventory through Redis reduces database access.
* 3. In order to protect the system from crashing due to high traffic, use RabbitMQ to process orders asynchronously with a queue, which actually provides a buffer protection and a window model. The window model will refresh the user's rush buying status in real time.
* 4. The client uses js to poll an interface to get the processing status.

### 8. 解决超卖
描述：比如某商品的库存为1，此时用户1和用户2并发购买该商品，用户1提交订单后该商品的库存被修改为0，而此时用户2并不知道的情况下提交订单，该商品的库存再次被修改为-1，这就是超卖现象
Solve overselling
Description: For example, if a product's inventory is 1, and at the same time user 1 and user 2 concurrently purchase the product, after user 1 submits the order, the product's inventory is modified to 0, and at this time, user 2 submits the order without knowing, and the product's inventory is modified to -1 again, this is the phenomenon of overselling.

实现：

1. 对库存更新时，先对库存判断，只有当库存大于0才能更新库存
2. 对用户id和商品id建立一个唯一索引，通过这种约束避免同一用户发同时两个请求秒杀到两件相同商品
3. 实现乐观锁，给商品信息表增加一个version字段，为每一条数据加上版本。每次更新的时候version+1，并且更新时候带上版本号，当提交前版本号等于更新前版本号，说明此时没有被其他线程影响到，正常更新，如果冲突了则不会进行提交更新。当库存是足够的情况下发生乐观锁冲突就进行一定次数的重试。
1. When updating inventory, first check the inventory, only when the inventory is greater than 0 can the inventory be updated.
2. Create a unique index for user id and product id to avoid a user sending two requests to rush to buy two identical products at the same time.
3. Implement optimistic locking, add a version field to the product information table, and add a version to each data. Every time you update, the version+1, and update with the version number. If the version number before submission is equal to the version number before update, it means that it has not been affected by other threads at this time, and it is updated normally. If there is a conflict, it will not be submitted for update. When there is enough inventory, an optimistic lock conflict occurs and a certain number of retries are made.

### 9. 使用数学公式验证码 Using math formula captcha
描述：点击秒杀前，先让用户输入数学公式验证码，验证正确才能进行秒杀。
Description: Before clicking on the rush buying, let the user input a math formula captcha first, and verify it correctly before proceeding with the rush buying.
好处：
1. 防止恶意的机器人和爬虫 
2. 分散用户的请求
Benefits:
1. Prevent malicious robots and crawlers
2. Disperse user requests

实现：
1. 前端通过把商品id作为参数调用服务端创建验证码接口
2. 服务端根据前端传过来的商品id和用户id生成验证码，并将商品id+用户id作为key，生成的验证码作为value存入redis，同时将生成的验证码输入图片写入imageIO让前端展示
3. 将用户输入的验证码与根据商品id+用户id从redis查询到的验证码对比，相同就返回验证成功，进入秒杀；不同或从redis查询的验证码为空都返回验证失败，刷新验证码重试
Implementation:
1. The front-end calls the server-side create captcha interface by passing the product id as a parameter.
2. The server-side generates a captcha based on the product id and user id passed from the front-end, and stores the product id + user id as the key and the generated captcha as the value in redis. At the same time, it writes the generated captcha into an image through imageIO for the front-end to display.
3. Compare the user's input captcha with the captcha queried from redis based on the product id + user id. If they are the same, return the verification success and enter the rush buying; if they are different or the captcha queried from redis is empty, return the verification failure and refresh the captcha to try again.
### 10. 使用RateLimiter实现限流 Using RateLimiter to implement rate limiting
描述：当我们去秒杀一些商品时，此时可能会因为访问量太大而导致系统崩溃，此时要使用限流来进行限制访问量，当达到限流阀值，后续请求会被降级；降级后的处理方案可以是：返回排队页面（高峰期访问太频繁，等一会重试）、错误页等。
Description: When we rush to buy some products, the system may crash due to too much traffic. At this time, we need to use rate limiting to restrict the traffic volume. When the rate limiting threshold is reached, subsequent requests will be downgraded. The processing plan for the downgrade can be: returning to the queue page (too frequent access during peak hours, wait a moment and try again), error page, etc.

实现：项目使用RateLimiter来实现限流，RateLimiter是guava提供的基于令牌桶算法的限流实现类，通过调整生成token的速率来限制用户频繁访问秒杀页面，从而达到防止超大流量冲垮系统。（令牌桶算法的原理是系统会以一个恒定的速度往桶里放入令牌，而如果请求需要被处理，则需要先从桶里获取一个令牌，当桶里没有令牌可取时，则拒绝服务）

Implementation: The project uses RateLimiter to implement rate limiting. RateLimiter is a rate limiting implementation class provided by guava based on the token bucket algorithm. By adjusting the rate of generating tokens, it limits the user's frequent access to the rush buying page, thereby preventing the system from crashing due to excessive traffic. (The principle of the token bucket algorithm is that the system puts tokens into the bucket at a constant rate. If a request needs to be processed, it needs to get a token from the bucket first. If there are no tokens available in the bucket, the service is refused.


## 压测效果 Testing for High Concurrency
Presentation video: start at 6:55 [link](https://www.youtube.com/watch?v=mI_jZkse2JI)

-----

本项目是学习了imooc网视频之后的个人理解和知识汇总，[学习链接](https://coding.imooc.com/class/168.html)
This project is my personal understanding and summary of knowledge after learning from imooc's video[link](https://coding.imooc.com/class/168.html)
