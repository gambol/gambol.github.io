---
layout:     post
title:      "流量控制"
subtitle:   "traffic的流量控制"
date:       2016-01-03 12:00:00
author:     "gambol"
header-img: "img/post-bg-2015.jpg"
tags:
    - 技术
---


# 流量控制

标签（空格分隔）： 技术 流量控制 限速

---

# 作用
流量控制的作用那可就多了, 实用场景非常多. 譬如 某个接口抗不了太大的压力,最好增加一个限制, 譬如qps不超过10. 

# 实现
## 简单实现
最简单的做法是维护一个单位时间内的Counter，如判断单位时间已经过去，则将Counter重置零。 这种方法的问题是, 不能在连续的时间里,保证流量控制. 因此出现了漏斗的算法

## bucket算法
有两种常见的bucket算法. 第一种算法是
### Leaky Bucket
思想是认为有一个会漏水的桶，水以恒定速率滴出，上方会有水滴（请求）进入水桶。显然，如果上方水滴进入速率超过水滴出的速率，那么水桶就会溢出，这里的溢出就是traffic shaping和traffic policing的条件，即执行某个过载任务的时候。

可以参考 wiki上的这个链接. https://en.wikipedia.org/wiki/Leaky_bucket

引用![leaky bucket][1]

java 代码实现请看

     /**
     * 水溜走的速度(相当于允许最大的速率)
     */
    long rate;

    /**
     * 桶子的大小
     */
    long bucketSize;

    /**
     * 上次运行的时间
     */
    long lastCalcTime;

    /**
     * 可以接纳的容量
     */
    long water;

    public LeakyBucketLimiter(long bucketSize, long rate) {
        this.rate = rate;
        this.bucketSize = bucketSize;
    }

    synchronized boolean take() {
        long now = System.currentTimeMillis();
        water = Math.max(water - (now - lastCalcTime) * rate / 1000, 0);
        lastCalcTime = now;

        if (water < bucketSize) {
            water++;
            return true;
        }

        return false;
    }

    public static void main(String[] args) throws Exception {
        LeakyBucketLimiter limiter = new LeakyBucketLimiter(5, 5);

        for(int i = 0; i < 20; i++) {
            boolean result = limiter.take();
            logger.info("runtime :{},  result:{}", i, result);
        }

        Thread.sleep(1000);

        for(int i = 0; i < 20; i++) {
            boolean result = limiter.take();
            logger.info("second time runtime :{},  result:{}", i, result);
        }
    }

## token bucket
token bucket 和 leaky bucket 基本相同, 思路更加符合人的想法.
同样有一个桶，令牌以恒定速率放入桶，桶内的令牌数有上限，每个请求会acquire一个令牌，如果某个请求来到而桶内没有令牌了，请说明这个请求是过载的。和Lecky Bucket不同的是，Token Bucket存在burst rate。比如当前令牌放入速率4个每秒，桶的令牌上限是8，第一秒内没有请求，第二秒实际就可以处理8个请求！虽然平均速率还是4个每秒，但是爆发速率是8个每秒。

用图片表示, 思路如下:
![token bucket][2]

wiki地址是
https://en.wikipedia.org/wiki/Token_bucket

代码也比较简单, 请看

    /**
     * 最大允许的token数量
     */
    double maxAllowedToken;

    /**
     * 现有的空闲token数量
     */
    double availableToken;

    /**
     * 下一个可以使用token的时间
     */
    long lastCalcTime;

    /**
     * 每秒增加的token数量
     */
    double tokenPerSecond;


    public TokenBucketLimiter(double maxAllowedToken, double tokenPerSecond) {
        this.maxAllowedToken = maxAllowedToken;
        this.tokenPerSecond = tokenPerSecond;
    }


    synchronized boolean take() {
        long now = System.currentTimeMillis();
        availableToken = Math.min(availableToken +  (now - lastCalcTime) * tokenPerSecond / 1000.0, maxAllowedToken);
        lastCalcTime = now;

        if (availableToken >= 1) {
            availableToken --;
            return true;
        }

        return false;
    }
    
## token bucket 与 leaky bucket比较
Leacky Bucket算法默认一开始水桶是空的，可以立即就接收最多burst的请求，而Token Bucket就要设置初始Token的数量。
其他方面,这两个差不多.
    
## RateLimiter
google guava有一个token bucket的实现, 他在这个基础上,增加了很多工业上的优化. 源码解释可以参考 [guava ratelimiter 源码解释][3]

## 分布式的rateLimiter实现
基于redis的基础上, 我写了一个分布式环境下的rate limiter, 利用redis, 简单写了一个基于token bucket的算法. 这个代码代码有一个缺陷, 使用了redis的事务. 但是redis的事务有一些缺陷.并且还有一些代码没有写完整, 将来再补充.

     private final Object mutex = new Object();

    public static RateRestrainer createRedisRestrainer(double rate, String redisKey, JedisPool jedisPool) {
        return new RedisTokenBucket(rate, redisKey, jedisPool);
    }

    /**
     * 返回现在限制的最大速度
     *
     * @return
     */
    public final double getRate() {
        // TODO 增加实现
        return 0.0;
    }

    /**
     * 获取执行这条记录的permit。 如果获取不到，一直睡眠，直到获取成功
     */
    public double acquire() {
        long stepIntoTime = Ticker.read();
        ReserveResult reserveResult;

        do {
            synchronized (mutex) {
                reserveResult = reserveTicket();
            }
            Ticker.sleep(reserveResult.getMillsToWait());
        } while (reserveResult.continueWait);

        long stepOutTime = Ticker.read();
        return 1.0 * (stepOutTime - stepIntoTime);
    }

    // TODO
    public double tryAcquire() {
        return 0.0;
    }

    // 尝试去获取一个许可。
    abstract ReserveResult reserveTicket();

    private static class RedisTokenBucket extends RateRestrainer {

        /**
         * 最大允许的token数量
         */
        double maxAllowedToken;

        /**
         * 现有的空闲token数量
         */
        double availableToken;

        /**
         * 每秒增加的token数量
         */
        double tokenPerSecond;

        /**
         * jedis的pool
         */
        private JedisPool pool;

        /**
         * jedis template。封装了一些jedis的操作
         */
        private JedisTemplate jedisTemplate;

        /**
         * redis 要用到的key的prefix
         */
        private String redisKey;

        final static String AVAILABLE_TOKEN = "availableToken";
        final static String LAST_REFRESH_TIME = "lastRefreshTime";

        public RedisTokenBucket(double rate, String redisKey, JedisPool jedisPool) {
            tokenPerSecond = rate;
            this.redisKey = redisKey;
            pool = jedisPool;
            maxAllowedToken = rate;
            availableToken = 0;
            jedisTemplate = new JedisTemplate(jedisPool);
            clearRedis();
        }

        @Override
        ReserveResult reserveTicket() {
            final Jedis jedis = pool.getResource();
            jedis.watch(redisKey);

            double availableTokenInRedis = getDoubleFromRedis(AVAILABLE_TOKEN);
            long lastRefreshTimeInRedis = getLongFromRedis(LAST_REFRESH_TIME);
            long timeInRedisMills = fetchRedisTimeMills();

            boolean flag = false; // 运行结果
            try {
                final Transaction transaction = jedis.multi();

                availableToken = Math.min(availableTokenInRedis + (timeInRedisMills - lastRefreshTimeInRedis)
                        * tokenPerSecond / TimeUnit.SECONDS.toMillis(1L), maxAllowedToken);
                transaction.hset(redisKey, LAST_REFRESH_TIME, String.valueOf(timeInRedisMills));

                if (availableToken >= 1) {
                    availableToken--;
                    flag = true;
                }

                transaction.hset(redisKey, AVAILABLE_TOKEN, String.valueOf(availableToken));
                List<Object> resultList = transaction.exec();

                if (CollectionUtils.isEmpty(resultList)) {
                    flag = false; // redis提交失败， 说明别的进程/线程在这次提交之前先运行了。这次申请失败
                }
            } finally {
                pool.returnResource(jedis);
            }

            if (!flag) {
                long sleepFor = Math.round(TimeUnit.SECONDS.toMillis(1L) * (1.0 - availableToken) / tokenPerSecond);
                return ReserveResult.createWaitResult(sleepFor);
            }
            return ReserveResult.createContinueResult();
        }

        /**
         * 清掉redis里的这个key对应的值
         */
        void clearRedis() {
            jedisTemplate.del(redisKey);
        }

        /**
         * 获取redis所在机器的系统时间。 单位是ms
         *
         * @return 时间
         */
        long fetchRedisTimeMills() {
            /**
             * 给默认值
             */
            long timeInRedis = Ticker.read();
            try {
                List<String> timeList = jedisTemplate.execute((Jedis jedis) -> jedis.time());
                timeInRedis = Long.parseLong(timeList.get(0)) * 1000 + Long.parseLong(timeList.get(1)) / 1000;
            } catch (NullPointerException | NumberFormatException | IndexOutOfBoundsException e) {
                logger.error("error in parse time. use system time", e);
            }
            return timeInRedis;
        }

        /**
         * 从redis 的hash 里 取出相应field的值，转换成double。 如果转换失败（譬如为null，或者这个值不为double），则返回0
         *
         * @return double值
         */
        double getDoubleFromRedis(String field) {
            double ret = 0.0;
            String v = jedisTemplate.hget(redisKey, field);
            try {
                ret = v != null ? Double.parseDouble(v) : 0.0;
            } catch (NumberFormatException | NullPointerException e) {
                logger.warn("error in parseDouble.key:{}, field:{}", redisKey, field, e);
            }
            return ret;
        }

        long getLongFromRedis(String field) {
            long ret = 0;
            String v = jedisTemplate.hget(redisKey, field);
            try {
                ret = v != null ? Long.parseLong(v) : 0;
            } catch (NumberFormatException e) {
                logger.warn("error in parseLong.key:{}, field:{}", redisKey, field, e);
            }

            return ret;
        }
    }

    @Getter
    @Setter
    @AllArgsConstructor
    final static class ReserveResult {
        /**
         * 这次睡眠的时间，单位为 毫秒
         */
        long millsToWait;

        /**
         * 是否需要继续等待
         */
        boolean continueWait;

        /**
         * 预定成功，继续运行
         */
        final static ReserveResult CONTINUE_RESULT = new ReserveResult(0, false);

        /**
         * 创建一个需要等待的对象
         *
         * @param millsToWait
         * @return
         */
        static ReserveResult createWaitResult(long millsToWait) {
            return new ReserveResult(millsToWait, true);
        }

        /**
         * 不需要等待
         *
         * @return
         */
        static ReserveResult createContinueResult() {
            return CONTINUE_RESULT;
        }
    }

    /**
     * 计时器.
     */
    final static class Ticker {

        /**
         * @return 获取当前时间戳。 单位是 ms
         */
        static long read() {
            return System.currentTimeMillis();
        }

        static void sleep(long millsToSleep) {
            if (millsToSleep > 0) {
                Uninterruptibles.sleepUninterruptibly(millsToSleep, TimeUnit.MILLISECONDS);
            }
        }
    }





  [1]: https://upload.wikimedia.org/wikipedia/commons/4/4c/Leaky_bucket_as_a_meter-policing.JPG
  [2]: http://ww4.sinaimg.cn/large/56d8760agw1f04rnswvz8j20bp06pwek.jpg
  [3]: http://xiaobaoqiu.github.io/blog/2015/07/02/ratelimiter/
