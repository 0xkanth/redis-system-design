# redis-rate-limiter-system-design

## References:
- https://redis.io/glossary/rate-limiting/#Types_of_rate_limiting
- https://medium.com/redis-with-raphael-de-lio/sliding-window-counter-rate-limiter-redis-java-1ba8901c02e5

## Fixed-Bucket RateLimiter

### Register and Use

1. Register the Function Library
   You only need to run this command once (e.g., via redis-cli or a setup script) to load the library into Redis 

```lua
-- Registering a library named 'rate_limiter' with a function 'check_limit'
FUNCTION LOAD "#!lua name=rate_limiter\n 
redis.register_function('check_limit', function(keys, args)
    local current = redis.call('INCR', keys[1])
    if current == 1 then
        redis.call('EXPIRE', keys[1], args[1])
    end
    return current
end)"
```

```typescript
import express, { Request, Response, NextFunction } from 'express';
import { createClient } from 'redis';

const app = express();
const redisClient = createClient();
redisClient.connect().catch(console.error);

export const redisFunctionLimiter = async (req: Request, res: Response, next: NextFunction) => {
    // 1. Normalize key (handling casing)
    const userId = (req.headers['x-user-id'] as string || req.ip || 'anonymous').toLowerCase();
    const key = `ratelimit:${userId}`;
    
    const limit = 10;
    const windowSeconds = 60;

    try {
        // 2. Call the pre-registered Redis Function
        // Syntax: FCALL <function_name> <num_keys> <key> <args>
        const currentCount = await redisClient.fCall(
            'check_limit', 
            [key], 
            [windowSeconds.toString()]
        ) as number;

        if (currentCount > limit) {
            return res.status(429).json({ 
                error: "Too many requests", 
                retryAfter: windowSeconds 
            });
        }
        
        next();
    } catch (err) {
        console.error("Redis Function Error:", err);
        next(); // Fail open for reliability
    }
};

app.use(redisFunctionLimiter);
```


### Direct Use with backend

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

