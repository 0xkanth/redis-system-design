# redis-system-design

## Fixed-Bucket RateLimiter


```typescript
    // FixedBucket Rate Limiter
    
    /**
     * In the "out of the box" Redis implementation, the request count is incremented using the atomic INCR command. 
        ## Where the Increment Happens
         - When a request arrives, your application sends an INCR command for a specific user key (e.g., rate_limit:user_123).
         - Initialization: If the key doesn't exist, Redis creates it with a value of 0 and then increments it to 1.
         - Atomicity: INCR is atomic, meaning even if multiple requests arrive at the same millisecond, Redis will increment the count sequentially without race conditions.
         - Expiration: To reset the window, an EXPIRE command is usually sent alongside INCR (often inside a MULTI/EXEC block or Lua script) to ensure the key is deleted after your defined time window. 
     */
    import { Request, Response, NextFunction } from 'expressjs';
    import { createClient } from 'redis';
    
    const redisClient = createClient();
    await redisClient.connect();
    
    const REDIS_LUA_SCRIPT = `
        local current = redis.call('INCR', KEYS[1])
        if current == 1 then
         redis.call("EXPIRE", KEYS[1], ARGS[1])
        end
        return current
    `;
    
    export const middleware = async (req: Request, resp: Response, next: NextFunction) => {
        const user = req.headers['user-id'] || req.ip;
        const key = `rate_limit:${user}`;
        const limit = 10;
        // 1 hour
        const expiryWindowInSeconds = 60 * 60;
        
        try {
            const requestCount = redisClient.eval(REDIS_LUA_SCRIPT, {
                keys : [key],
                arguments: [expiryWindowInSeconds.toString()]
            });
            
            if(requestCount > limit) {
               return resp.status(429).json({"error": "Too Many Requests"});
            }
             
            next();
        } catch(err) {
            console.error(`Failed to process as user reached the request limit: ${limit}`);
            next();
        }
    };
```

