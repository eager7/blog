# Redis缓存实现

\[TOC\]

## 缓存穿透

请求去查询一条数据库中不存在的数据，也就是缓存和数据库都查询不到这条数据，那么请求每次都会打到数据库上面去。这种查询不存在数据的现象我们称为缓存穿透。 解决办法为缓存失败的请求，但是不能缓存太长时间，否则在程序有bug返回错误时会无法及时恢复，影响正常业务

## 缓存击穿

在高并发的系统中，大量的请求同时查询一个 key，如果此时这个key正好失效了，就会导致大量的请求都打到数据库上面去。这种现象我们称为缓存击穿。 解决办法是在相同请求上加锁，如果有多个相同请求只能让一个请求落到DB，其余请求需要等待此请求返回，然后共用数据。或者我们保留旧数据备份在缓存，让其余接口去拿旧数据返回。

## 缓存雪崩

缓存雪崩是指在我们设置缓存时采用了相同的过期时间，导致缓存在某一时刻同时失效，请求全部转发到DB，DB瞬时压力过重雪崩。 解决办法是设置随机过期时间，避免同时过期。

## 热点数据失效

和缓存雪崩类似，几个热点数据同时过期导致DB压力激增。 解决办法也是设置随机过期时间即可。

## 代码实现

下面是go代码实现\(未处理cookie，请根据需要自行处理\)：

```text
const LOCK, RequestCacheTime = "perm_lock:", "request_cache_time:"
var code proto.Message


start, key, cacheTime, isFromDb := time.Now(), "", time.Second * 50, true     //请求key以及单个接口缓存时间，默认50s


if !c.enable {
   return handler(ctx, req) //调用请求
}
log.Notice("redis key:", key, cacheTime)


/*
** 使用single模式解决单机缓存击穿问题，同一时刻只能有一个请求透过缓存层下放，其余都会在此等待，share为true表示和其他请求共用了一个返回值
 */
sRet, err, share := c.single.Do(LOCK+key, func() (i interface{}, err error) {
   /*
   ** 单机透过还需要检测多机部署情况，使用redis避免多机部署下缓存雪崩问题，这里主要是避免某些极为耗时的数据库操作重复透到DB
    */
   defer c.cRedis.Del(LOCK + key) //退出时自动解除锁定
   /*
   ** 锁定请求接口，保证多机部署下也只有一个请求可以下到DB层,
   ** 锁定一分钟，如果程序异常宕掉，一分钟后可以自动释放分布式锁，
   ** 保证缓存机制生效，但是如果请求很耗时，经常超出这个值，则需要考虑将此时间加长
    */
   ret := c.cRedis.SetNX(LOCK+key, "lock-key", time.Minute*1)
   if err := ret.Err(); err != nil { //设置失败，通常是redis异常，此时走DB保证业务
      log.Warn("redis set failed:", err)
      isFromDb = true
      return handler(ctx, req)
   }
   if !ret.Val() { //此分布式锁已经被锁定了，表示有相同请求在前面还未处理完成，发生了多机缓存击穿，在这里使用旧数据解决缓存击穿问题
      if err := GetRedisContent(c.cRedis, code, key); err == nil { //从缓存取数据
         isFromDb = false
         return code, nil
      } else {
         log.Warn("redis key is already lock, get content failed:", err)
      }
      resp, err := handler(ctx, req) //程序刚刚启动时无法解决缓存击穿问题，这里下放到DB
      isFromDb = true
      return resp, err
   }


   /*
   ** 接口没有被锁定，那么执行过期检测，如果过期，更新缓存，没有过期，返回缓存
    */
   if c.cRedis.Get(RequestCacheTime+key).Err() == nil { //获取更新key来判断是否需要更新缓存内容
      if err := GetRedisContent(c.cRedis, code, key); err == nil { //从缓存取数据，取不到，则有可能缓存刚刚被清空，走DB
         isFromDb = false
         return code, nil
      } else {
         log.Warn("redis key is not lock and update key not expired, get content failed:", err)
      }
   }


   /*
   ** 接口未锁定，且缓存过期或数据被清空，则下放接口到数据库
    */
   resp, err := handler(ctx, req) //调用请求，返回数据


   /*
   ** 我们这里不判断返回情况，对错误请求以及不存在的数据也返回正常进行缓存，从而可以解决缓存穿透问题，
   ** 即如果从存储层查不到数据则不写入缓存，这将导致这个不存在的数据每次请求都要到存储层去查询，失去了缓存的意义。
    */
   if rc, ok := resp.(proto.Message); ok {
      data, err := proto.Marshal(rc)
      if err != nil {
         log.Warn("marshal data failed:", err)
         return resp, nil
      }
      /*
      ** 这里对数据和缓存时间进行拆分，可以解决缓存击穿问题，即缓存在某个时间点过期的时候，恰好在这个时间点对这个Key有大量的并发请求过来，
      ** 会命中上面锁定机制，走缓存拿数据，如果此时没有数据，就又会落到DB，将数据设置的比缓存时间长，可以让锁定请求拿到旧数据，不至于全部落到DB
      ** 同时随机设置过期时间值，防止同一时刻大量key失效引起的缓存雪崩问题
       */
      if err := c.cRedis.Set(key, hex.EncodeToString(data), time.Hour*(5+time.Duration(rand.Int31n(24)))).Err(); err != nil { 
         log.Warn("The cache interface failed err：", key, err)
         return resp, nil
      }
      if err := c.cRedis.Set(RequestCacheTime+key, "update-key", cacheTime).Err(); err != nil { //设置更新key的缓存，当此key失效表示需要更新数据，但是还有一份旧数据保留在redis中，正常放回后会覆盖掉
         log.Warn("Cache lock time and cache time failed err：", RequestCacheTime+key, err)
         return resp, nil
      }
   } else {
      log.Warn("resp type:", reflect.TypeOf(resp))
   }
   isFromDb = true
   return resp,nil
})
log.Notice("is from db:", isFromDb, "is share:", share, time.Since(start))


return sRet, err
```

