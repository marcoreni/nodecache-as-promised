# @nrk/nodecache-as-promised

> Fast and resilient cache for NodeJs targeting high-volume sites

- [Installing](#installing)
- [Publish](#publish)
- [Features](#features)
- [APIs](#apis)
- [Examples](#examples)
- [Middlewares](#middlewares)
- [Creating your own middlewares](#creating-your-own-middleware)
- [Local development](#local-development)
- [Building and committing](#building-and-committing)

## Installing

```
npm install @nrk/nodecache-as-promised --save
```


## Publish

```
npm install 
npm login 

# one of
npm version patch -m 'Release patch %s'
npm version minor -m 'Release minor %s'
npm version major -m 'Release major %s'

npm run build
git push
npm publish

```

## Motivation
Sometimes Node.js needs to do some heavy lifting, performing CPU or network intensive tasks and yet respond quickly on incoming requests. For repetitive tasks like Server side rendering of markup or parsing big JSON responses caching can give the application a great performance boost. Since many requests may hit the server concurrently, you do not want more than **one** worker to run for a given resource at the same time. In addition - serving stale content when a backend resource is down may save your day! The intention of `nodecache-as-promised` is to give you a fairly simple interface, yet powerful application cache, with fine-grained control over caching behaviour.

`nodecache-as-promised` is inspired by how [Varnish](https://varnish-cache.org/) works. It is not intended to replace Varnish (but works great in combination). Whereas Varnish is a high-performant edge/burst/failover cache, working as a reverse proxy and loadbalancer, it depends on a fast backend when configured with short a cache window (ie. TTL ~1s). It uses URLs in combination with cookies as keys for its cached content. Since there are no restrictions on conformant URLs/cookies for clients requesting content, it is quite easy to bust it's cache without any security measures. `nodecache-as-promised` on the other hand is running at application level for more strict handling of cache keys, and may use many different caches and policies on how the web page is built.

### Features
- __In-memory cache__ is used as primary storage since it will always be faster than parsing and fetching data from disk or via network. An [LRU-cache](https://www.npmjs.com/package/lru-cache) is enabled to constrain the amount of memory used.
- __Caches are filled using worker promises__ since cached objects often are depending on async operations. [RxJs](https://www.npmjs.com/package/rxjs) is used to queue concurrent requests for the same key; thus ensuring that only __one__ worker is performed when cached content is missing/stale.
- __Caching of custom class instances, functions and native objects__ such as Date, RegExp and Redux stores are supported through in-memory caching. Non-serializable (using JSON.stringify) objects are filtered out in persistent caches though.
- __Grace mode__ is used if a worker fails (eg. caused by failing backends), ie.  stale cache is returned instead.
- __Avoidance of spamming backend resources__ using a configurable deltaWait parameter, serving either a stale object or a rejection.
- __Middleware support__ so you may create your own custom extensions. Provided middlewares:
  - __Persistent cache__ is used as secondary storage to avoid high back-pressure when inMemoryCaches are cleared after server restarts. This is achieved storing cache-misses and deletions on cache evictions using a [ioredis](https://www.npmjs.com/package/ioredis)-factory connecting to a redis instance.
  - __Distributed on demand expiry__ so that new content may be published across servers/instances before cache-TTL is reached. This is achieved using Redis pub/sub depending on a [ioredis](https://www.npmjs.com/package/ioredis)-factory

### Performance testing

Parsing a json-file at around 47kb (file contents are cached at startup). Using a Macbook pro, mid 2015, 16gb ram, i7 CPU.

<p align="left">
  <img src="./test/linear-perftest-nocache.jpeg?raw=true" width="50%"/>
</p>

The image shows a graph from running the test script `npm run perf:nocache-cache-file -- --type=linear`. At around 1300 iterations the event loop starts lagging, and at around 1500 iterations the process stops responding. It displays that even natively optimized JSON.parse could be a bottleneck when fetching remote API-data for rendring. (`React.render` would be even slower)

<p align="left">
  <img src="./test/linear-perftest-cache.jpeg?raw=true" width="50%"/>
</p>

The second image is a graph from running test script `npm run perf:cache -- --type=linear`. At around 3.1 million iterations the event loop starts lagging, and at around 3.4 million iterations the process runs out of memory and crashes. The graph has no relation to how fast JSON.parse is, but what speed is achievable by skipping it altogether (ie. `Promise`-processing)

## APIs
Create a new `inMemoryCache` instance using a factory method. This instance may be extended by the `distCache` and/or `persistentCache` middlewares (`.use(..)`).

### inMemoryCache factory
Creating a new instance

```js
import inMemoryCache from '@nrk/nodecache-as-promised'
const cache = inMemoryCache(options)
```

#### options
An object containing configuration
- initial - `Object`. Initial key/value set to prefill cache. Default: `{}`
- maxLength - `Number`. Max key count before LRU-cache evicts object. Default: `1000`
- maxAge - `Number`. Max time before a (stale) key is evicted by LRU-cache (in ms). Default: `172800000` (48h)
- log - `Object with log4j-facade`. Used to log internal work. Default: `console`

### Instance methods
When the factory is created (with or without middlewares), the following methods may be used.

#### .get(key, [options])
Get an item from the cache.
```js
const {value} = cache.get('myKey')
console.log(value)
```

Using parameter `options` - the function either fetches a value from cache or executes provided worker if the cache is stale or cold. The worker will set the cache key if ran and thus returns a Promise

```js
cache.get('myKey', options)
  .then(({value}) => {
    console.log(value)
  })
```
#### options
Configuration for the newly created object
- worker - `function`. A function that returns a promise which resolves new value to be set in cache.
- ttl - `Number`. Ttl (in ms) before cached object becomes stale. Default: `86400000` (24h)
- workerTimeout - `Number`. max time allowed to run promise. Default: `5000`
- deltaWait - `Number`. delta wait (in ms) before retrying promise, when stale. Default: `10000`

#### returned object
- value - `any` - value set in cache
- created - `Number` - UX timestamp (ms) when the value was created
- cache - `Enum(hit|miss|stale)` - status of cached content
- TTL - `Number` - Amount of ms until until value becomes stale since creation

**NOTE:** It might seem a bit strange to set cache values using `.get` - but it is to avoid a series of operations using `.get()` to check if a value exists, then call `.set()`, and finally running `.get()` once more (making queing difficult). In summary: `.get()` returns a value from cache or a provided worker.

#### .set(key, value, [ttl])
Set a new cache value.
```js
// set a cache value that becomes stale after 1 minute
cache.set('myKey', 'someData', 60 * 1000)
```

If `ttl`-parameter is omitted, a default will be used: `86400000` (24h)


#### .has(key)
Check if a key is in the cache, without updating the recent-ness or deleting it for being stale.

#### .del(key)
Deletes a key out of the cache.

#### .expire(keys)
Mark keys as stale (ie. set TTL = 0)
```js
cache.expire(['myKey*', 'anotherKey'])
```

Asterisk `*` is used for wildcards

#### .keys()
Get all keys as an array of strings stored in cache

#### .values()
Get all values as an array of all values in cache

#### .entries()
Get all entries as a Map of all keys and values in cache

#### .clear()
Clear the cache entirely, throwing away all values.

#### .addDisposer(callback)
Add callback to be called when an item is evicted by LRU-cache. Used to do cleanup
```js
const cb = (key, value) => cleanup(key, value)
cache.addDisposer(cb)
```

#### .removeDisposer(callback)
Remove callback attached to LRU-cache
```js
cache.removeDisposer(cb)
```

#### .debug([extraData])
Prints debug information about current cache (ie. hot keys, stale keys, keys in waiting state etc). Use `extraData` to add custom properties to the debug info, eg. hostname.
```js
cache.debug({hostname: os.hostname()})
```

#### .log.[trace|debug|info|warn|error] (data)
Logger instance exposed to be used by middlewares
```js
cache.log.info('hello world!')
```

## Examples
*Note! These examples are written using ES2015 syntax. The lib is exported using Babel as CJS modules*

### Basic usage
```js
import inMemoryCache from '@nrk/nodecache-as-promised'
const cache = inMemoryCache({ /* options */})

// implicit set cache on miss, or use cached value
cache.get('key', { worker: () => Promise.resolve({hello: 'world'}) })
  .then((data) => {
    console.log(data)
    // {
    //   value: {
    //     hello: 'world'
    //   },
    //   created: 123456789,
    //   cache: 'miss',
    //   TTL: 86400000
    // }
  })
```

### Basic usage with options
```js
import inMemoryCache from '@nrk/nodecache-as-promised';

const cache = inMemoryCache({
  initial: {                    // initial state
    foo: 'bar'
  },                            
  maxLength: 1000,              // LRU max object count
  maxAge: 24 * 60 * 60 * 1000   // LRU max age in ms
})
// set/overwrite cache key
cache.set('key', {hello: 'world'})
// imiplicit set cache on miss, or use cached value
cache.get('anotherkey', {
  worker: () => Promise.resolve({hello: 'world'}),
  ttl: 60 * 1000,               // TTL for cached object, in ms
  workerTimeout: 5 * 1000,      // worker timeout, in ms
  deltaWait: 5 * 1000,          // wait time, if worker fails
}).then((data) => {
    console.log(data)
    // {
    //   value: {
    //     hello: 'world'
    //   },
    //   created: 123456789,
    //   cache: 'miss',
    //   TTL: 86400000
    // }
  })
```

## Middlewares

### distCache middleware
Creating a new distCache middleware instance. The distCache middleware is extending the inMemoryCache instance by making a publish call to Redis using the provided `namespace` when the `.expire`-method is called. A subscription to the `namespace` ensures calls to `.expire` is distributed to all instances of the inMemoryCache using the same distCache middleware with the same `namespace`. It adds a couple of parameters to the `.debug`-method.

```js
import cache, {distCache} from '@nrk/nodecache-as-promised'
const cache = inMemoryCache()
cache.use(distCache(redisFactory, namespace))
```

#### Parameters
Parameters that must be provided upon creation:
- redisFactory - `Function`. A function that returns an ioredis compatible redisClient.
- namespace - `String`. Pub/sub-namespace used for distributed expiries

#### Example
```js
import inMemoryCache, {distCache} from '@nrk/nodecache-as-promised'
import Redis from 'ioredis'

// a factory function that returns a redisClient
const redisFactory = () => new Redis(/* options */)
const cache = inMemoryCache({initial: {fooKey: 'bar'}})
cache.use(distCache(redisFactory, 'namespace'))
// publish to redis (using wildcard)
cache.expire(['foo*'])
setTimeout(() => {
  cache.get('fooKey').then(console.log)
  // expired in server # 1 + 2
  // {value: {fooKey: 'bar'}, created: 123456789, cache: 'stale', TTL: 86400000}
}, 1000)
```

### persistentCache middleware
Creating a new persistentCache middleware instance. The persistentCache middleware is extending the inMemoryCache instance by serializing and storing any new values recieved via workers in `.get` or in `.set`-calls to Redis. In addition it deletes values from Redis when the `.del` and `.clear`-methods are called. Cache values evicted by the LRU-cache are also deleted. On creation it will load and set initial cache values by doing a search for stored keys on the provided `keySpace` (may be disabled using the option `bootLoad: false` - so that loading may be done afterwards using the provided `.load`-method). It adds a couple of parameters to the `.debug`-method.


```js
import cache, {persistentCache} from '@nrk/nodecache-as-promised'
const cache = inMemoryCache()
cache.use(persistentCache(redisFactory, options))
```

#### Parameters
Parameters that must be provided upon creation:
- redisFactory - `Function`. A function that returns an ioredis compatible redisClient.

#### options
- doNotPersist - `RegExp`. Keys matching this regexp is not persisted to cache. Default `null`
- keySpace - `String`. Prefix used when storing keys in redis.
- grace - `Number`. Used to calculate TTL in redis (before auto removal), ie. object.TTL + grace. Default `86400000` (24h)
- bootload - `Boolean`. Flag to choose if persisted cache is loaded from redis on middleware creation. Default `true`

#### Example
```js
import inMemoryCache, {persistentCache} from '@nrk/nodecache-as-promised'
import Redis from 'ioredis'

const redisFactory = () => new Redis(/* options */)
const cache = inMemoryCache({/* options */})
cache.use(persistentCache(
  redisFactory,
  {
    keySpace: 'myCache',   // key prefix used when storing in redis
    grace: 60 * 60         // auto expire unused keys in Redis after TTL + grace seconds
  }
))

cache.get('key', { worker: () => Promise.resolve('hello') })
// will store a key in redis, using key: myCache-<key>
// {value: 'hello', created: 123456789, cache: 'hit', TTL: 60000}
```

#### Combining middlewares

Example in combining persistentCache __and__ distCache

```js
import inMemoryCache, {distCache, persistentCache} from '@nrk/nodecache-as-promised'
import Redis from 'ioredis'

const redisFactory = () => new Redis(/* options */)
const cache = inMemoryCache({/* options */})
cache.use(distCache(redisFactory, 'namespace'))
cache.use(persistentCache(
  redisFactory,
  {
    keySpace: 'myCache',   // key prefix used when storing in redis
    grace: 60 * 60         // auto expire unused keys in Redis after TTL + grace seconds
  }
))

cache.expire(['foo*'])  // distributed expire of all keys starting with foo
cache.get('key', {
  worker: () => Promise.resolve('hello'),
  ttl: 60000,                       // in ms
  workerTimeout: 5000,
  deltaWait: 5000
}).then(console.log)
// will store a key in redis, using key: myCache-<key>
// {value: 'hello', created: 123456789, cache: 'miss', TTL: 60000}
```

## Creating your own middleware
A middleware consists of three parts:
1) an exported factory function
2) constructor arguments to be used within the middleware
3) an exported facade that corresponds with the overriden functions (appending a `next` parameter that runs the next function in the middleware chain)

Lets say you want to build a middleware that notifies some other part of your application that a new value has been set (eg. using RxJs streams).

Here's an example on how to achieve this:
```js
// export namespace to be applied in inMemoryCache.use().
export const streamingMiddleware = (onSet, onDispose) => (cacheInstance) => {
  // create a function that runs before the others in the middleware chain
  const set = (key, value, next) => {
    onSet(key, value)
    next(key, value)
  }

  // use functionality exposed by the inMemoryCache instance
  cacheInstance.addDisposer(onDispose)

  // export facade
  return {
    set
  }
}
```


---

## Local development
First clone the repo and install its dependencies:

```bash
git clone git@github.com:nrkno/nodecache-as-promised.git
git checkout -b feature/my-changes
cd nodecache-as-promised
npm install && npm run build && npm run test
```

## Building and committing
After having applied changes, remember to build and run/fix tests before pushing the changes upstream.

```bash
# run the tests, generate code coverage report
npm run test
# inspect code coverage
open ./coverage/lcov-report/index.html
# update the code
npm run build
git commit -am "Add my changes"
git push origin feature/my-changes
# then make a PR to the master branch,
# and assign one of the maintainers to review your code
```

> NOTE! Please make sure to keep commits small and clean (that the commit message actually refers to the updated files). Stylistically, make sure the commit message is **Capitalized** and **starts with a verb in the present tense** (eg. `Add minification support`).

## License

MIT © [NRK](https://www.nrk.no)
