# Caching (@nestjs/cache-manager)

- Install `@nestjs/cache-manager cache-manager`. Stores are Keyv instances, in memory by default. https://docs.nestjs.com/data/caching#installation
- TTL is in milliseconds. No `ttl` means entries never expire. https://docs.nestjs.com/data/caching#time-to-live-ttl
  ```typescript
  CacheModule.register({ isGlobal: true, ttl: 5000 })
  await this.cacheManager.set('key', 'value', 1000);
  const value = await this.cacheManager.get('key'); // undefined on a miss
  ```
- `get()` returns `undefined` on a miss. `cache-manager` v6 returned `null`: treat both as a miss when migrating. https://docs.nestjs.com/data/caching#interacting-with-the-cache-store
- The in-memory store returns the same object reference you stored: mutating it mutates the cache. Keyv stores serialize to JSON, so class instances come back as plain objects and `Date` as strings. https://docs.nestjs.com/data/caching#interacting-with-the-cache-store
- Several stores: `stores: [...]`, the first is the default and the rest are fallbacks. https://docs.nestjs.com/data/caching#using-alternative-cache-stores
  ```typescript
  const redisStore = createKeyv('redis://localhost:6379');
  redisStore.on('error', (err) => console.error('Redis error:', err));
  return { stores: [new Keyv({ store: new KeyvCacheableMemory({ ttl: 60000, lruSize: 5000 }) }), redisStore] };
  ```
- Always attach an `'error'` listener to the Redis store: Keyv throws when nothing listens and the app can crash on a dropped connection. https://docs.nestjs.com/data/caching#using-alternative-cache-stores
- `CacheInterceptor` caches only `GET` routes by default (an explicit `@CacheKey()` bypasses that check) and cannot cache routes using `@Res()`. The key is the request URL. https://docs.nestjs.com/data/caching#auto-caching-responses
- To key by something else (the `Authorization` header), subclass `CacheInterceptor` and override `trackBy()`. https://docs.nestjs.com/data/caching#adjust-tracking
- A URL-keyed cache answers before the handler, so on routes that check records per user it serves one user's data to another. Cache below the check or per user. https://docs.nestjs.com/security/authorization#production-checklist
- `CacheInterceptor` does not work properly with GraphQL (interceptors run per field). https://docs.nestjs.com/data/caching#auto-caching-responses
- In WebSocket and microservice handlers `@CacheKey()` is required, otherwise nothing is cached. https://docs.nestjs.com/data/caching#websockets-and-microservices
- Method `@CacheTTL()` wins over controller `@CacheTTL()`. https://docs.nestjs.com/data/caching#global-cache-overrides
- Never cache actions that perform business operations rather than only querying data. https://docs.nestjs.com/data/caching#websockets-and-microservices
- An idempotency interceptor must stay outside other global interceptors such as a cache. https://docs.nestjs.com/reliability/idempotency#production-checklist
