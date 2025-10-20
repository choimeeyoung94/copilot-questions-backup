# Redis 성능 최적화 빠른 참조 가이드

이 문서는 Redis를 활용한 성능 최적화의 핵심 내용을 빠르게 참조할 수 있도록 요약한 것입니다.

자세한 내용은 [redis-optimization-guide.md](./redis-optimization-guide.md)를 참조하세요.

## 핵심 활용 패턴 요약

| 기능 | Redis 자료구조 | 키 패턴 | TTL | 주요 메서드 |
|------|---------------|---------|-----|-----------|
| **임시저장** | String | `temp_mail:{userId}` | 24시간 | `set`, `get` |
| **메일 목록** | List | `mail_list:{userId}:{page}` | 1시간 | `leftPush`, `range` |
| **메일 상세** | String | `mail_detail:{mailId}` | 1시간 | `set`, `get` |
| **예약 메일** | Sorted Set | `scheduled_mails` | 영구 | `add`, `rangeByScore`, `remove` |
| **읽음 상태** | Set | `mail_read:{mailId}` | 영구 | `add`, `isMember`, `size` |
| **파일 업로드** | Hash | `file_upload:{uploadId}` | 1-24시간 | `putAll`, `put`, `entries` |
| **채팅 메시지** | List | `chat_room:{roomId}` | - | `leftPush`, `range`, `trim` |
| **알림** | Sorted Set | `notifications:{userId}` | 영구 | `add`, `reverseRange`, `remove` |
| **읽지 않은 수** | String | `unread_count:{userId}` | 영구 | `increment`, `get`, `set` |
| **최근 대화상대** | Sorted Set | `recent_contacts:{userId}` | - | `add`, `reverseRange` |
| **검색 결과** | List | `search:{roomId}:{hash}` | 5-10분 | `rightPush`, `range` |

## 이메일 기능별 최적화

### 1. 임시저장 (temporary_storage)
```java
// 저장
redisTemplate.opsForValue().set("temp_mail:" + userId, json, 24, TimeUnit.HOURS);

// 조회
String draft = redisTemplate.opsForValue().get("temp_mail:" + userId);
```

### 2. 이메일 조회 (detail, sended_mail, trash)
```java
// 목록 캐싱
redisTemplate.opsForList().rightPush("mail_list:" + userId + ":" + page, mail);

// 상세 캐싱
redisTemplate.opsForValue().set("mail_detail:" + mailId, json, 1, TimeUnit.HOURS);
```

### 3. 예약 메일 (reservation_mail)
```java
// 예약 등록
redisTemplate.opsForZSet().add("scheduled_mails", json, timestamp);

// 처리할 예약 조회
Set<String> pending = redisTemplate.opsForZSet().rangeByScore("scheduled_mails", 0, now);
```

### 4. 수신확인 (receiver_check)
```java
// 읽음 처리
redisTemplate.opsForSet().add("mail_read:" + mailId, userId);

// 읽음 확인
Boolean isRead = redisTemplate.opsForSet().isMember("mail_read:" + mailId, userId);
```

### 5. 첨부파일 (attach)
```java
// 업로드 정보 저장
redisTemplate.opsForHash().putAll("file_upload:" + uploadId, fileInfo);

// 진행률 업데이트
redisTemplate.opsForHash().put("file_upload:" + uploadId, "progress", "50");
```

## 채팅 기능별 최적화

### 1. 메시지 전송 (send)
```java
// 메시지 추가 (최신 메시지 앞에)
redisTemplate.opsForList().leftPush("chat_room:" + roomId, json);
redisTemplate.opsForList().trim("chat_room:" + roomId, 0, 999); // 최근 1000개만

// 최근 메시지 조회
List<String> messages = redisTemplate.opsForList().range("chat_room:" + roomId, 0, 99);
```

### 2. 파일/이미지 첨부 (file_attach, image_attach)
```java
// 파일 메타데이터 저장
Map<String, Object> fileInfo = Map.of(
    "roomId", roomId,
    "fileName", fileName,
    "status", "UPLOADING"
);
redisTemplate.opsForHash().putAll("chat_file:" + uploadId, fileInfo);
```

### 3. 알림 (alarm)
```java
// 알림 추가
redisTemplate.opsForZSet().add("notifications:" + userId, json, timestamp);
redisTemplate.opsForValue().increment("unread_count:" + userId);

// 알림 조회
Set<String> notifs = redisTemplate.opsForZSet().reverseRange("notifications:" + userId, 0, 49);
```

### 4. 메시지 상세조회 (detail)
```java
// 캐싱
redisTemplate.opsForValue().set("message_detail:" + messageId, json, 1, TimeUnit.HOURS);

// 검색 결과 캐싱
redisTemplate.opsForList().rightPush("search:" + roomId + ":" + hash, result);
redisTemplate.expire("search:" + roomId + ":" + hash, 10, TimeUnit.MINUTES);
```

### 5. 수신자 선택 (receiver)
```java
// 최근 대화 상대 저장
redisTemplate.opsForZSet().add("recent_contacts:" + userId, contactId, timestamp);

// 최근 대화 상대 조회 (최신 순)
Set<String> contacts = redisTemplate.opsForZSet().reverseRange("recent_contacts:" + userId, 0, 49);
```

## 핵심 구현 패턴

### 1. Cache-Aside Pattern (가장 일반적)
```java
public Object getData(String key) {
    // 1. Redis 조회
    Object cached = redisTemplate.opsForValue().get(key);
    if (cached != null) return cached;
    
    // 2. DB 조회
    Object data = database.query(key);
    
    // 3. Redis 저장
    if (data != null) {
        redisTemplate.opsForValue().set(key, data, 1, TimeUnit.HOURS);
    }
    return data;
}
```

### 2. Write-Through Pattern
```java
public void saveData(String key, Object data) {
    database.save(data);  // DB 먼저
    redisTemplate.opsForValue().set(key, data, 1, TimeUnit.HOURS);  // Redis 캐싱
}
```

### 3. Write-Behind Pattern
```java
public void updateData(String key, Object data) {
    redisTemplate.opsForValue().set(key, data);  // Redis 먼저
    asyncExecutor.execute(() -> database.save(data));  // DB는 비동기
}
```

### 4. Pub/Sub Pattern (실시간 메시징)
```java
// 발행
redisTemplate.convertAndSend("chat_channel:" + roomId, message);

// 구독 (설정)
@Bean
public RedisMessageListenerContainer container(RedisConnectionFactory factory) {
    RedisMessageListenerContainer container = new RedisMessageListenerContainer();
    container.setConnectionFactory(factory);
    container.addMessageListener(listener, new PatternTopic("chat_channel:*"));
    return container;
}
```

### 5. 분산 락 Pattern
```java
public <T> T executeWithLock(String lockKey, Supplier<T> task) {
    String requestId = UUID.randomUUID().toString();
    try {
        // 락 획득
        while (!redisTemplate.opsForValue().setIfAbsent(lockKey, requestId, 10, TimeUnit.SECONDS)) {
            Thread.sleep(100);
        }
        return task.get();
    } finally {
        // 락 해제
        unlock(lockKey, requestId);
    }
}
```

### 6. Rate Limiting Pattern
```java
public boolean isAllowed(String userId, int limit, int seconds) {
    long now = System.currentTimeMillis();
    String key = "rate_limit:" + userId;
    
    redisTemplate.opsForZSet().removeRangeByScore(key, 0, now - seconds * 1000);
    Long count = redisTemplate.opsForZSet().size(key);
    
    if (count >= limit) return false;
    
    redisTemplate.opsForZSet().add(key, String.valueOf(now), now);
    return true;
}
```

## 성능 최적화 팁

### 1. Pipeline 사용 (일괄 처리)
```java
redisTemplate.executePipelined(new SessionCallback<Object>() {
    @Override
    public Object execute(RedisOperations operations) {
        for (String key : keys) {
            operations.opsForValue().get(key);
        }
        return null;
    }
});
```

### 2. Lua Script 사용 (원자성 보장)
```java
String script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                "return redis.call('del', KEYS[1]) else return 0 end";
redisTemplate.execute(new DefaultRedisScript<>(script, Long.class), 
    Collections.singletonList(key), value);
```

### 3. TTL 항상 설정
```java
// 모든 캐시에 TTL 설정하여 메모리 관리
redisTemplate.opsForValue().set(key, value, 1, TimeUnit.HOURS);
```

### 4. 큰 컬렉션 크기 제한
```java
// List 크기 제한
redisTemplate.opsForList().leftPush(key, value);
redisTemplate.opsForList().trim(key, 0, 999);

// Sorted Set 크기 제한
Long size = redisTemplate.opsForZSet().size(key);
if (size > 50) {
    redisTemplate.opsForZSet().removeRange(key, 0, size - 51);
}
```

### 5. 캐시 무효화
```java
// 패턴 기반 삭제
Set<String> keys = redisTemplate.keys("mail_list:" + userId + ":*");
if (keys != null && !keys.isEmpty()) {
    redisTemplate.delete(keys);
}
```

## 주의사항 체크리스트

- ✅ 모든 캐시 데이터에 TTL 설정
- ✅ 중요 데이터는 반드시 DB에도 저장
- ✅ Redis 장애 시 Fallback 로직 구현
- ✅ 캐시 무효화 전략 수립
- ✅ 큰 컬렉션 크기 제한
- ✅ 연결 풀 적절히 설정
- ✅ 메모리 정책 설정 (maxmemory, maxmemory-policy)
- ✅ 모니터링 및 알림 설정
- ✅ Redis 비밀번호 설정
- ✅ 네트워크 격리 (bind 설정)

## Spring Boot Redis 기본 설정

```yaml
# application.yml
spring:
  redis:
    host: localhost
    port: 6379
    password: ${REDIS_PASSWORD:}
    timeout: 2000ms
    lettuce:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5
        max-wait: 2000ms
```

```java
// RedisConfig.java
@Configuration
public class RedisConfig {
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

## 언제 Redis를 사용해야 하는가?

### ✅ Redis 사용이 적합한 경우:
1. 빠른 응답 시간이 필요한 경우 (< 100ms)
2. 읽기가 많은 경우 (읽기:쓰기 = 10:1 이상)
3. 실시간 데이터 처리 (채팅, 알림)
4. 세션/캐시 데이터 관리
5. 임시 데이터 저장
6. 순위/카운팅 (Sorted Set 활용)
7. 분산 환경 동기화 (분산 락)
8. Rate Limiting / Throttling

### ❌ Redis 사용이 부적합한 경우:
1. 영구 저장이 필요한 중요 데이터 (DB 사용)
2. 복잡한 쿼리가 필요한 경우 (JOIN, 집계 등)
3. 트랜잭션이 중요한 경우
4. 매우 큰 데이터 (> 100MB)
5. 자주 변경되지 않는 데이터 (캐싱 효과 낮음)

## 자주 사용하는 Redis 명령어

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `SET key value` | 문자열 저장 | `SET user:1 "John"` |
| `GET key` | 문자열 조회 | `GET user:1` |
| `SETEX key seconds value` | TTL과 함께 저장 | `SETEX session:123 3600 "data"` |
| `LPUSH key value` | 리스트 앞에 추가 | `LPUSH messages "Hello"` |
| `LRANGE key start stop` | 리스트 범위 조회 | `LRANGE messages 0 9` |
| `ZADD key score member` | Sorted Set 추가 | `ZADD leaderboard 100 "user1"` |
| `ZRANGE key start stop` | Sorted Set 범위 조회 | `ZRANGE leaderboard 0 9` |
| `SADD key member` | Set에 추가 | `SADD tags "redis"` |
| `SMEMBERS key` | Set 전체 조회 | `SMEMBERS tags` |
| `HSET key field value` | Hash 필드 설정 | `HSET user:1 name "John"` |
| `HGETALL key` | Hash 전체 조회 | `HGETALL user:1` |
| `EXPIRE key seconds` | TTL 설정 | `EXPIRE session:123 3600` |
| `DEL key` | 키 삭제 | `DEL user:1` |
| `KEYS pattern` | 패턴으로 키 검색 | `KEYS user:*` |

## 더 자세한 내용

전체 구현 예시와 모범 사례는 [redis-optimization-guide.md](./redis-optimization-guide.md)를 참조하세요.
