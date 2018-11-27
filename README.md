# sequelize-simple-cache

This is a simple, transparent, client-side, in-memory cache for [Sequelize](https://github.com/sequelize/sequelize) v4.
Cache invalidation is based on time-to-live (ttl).
Selectively add your Sequelize models to the cache.
Works with all storage engines supported by Sequelize.

[![Build Status](https://travis-ci.org/frankthelen/sequelize-simple-cache.svg?branch=master)](https://travis-ci.org/frankthelen/sequelize-simple-cache)
[![Coverage Status](https://coveralls.io/repos/github/frankthelen/sequelize-simple-cache/badge.svg?branch=master)](https://coveralls.io/github/frankthelen/sequelize-simple-cache?branch=master)
[![Dependencies Status](https://david-dm.org/frankthelen/sequelize-simple-cache.svg)](https://david-dm.org/frankthelen/sequelize-simple-cache)
[![Greenkeeper badge](https://badges.greenkeeper.io/frankthelen/sequelize-simple-cache.svg)](https://greenkeeper.io/)
[![Maintainability](https://api.codeclimate.com/v1/badges/c8bdb1fc29ef12070cac/maintainability)](https://codeclimate.com/github/frankthelen/sequelize-simple-cache/maintainability)
[![node](https://img.shields.io/node/v/sequelize-simple-cache.svg)]()
[![code style](https://img.shields.io/badge/code_style-airbnb-brightgreen.svg)](https://github.com/airbnb/javascript)
[![License Status](http://img.shields.io/npm/l/sequelize-simple-cache.svg)]()

This cache might work for you if you have database tables that
(1) are frequently read but very rarely written and
(2) contain only few rows of data.

In a project, we had a couple of database tables that were holding a sort of system configuration.
Something like 4 or 5 tables with some 50 rows of data.
Nearly every request needed this data, i.e., it was read all the time.
But updated only very rarely, once a day maybe.
So, pre-fetching or simple in-memory caching would work for us.

If that's not matching your scenario,
better look for something more sophisticated such as Redis or Memcached.

## Install

```bash
npm install sequelize-simple-cache
```

## Usage

Setup the cache along with loading your Sequelize models like this:
```javascript
const Sequelize = require('sequelize');
const SequelizeSimpleCache = require('sequelize-simple-cache');

const sequelize = new Sequelize('database', 'username', 'password', { ... });

// initialize cache
const cache = new SequelizeSimpleCache({
  User: { ttl: 5 * 60 }, // 5 minutes
  Page: { }, // default ttl is 1 hour
});

// add your models to the cache like this
const User = cache.init(sequelize.import('./models/user'));
const Page = cache.init(sequelize.import('./models/page'));

// no caching for this one (because it's not configured to be cached)
// will only add dummy decorators to the model for a homogeneous interface to all models
const Order = cache.init(sequelize.import('./models/order'));

// the Sequelize model API is fully transparent, no need to change anything.
// first time resolved from database, subsequent times from local cache.
const fred = User.findOne({ where: { username: 'fred' }});
```

## More Details

### Supported methods

Currently, the following methods on Sequelize model instances are supported for caching:
`findById`, `findOne`, `findAll`, `findAndCountAll`, `count`, `min`, `max`, `sum`.

### Non-cacheable queries / bypass caching

You need to avoid non-cacheable queries, e.g., queries containing dynamic timestamps.
```javascript
const { Op, fn } = require('sequelize');
// this is not good
Model.findAll({ where: { startDate: { [Op.lte]: new Date() }, } });
// you should do it this way
Model.findAll({ where: { startDate: { [Op.lte]: fn('NOW') }, } });
// if you don't want a query to be cached, you may explicitly bypass the cache like this
Model.noCache().findAll(...);
```

### Clear cache

There are these ways to clear the cache.
```javascript
const cache = new SequelizeSimpleCache({...});
// clear all
cache.clear();
// clear all entries of specific models
cache.clear('User', 'Page');
// or do the same on any model
Model.clearCache(); // only model
Model.clearCacheAll(); // entire cache
```

### Bypass caching

Caching can explicitly be bypassed like this:
```javascript
Model.noCache().findOne(...);
```

### Limit

This cache meant as a simple in-memory read cache for a very limited amount of data.
So, you should be able to control the size of the cache.
```javascript
const cache = new SequelizeSimpleCache({
  User: { }, // default limit is 50
  Page: { limit: 30 },
});
```

### Logging

There is "debug" and "ops" logging -- both are off by default.
Logging goes to `console.debug()` unless you set `delegate` to log somewhere else.
`event` is one of: `init`, `hit`, `miss`, `load`, `purge` or `ops`.
```javascript
const cache = new SequelizeSimpleCache({
  // ...
}, {
  debug: true,
  ops: 60, // seconds
  delegate: (event, details) => { ... },
});
```

### Unit testing

If you are mocking your Sequelize models in unit tests with [Sinon](https://sinonjs.org/) et al.,
caching might be somewhat counterproductive.
So, either clear the cache as needed in your unit tests. For example (using [mocha](https://mochajs.org/)):
```javascript
describe('My Test Suite', () => {
  beforeEach(() => {
    Model.clearCacheAll(); // on any model with the same effect
  });
  // ...
```

Or disable the cache right from the beginning.
A quick idea... have a specific config value in your project's `/config/default.js`
and `/config/test.js` to enable or disable the cache respectively.
And start your unit tests with setting `NODE_ENV=test` before.
This is actually the way I am doing it; plus a few extra unit tests for caching.
```javascript
const config = require('config');
const useCache = config.get('database.cache');
// initializing the cache
const cache = useCache ? new SequelizeSimpleCache({...}) : undefined;
// loading the models
const model = sequelize.import('./models/model');
const Model = useCache ? cache.init(model) : model;
```
