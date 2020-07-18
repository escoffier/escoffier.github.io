

#### RedisIndexedSessionRepository

A SessionRepository that is implemented using Spring Data's RedisOperations. In a web environment, this is typically used in combination with SessionRepositoryFilter . This implementation supports SessionDeletedEvent and SessionExpiredEvent by implementing MessageListener.
Creating a new instance
A typical example of how to create a new instance can be seen below:

```java
RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<>();
// ... configure redisTemplate ...

 RedisIndexedSessionRepository redisSessionRepository =
         new RedisIndexedSessionRepository(redisTemplate);
```

For additional information on how to create a RedisTemplate, refer to the Spring Data Redis Reference.

##### Storage Details

The sections below outline how Redis is updated for each operation. An example of creating a new session can be found below. The subsequent sections describe the details.

```
HMSET spring:session:sessions:33fdd1b6-b496-4b33-9f7d-df96679d32fe creationTime 1404360000000 maxInactiveInterval 1800 lastAccessedTime 1404360000000 sessionAttr:attrName someAttrValue sessionAttr2:attrName someAttrValue2
 EXPIRE spring:session:sessions:33fdd1b6-b496-4b33-9f7d-df96679d32fe 2100
 APPEND spring:session:sessions:expires:33fdd1b6-b496-4b33-9f7d-df96679d32fe ""
 EXPIRE spring:session:sessions:expires:33fdd1b6-b496-4b33-9f7d-df96679d32fe 1800
 SADD spring:session:expirations:1439245080000 expires:33fdd1b6-b496-4b33-9f7d-df96679d32fe
 EXPIRE spring:session:expirations1439245080000 2100
```



##### Saving a Session

Each session is stored in Redis as a Hash . Each session is set and updated using the HMSET command . An example of how each session is stored can be seen below.

```
HMSET spring:session:sessions:33fdd1b6-b496-4b33-9f7d-df96679d32fe creationTime 1404360000000 maxInactiveInterval 1800 lastAccessedTime 1404360000000 sessionAttr:attrName someAttrValue sessionAttr:attrName2 someAttrValue2
```

In this example, the session following statements are true about the session:

- The session id is 33fdd1b6-b496-4b33-9f7d-df96679d32fe
-  The session was created at 1404360000000 in milliseconds since midnight of 1/1/1970 GMT.
-  The session expires in 1800 seconds (30 minutes).
-  The session was last accessed at 1404360000000 in milliseconds since midnight of 1/1/1970 GMT.
 - The session has two attributes. The first is "attrName" with the value of "someAttrValue". The second session attribute is named "attrName2" with the value of "someAttrValue2".



##### Optimized Writes

The **RedisIndexedSessionRepository.RedisSession** keeps track of the properties that have changed and only updates those. This means if an attribute is written once and read many times we only need to write that attribute once. For example, assume the session attribute "sessionAttr2" from earlier was updated. The following would be executed upon saving:

```
HMSET spring:session:sessions:33fdd1b6-b496-4b33-9f7d-df96679d32fe sessionAttr:attrName2 newValue
```



##### SessionCreatedEvent

When a session is created an event is sent to Redis with the channel of "spring:session:channel:created:33fdd1b6-b496-4b33-9f7d-df96679d32fe" such that "33fdd1b6-b496-4b33-9f7d-df96679d32fe" is the session id. The body of the event will be the session that was created.
If registered as a *MessageListener*, then **RedisIndexedSessionRepository** will then translate the Redis message into a **SessionCreatedEvent**.

##### Expiration

An expiration is associated to each session using the EXPIRE command  based upon the **RedisIndexedSessionRepository.RedisSession.getMaxInactiveInterval()** . For example:

```
 EXPIRE spring:session:sessions:33fdd1b6-b496-4b33-9f7d-df96679d32fe 2100
```

You will note that the expiration that is set is 5 minutes after the session actually expires. This is necessary so that the value of the session can be accessed when the session expires. An expiration is set on the session itself five minutes after it actually expires to ensure it is cleaned up, but only after we perform any necessary processing.

> **NOTE**: The findById(String) method ensures that no expired sessions will be returned. This means there is no need to check the expiration before using a session

Spring Session relies on the expired and delete keyspace notifications  from Redis to fire a **SessionDestroyedEvent**. It is the **SessionDestroyedEvent** that ensures resources associated with the Session are cleaned up. For example, when using Spring Session's WebSocket support the Redis expired or delete event is what triggers any WebSocket connections associated with the session to be closed.
Expiration is not tracked directly on the session key itself since this would mean the session data would no longer be available. Instead a special session expires key is used. In our example the expires key is: 

```
APPEND spring:session:sessions:expires:33fdd1b6-b496-4b33-9f7d-df96679d32fe ""
 EXPIRE spring:session:sessions:expires:33fdd1b6-b496-4b33-9f7d-df96679d32fe 1800
```

When a session expires key is deleted or expires, the keyspace notification triggers a lookup of the actual session and a **SessionDestroyedEvent** is fired.

One problem with relying on Redis expiration exclusively is that Redis makes no guarantee of when the expired event will be fired if the key has not been accessed. Specifically the background task that Redis uses to clean up expired keys is a low priority task and may not trigger the key expiration. For additional details see Timing of expired events  section in the Redis documentation.
To circumvent the fact that expired events are not guaranteed to happen we can ensure that each key is accessed when it is expected to expire. This means that if the TTL is expired on the key, Redis will remove the key and fire the expired event when we try to access the key.

For this reason, each session expiration is also tracked to the nearest minute. This allows a background task to access the potentially expired sessions to ensure that Redis expired events are fired in a more deterministic fashion. For example:

```
 SADD spring:session:expirations:1439245080000 expires:33fdd1b6-b496-4b33-9f7d-df96679d32fe
 EXPIRE spring:session:expirations1439245080000 2100
```

The background task will then use these mappings to explicitly request each session expires key. By accessing the key, rather than deleting it, we ensure that Redis deletes the key for us only if the TTL is expired.

> NOTE: We do not explicitly delete the keys since in some instances there may be a race condition that incorrectly identifies a key as expired when it is not. Short of using distributed locks (which would kill our performance) there is no way to ensure the consistency of the expiration mapping. By simply accessing the key, we ensure that the key is only removed if the TTL on that key is expired.





