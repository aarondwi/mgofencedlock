# mgofencedlock
Distributed locks using mongodb, with fencing

Status: Not Yet Ready


Setup
----------------------
Run this script below in your mongo:

```javascript
db.lock.createIndex( { "key": 1 }, { unique: true } )
db.lock.createIndex( { "last_seen": 1 }, { expireAfterSeconds: 600 } )
```

**Notes:**

ExpireAfterSeconds is not set to remove lock directly, 
but at much higher time than lock TTL

This is used to implement fencing token generation,
and getting out of really small timing issue which restarts the version
It is also for not bloating the storage, in case it is not deleted normally

You will need to the above function manually
While doing that, you can also change default expiry to meet your use case


Queries use internally
------------------------------------
**acquire_lock**: 

```
db.lock.update({
    key: 'random-id', 
    $or: [
      {last_seen: null}, 
      {last_seen: {$lt: new Date() - 10000}}]
  }, {
    $inc: {version: 1}, 
    $set:{'owner': 'me', last_seen: new Date()}}, 
  {upsert: true})
```

**get_lock_data**:

```
db.lock.find({key: 'random-id', 'owner': 'me'}, {_id: 0})
```

**delete_lock**:

```
db.lock.remove({key: 'random-id', 'owner': 'me', version: 1})
```

**refresh_lock**:

```
db.lock.update(
  {key: 'random-id', 'owner': 'me', version: 1},
  {$set: {last_seen: new Date()}}
)
```

Document Schema
-------------------------
```json
{
  "version": 1, #fencing token
  "onwer": "random id generated by each lock, or supplied",
  "key": "unique id for your resource",
  "last_seen": "to check for expiry (probably stale)"
}
```