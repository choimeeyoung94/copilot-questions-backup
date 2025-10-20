# Redis를 활용한 성능 최적화 가이드

이 문서는 이메일 및 채팅 시스템에서 Redis를 활용하여 성능을 최적화하는 방법을 설명합니다.

## 목차
1. [이메일 관련 기능](#1-이메일-관련-기능)
2. [채팅 관련 기능](#2-채팅-관련-기능)
3. [Redis 활용 패턴 요약](#3-redis-활용-패턴-요약)
4. [구체적인 구현 예시](#4-구체적인-구현-예시)
5. [모범 사례 및 주의사항](#5-모범-사례-및-주의사항)

---

## 1. 이메일 관련 기능

### 1.1 임시저장 (temporary_storage)

**왜 Redis를 사용해야 하는가?**
- 메일 작성 중 데이터 손실 방지 (네트워크 끊김, 세션 만료 등)
- 빠른 임시 저장/복구 (밀리초 단위 응답)
- 자동 만료 기능으로 오래된 임시 데이터 자동 정리

**구현 방법:**
```java
// 임시저장 저장
public void saveDraft(String userId, MailDraft draft) {
    String key = "temp_mail:" + userId;
    String value = objectMapper.writeValueAsString(draft);
    
    // 24시간 TTL 설정
    redisTemplate.opsForValue().set(key, value, 24, TimeUnit.HOURS);
}

// 임시저장 복구
public MailDraft getDraft(String userId) {
    String key = "temp_mail:" + userId;
    String value = redisTemplate.opsForValue().get(key);
    
    if (value != null) {
        return objectMapper.readValue(value, MailDraft.class);
    }
    return null;
}

// 자동 저장 (프론트엔드에서 30초마다 호출)
@PostMapping("/api/mail/autosave")
public ResponseEntity<Void> autoSave(@RequestBody MailDraft draft) {
    saveDraft(getCurrentUserId(), draft);
    return ResponseEntity.ok().build();
}
```

**Redis 키 구조:**
- 키: `temp_mail:{userId}`
- 값: JSON 형식의 메일 내용 (제목, 본문, 수신자, 첨부파일 메타데이터)
- TTL: 24시간

### 1.2 이메일 조회 (detail, sended_mail, trash)

**왜 Redis를 사용해야 하는가?**
- 자주 조회하는 메일 목록의 빠른 응답
- DB 부하 감소
- 페이지네이션 성능 향상

**구현 방법:**
```java
// 최근 메일 목록 캐싱
public List<MailSummary> getRecentMails(String userId, int page, int size) {
    String cacheKey = "mail_list:" + userId + ":" + page;
    
    // Redis에서 먼저 조회
    List<MailSummary> cached = redisTemplate.opsForList()
        .range(cacheKey, 0, size - 1);
    
    if (cached != null && !cached.isEmpty()) {
        return cached;
    }
    
    // Redis에 없으면 DB 조회
    List<MailSummary> mails = mailRepository.findByUserId(userId, 
        PageRequest.of(page, size));
    
    // Redis에 캐싱 (1시간 TTL)
    if (!mails.isEmpty()) {
        mails.forEach(mail -> 
            redisTemplate.opsForList().rightPush(cacheKey, mail));
        redisTemplate.expire(cacheKey, 1, TimeUnit.HOURS);
    }
    
    return mails;
}

// 메일 상세 조회
public Mail getMailDetail(String mailId) {
    String cacheKey = "mail_detail:" + mailId;
    
    // Redis 조회
    String cached = redisTemplate.opsForValue().get(cacheKey);
    if (cached != null) {
        return objectMapper.readValue(cached, Mail.class);
    }
    
    // DB 조회 및 캐싱
    Mail mail = mailRepository.findById(mailId).orElse(null);
    if (mail != null) {
        redisTemplate.opsForValue().set(cacheKey, 
            objectMapper.writeValueAsString(mail), 
            1, TimeUnit.HOURS);
    }
    
    return mail;
}

// 메일 전송/삭제 시 캐시 무효화
public void invalidateMailCache(String userId, String mailId) {
    // 목록 캐시 삭제
    Set<String> keys = redisTemplate.keys("mail_list:" + userId + ":*");
    if (keys != null && !keys.isEmpty()) {
        redisTemplate.delete(keys);
    }
    
    // 상세 캐시 삭제
    redisTemplate.delete("mail_detail:" + mailId);
}
```

**Redis 키 구조:**
- 메일 목록: `mail_list:{userId}:{page}` (List 타입)
- 메일 상세: `mail_detail:{mailId}` (String 타입)
- TTL: 1시간

### 1.3 이메일 예약/알림 (reservation_mail)

**왜 Redis를 사용해야 하는가?**
- 시간 기반 스케줄링 최적화
- 예약 메일의 실시간 처리
- 분산 환경에서도 중복 처리 방지

**구현 방법:**
```java
// 예약 메일 등록
public void scheduleEmail(ReservationMail mail) {
    String key = "scheduled_mails";
    long timestamp = mail.getScheduledTime().getTime();
    
    // Sorted Set에 예약 시간을 score로 저장
    redisTemplate.opsForZSet().add(key, 
        objectMapper.writeValueAsString(mail), 
        timestamp);
}

// 예약 메일 스케줄러 (1분마다 실행)
@Scheduled(fixedRate = 60000)
public void processPendingEmails() {
    String key = "scheduled_mails";
    long now = System.currentTimeMillis();
    
    // 현재 시간 이전의 예약 메일 조회
    Set<String> pending = redisTemplate.opsForZSet()
        .rangeByScore(key, 0, now);
    
    if (pending != null && !pending.isEmpty()) {
        for (String mailJson : pending) {
            ReservationMail mail = objectMapper.readValue(
                mailJson, ReservationMail.class);
            
            // 메일 전송
            sendEmail(mail);
            
            // Redis에서 제거
            redisTemplate.opsForZSet().remove(key, mailJson);
        }
    }
}

// 예약 취소
public void cancelScheduledEmail(String mailId) {
    String key = "scheduled_mails";
    Set<String> all = redisTemplate.opsForZSet().range(key, 0, -1);
    
    if (all != null) {
        for (String mailJson : all) {
            ReservationMail mail = objectMapper.readValue(
                mailJson, ReservationMail.class);
            if (mail.getId().equals(mailId)) {
                redisTemplate.opsForZSet().remove(key, mailJson);
                break;
            }
        }
    }
}
```

**Redis 키 구조:**
- 키: `scheduled_mails` (Sorted Set 타입)
- Score: 예약 시간 (Unix timestamp)
- Value: 예약 메일 정보 (JSON)

### 1.4 수신확인 (receiver_check)

**왜 Redis를 사용해야 하는가?**
- 실시간 읽음 상태 업데이트
- 빠른 상태 조회
- DB 쓰기 부하 감소 (배치 처리 가능)

**구현 방법:**
```java
// 메일 읽음 처리
public void markAsRead(String mailId, String userId) {
    String key = "mail_read:" + mailId;
    
    // Set에 읽은 사용자 추가
    redisTemplate.opsForSet().add(key, userId);
    
    // 읽은 시간 저장
    redisTemplate.opsForHash().put(
        "mail_read_time:" + mailId, 
        userId, 
        String.valueOf(System.currentTimeMillis())
    );
    
    // 비동기로 DB 업데이트
    asyncUpdateReadStatus(mailId, userId);
}

// 읽음 상태 확인
public boolean isRead(String mailId, String userId) {
    String key = "mail_read:" + mailId;
    return redisTemplate.opsForSet().isMember(key, userId);
}

// 읽은 사용자 목록 조회
public Set<String> getReadUsers(String mailId) {
    String key = "mail_read:" + mailId;
    return redisTemplate.opsForSet().members(key);
}

// 읽음 카운트 조회
public Long getReadCount(String mailId) {
    String key = "mail_read:" + mailId;
    return redisTemplate.opsForSet().size(key);
}
```

**Redis 키 구조:**
- 읽음 여부: `mail_read:{mailId}` (Set 타입, 읽은 사용자 ID 저장)
- 읽은 시간: `mail_read_time:{mailId}` (Hash 타입)

### 1.5 첨부파일 업로드/전송 (attach)

**왜 Redis를 사용해야 하는가?**
- 업로드 진행 상태 실시간 추적
- 임시 파일 메타데이터 관리
- 업로드 완료 전 미리보기 제공

**구현 방법:**
```java
// 파일 업로드 시작
public String startFileUpload(String userId, FileMetadata metadata) {
    String uploadId = UUID.randomUUID().toString();
    String key = "file_upload:" + uploadId;
    
    // 업로드 정보 저장
    Map<String, Object> uploadInfo = Map.of(
        "userId", userId,
        "fileName", metadata.getFileName(),
        "fileSize", metadata.getFileSize(),
        "mimeType", metadata.getMimeType(),
        "status", "UPLOADING",
        "progress", 0,
        "createdAt", System.currentTimeMillis()
    );
    
    redisTemplate.opsForHash().putAll(key, uploadInfo);
    redisTemplate.expire(key, 1, TimeUnit.HOURS);
    
    return uploadId;
}

// 업로드 진행률 업데이트
public void updateUploadProgress(String uploadId, int progress) {
    String key = "file_upload:" + uploadId;
    redisTemplate.opsForHash().put(key, "progress", String.valueOf(progress));
    
    if (progress >= 100) {
        redisTemplate.opsForHash().put(key, "status", "COMPLETED");
    }
}

// 업로드 완료 처리
public void completeFileUpload(String uploadId, String s3Url) {
    String key = "file_upload:" + uploadId;
    
    redisTemplate.opsForHash().put(key, "status", "COMPLETED");
    redisTemplate.opsForHash().put(key, "s3Url", s3Url);
    redisTemplate.opsForHash().put(key, "progress", "100");
    
    // TTL 연장 (24시간)
    redisTemplate.expire(key, 24, TimeUnit.HOURS);
}

// 업로드 상태 조회
public Map<Object, Object> getUploadStatus(String uploadId) {
    String key = "file_upload:" + uploadId;
    return redisTemplate.opsForHash().entries(key);
}

// 사용자의 임시 파일 목록
public List<String> getUserTempFiles(String userId) {
    String pattern = "file_upload:*";
    Set<String> keys = redisTemplate.keys(pattern);
    
    List<String> userFiles = new ArrayList<>();
    if (keys != null) {
        for (String key : keys) {
            String storedUserId = (String) redisTemplate.opsForHash()
                .get(key, "userId");
            if (userId.equals(storedUserId)) {
                userFiles.add(key.replace("file_upload:", ""));
            }
        }
    }
    
    return userFiles;
}
```

**Redis 키 구조:**
- 키: `file_upload:{uploadId}` (Hash 타입)
- 필드: userId, fileName, fileSize, mimeType, status, progress, s3Url
- TTL: 업로드 중 1시간, 완료 후 24시간

---

## 2. 채팅 관련 기능

### 2.1 메시지 전송/수신 (send)

**왜 Redis를 사용해야 하는가?**
- 실시간 메시지 전달
- 최근 메시지 빠른 조회
- DB 쓰기 부하 분산

**구현 방법:**
```java
// 메시지 전송
public void sendMessage(ChatMessage message) {
    String roomKey = "chat_room:" + message.getRoomId();
    
    // Redis List에 메시지 추가 (최신 메시지가 앞에)
    redisTemplate.opsForList().leftPush(roomKey, 
        objectMapper.writeValueAsString(message));
    
    // 최근 1000개만 유지
    redisTemplate.opsForList().trim(roomKey, 0, 999);
    
    // 비동기로 DB에 저장
    asyncSaveMessageToDB(message);
    
    // 실시간 알림 (Pub/Sub)
    redisTemplate.convertAndSend(
        "chat_channel:" + message.getRoomId(), 
        message
    );
}

// 최근 메시지 조회
public List<ChatMessage> getRecentMessages(String roomId, int limit) {
    String roomKey = "chat_room:" + roomId;
    
    List<String> messages = redisTemplate.opsForList()
        .range(roomKey, 0, limit - 1);
    
    if (messages == null || messages.isEmpty()) {
        // Redis에 없으면 DB에서 조회 후 캐싱
        List<ChatMessage> dbMessages = chatRepository
            .findRecentMessages(roomId, limit);
        
        if (!dbMessages.isEmpty()) {
            dbMessages.forEach(msg -> 
                redisTemplate.opsForList().rightPush(roomKey, 
                    objectMapper.writeValueAsString(msg))
            );
        }
        
        return dbMessages;
    }
    
    return messages.stream()
        .map(json -> objectMapper.readValue(json, ChatMessage.class))
        .collect(Collectors.toList());
}

// 이전 메시지 로딩 (무한 스크롤)
public List<ChatMessage> loadOlderMessages(String roomId, 
                                          String lastMessageId, 
                                          int limit) {
    // 오래된 메시지는 DB에서 조회
    return chatRepository.findMessagesBeforeId(roomId, lastMessageId, limit);
}
```

**Redis 키 구조:**
- 메시지 목록: `chat_room:{roomId}` (List 타입)
- 메시지 저장: 최근 1000개만 유지
- 오래된 메시지: DB에서 조회

### 2.2 파일/이미지 첨부 (file_attach, image_attach)

**왜 Redis를 사용해야 하는가?**
- 업로드 진행 상태 추적
- 썸네일/미리보기 빠른 제공
- 임시 파일 관리

**구현 방법:**
```java
// 채팅 파일 업로드
public String uploadChatFile(String roomId, MultipartFile file) {
    String uploadId = UUID.randomUUID().toString();
    String key = "chat_file:" + uploadId;
    
    // 업로드 메타데이터 저장
    Map<String, Object> fileInfo = Map.of(
        "roomId", roomId,
        "fileName", file.getOriginalFilename(),
        "fileSize", file.getSize(),
        "contentType", file.getContentType(),
        "status", "UPLOADING",
        "uploadedAt", System.currentTimeMillis()
    );
    
    redisTemplate.opsForHash().putAll(key, fileInfo);
    
    // S3 업로드 (비동기)
    CompletableFuture.supplyAsync(() -> {
        String s3Url = s3Service.upload(file);
        
        // 업로드 완료 후 URL 저장
        redisTemplate.opsForHash().put(key, "s3Url", s3Url);
        redisTemplate.opsForHash().put(key, "status", "COMPLETED");
        
        // 이미지인 경우 썸네일 생성
        if (isImage(file.getContentType())) {
            String thumbnailUrl = generateThumbnail(s3Url);
            redisTemplate.opsForHash().put(key, "thumbnailUrl", thumbnailUrl);
        }
        
        return s3Url;
    });
    
    // TTL 설정 (24시간)
    redisTemplate.expire(key, 24, TimeUnit.HOURS);
    
    return uploadId;
}

// 파일 정보 조회
public Map<Object, Object> getChatFileInfo(String uploadId) {
    String key = "chat_file:" + uploadId;
    return redisTemplate.opsForHash().entries(key);
}
```

**Redis 키 구조:**
- 키: `chat_file:{uploadId}` (Hash 타입)
- 필드: roomId, fileName, fileSize, contentType, status, s3Url, thumbnailUrl
- TTL: 24시간

### 2.3 메시지 알림 (alarm)

**왜 Redis를 사용해야 하는가?**
- 실시간 알림 처리
- 읽지 않은 메시지 카운트 빠른 조회
- 알림 상태 관리

**구현 방법:**
```java
// 알림 생성
public void createNotification(String userId, ChatMessage message) {
    String key = "notifications:" + userId;
    
    // Sorted Set에 알림 추가 (타임스탬프를 score로)
    NotificationData notif = new NotificationData(
        message.getRoomId(),
        message.getSenderId(),
        message.getContent(),
        message.getTimestamp()
    );
    
    redisTemplate.opsForZSet().add(key, 
        objectMapper.writeValueAsString(notif), 
        message.getTimestamp());
    
    // 읽지 않은 알림 카운트 증가
    String countKey = "unread_count:" + userId;
    redisTemplate.opsForValue().increment(countKey);
}

// 읽지 않은 알림 수 조회
public Long getUnreadCount(String userId) {
    String key = "unread_count:" + userId;
    String count = redisTemplate.opsForValue().get(key);
    return count != null ? Long.parseLong(count) : 0L;
}

// 알림 목록 조회
public List<NotificationData> getNotifications(String userId, int limit) {
    String key = "notifications:" + userId;
    
    Set<String> notifications = redisTemplate.opsForZSet()
        .reverseRange(key, 0, limit - 1);
    
    if (notifications == null || notifications.isEmpty()) {
        return Collections.emptyList();
    }
    
    return notifications.stream()
        .map(json -> objectMapper.readValue(json, NotificationData.class))
        .collect(Collectors.toList());
}

// 알림 읽음 처리
public void markNotificationsAsRead(String userId) {
    String countKey = "unread_count:" + userId;
    redisTemplate.opsForValue().set(countKey, "0");
}

// 특정 채팅방 알림 읽음 처리
public void markRoomNotificationsAsRead(String userId, String roomId) {
    String key = "notifications:" + userId;
    Set<String> all = redisTemplate.opsForZSet().range(key, 0, -1);
    
    if (all != null) {
        int removedCount = 0;
        for (String notifJson : all) {
            NotificationData notif = objectMapper.readValue(
                notifJson, NotificationData.class);
            if (notif.getRoomId().equals(roomId)) {
                redisTemplate.opsForZSet().remove(key, notifJson);
                removedCount++;
            }
        }
        
        // 카운트 감소
        if (removedCount > 0) {
            String countKey = "unread_count:" + userId;
            redisTemplate.opsForValue().decrement(countKey, removedCount);
        }
    }
}
```

**Redis 키 구조:**
- 알림 목록: `notifications:{userId}` (Sorted Set 타입)
- 읽지 않은 수: `unread_count:{userId}` (String 타입)
- Score: 메시지 타임스탬프

### 2.4 메시지 상세조회 (detail)

**왜 Redis를 사용해야 하는가?**
- 자주 조회하는 메시지 캐싱
- 조회 성능 향상
- DB 부하 감소

**구현 방법:**
```java
// 메시지 상세 조회
public ChatMessage getMessageDetail(String messageId) {
    String key = "message_detail:" + messageId;
    
    // Redis 조회
    String cached = redisTemplate.opsForValue().get(key);
    if (cached != null) {
        return objectMapper.readValue(cached, ChatMessage.class);
    }
    
    // DB 조회
    ChatMessage message = chatRepository.findById(messageId).orElse(null);
    
    // 캐싱 (1시간)
    if (message != null) {
        redisTemplate.opsForValue().set(key, 
            objectMapper.writeValueAsString(message),
            1, TimeUnit.HOURS);
    }
    
    return message;
}

// 메시지 검색 결과 캐싱
public List<ChatMessage> searchMessages(String roomId, String keyword) {
    String cacheKey = "search:" + roomId + ":" + 
        DigestUtils.md5Hex(keyword);
    
    // Redis 조회
    List<String> cached = redisTemplate.opsForList()
        .range(cacheKey, 0, -1);
    
    if (cached != null && !cached.isEmpty()) {
        return cached.stream()
            .map(json -> objectMapper.readValue(json, ChatMessage.class))
            .collect(Collectors.toList());
    }
    
    // DB 검색
    List<ChatMessage> results = chatRepository.searchMessages(roomId, keyword);
    
    // 캐싱 (10분)
    if (!results.isEmpty()) {
        results.forEach(msg -> 
            redisTemplate.opsForList().rightPush(cacheKey, 
                objectMapper.writeValueAsString(msg))
        );
        redisTemplate.expire(cacheKey, 10, TimeUnit.MINUTES);
    }
    
    return results;
}
```

**Redis 키 구조:**
- 메시지 상세: `message_detail:{messageId}` (String 타입)
- 검색 결과: `search:{roomId}:{keywordHash}` (List 타입)
- TTL: 상세 1시간, 검색 10분

### 2.5 수신자 선택/조회 (receiver)

**왜 Redis를 사용해야 하는가?**
- 최근 대화 상대 빠른 조회
- 자동완성 성능 향상
- 주소록 캐싱

**구현 방법:**
```java
// 최근 대화 상대 저장
public void updateRecentContacts(String userId, String contactId) {
    String key = "recent_contacts:" + userId;
    
    // Sorted Set에 저장 (최근 시간을 score로)
    long timestamp = System.currentTimeMillis();
    redisTemplate.opsForZSet().add(key, contactId, timestamp);
    
    // 최근 50명만 유지
    Long size = redisTemplate.opsForZSet().size(key);
    if (size != null && size > 50) {
        redisTemplate.opsForZSet().removeRange(key, 0, size - 51);
    }
}

// 최근 대화 상대 조회
public List<String> getRecentContacts(String userId, int limit) {
    String key = "recent_contacts:" + userId;
    
    Set<String> contacts = redisTemplate.opsForZSet()
        .reverseRange(key, 0, limit - 1);
    
    return contacts != null ? new ArrayList<>(contacts) : 
        Collections.emptyList();
}

// 자동완성용 검색
public List<UserInfo> searchContacts(String userId, String query) {
    String cacheKey = "contact_search:" + userId + ":" + 
        DigestUtils.md5Hex(query);
    
    // Redis 조회
    List<String> cached = redisTemplate.opsForList()
        .range(cacheKey, 0, -1);
    
    if (cached != null && !cached.isEmpty()) {
        return cached.stream()
            .map(json -> objectMapper.readValue(json, UserInfo.class))
            .collect(Collectors.toList());
    }
    
    // DB 검색
    List<UserInfo> results = userRepository.searchByName(query);
    
    // 캐싱 (5분)
    if (!results.isEmpty()) {
        results.forEach(user -> 
            redisTemplate.opsForList().rightPush(cacheKey, 
                objectMapper.writeValueAsString(user))
        );
        redisTemplate.expire(cacheKey, 5, TimeUnit.MINUTES);
    }
    
    return results;
}

// 주소록 캐싱
public List<UserInfo> getAddressBook(String userId) {
    String key = "address_book:" + userId;
    
    // Redis 조회
    List<String> cached = redisTemplate.opsForList().range(key, 0, -1);
    
    if (cached != null && !cached.isEmpty()) {
        return cached.stream()
            .map(json -> objectMapper.readValue(json, UserInfo.class))
            .collect(Collectors.toList());
    }
    
    // DB 조회
    List<UserInfo> addressBook = userRepository.getAddressBook(userId);
    
    // 캐싱 (1시간)
    if (!addressBook.isEmpty()) {
        addressBook.forEach(user -> 
            redisTemplate.opsForList().rightPush(key, 
                objectMapper.writeValueAsString(user))
        );
        redisTemplate.expire(key, 1, TimeUnit.HOURS);
    }
    
    return addressBook;
}
```

**Redis 키 구조:**
- 최근 대화 상대: `recent_contacts:{userId}` (Sorted Set 타입)
- 검색 결과: `contact_search:{userId}:{queryHash}` (List 타입)
- 주소록: `address_book:{userId}` (List 타입)

---

## 3. Redis 활용 패턴 요약

| 기능 | Redis 활용 이유 & 방법 | Redis 자료구조 | TTL |
|------|----------------------|---------------|-----|
| **임시저장** | 빠른 임시 저장/복구, 세션 만료/이탈/네트워크 장애 대응 | String | 24시간 |
| **최근 조회/목록** | 최근 메일/채팅 메시지(100~1000개) 캐싱, 1초내 응답 | List | 1시간 |
| **알림/푸시** | 실시간 알림/읽음상태 관리, 빠른 상태변경 및 알림 전송 | Sorted Set, String | 영구 |
| **예약/스케줄** | 예약메일/예약푸시 등 시간 기반 처리 | Sorted Set | 영구 |
| **파일 임시메타** | S3 파일 업로드 진행중 정보, 임시 미리보기/URL 등 캐싱 | Hash | 1-24시간 |
| **자동완성** | 최근 대화/주소록/검색어 등 빠른 조회 | Sorted Set, List | 5분-1시간 |
| **읽음 상태** | 메일/메시지 읽음 여부 실시간 관리 | Set, Hash | 영구 |
| **실시간 메시지** | 채팅 메시지 실시간 전달 및 최근 메시지 캐싱 | List, Pub/Sub | - |

---

## 4. 구체적인 구현 예시

### 4.1 Redis 설정 (Spring Boot)

**application.yml:**
```yaml
spring:
  redis:
    host: localhost
    port: 6379
    password: ${REDIS_PASSWORD:}
    timeout: 2000ms
    lettuce:
      pool:
        max-active: 10
        max-idle: 10
        min-idle: 2
        max-wait: -1ms
```

**RedisConfig.java:**
```java
@Configuration
@EnableRedisRepositories
public class RedisConfig {
    
    @Value("${spring.redis.host}")
    private String host;
    
    @Value("${spring.redis.port}")
    private int port;
    
    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration config = 
            new RedisStandaloneConfiguration(host, port);
        return new LettuceConnectionFactory(config);
    }
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        
        // JSON 직렬화 설정
        Jackson2JsonRedisSerializer<Object> serializer = 
            new Jackson2JsonRedisSerializer<>(Object.class);
        
        ObjectMapper mapper = new ObjectMapper();
        mapper.setVisibility(PropertyAccessor.ALL, 
            JsonAutoDetect.Visibility.ANY);
        mapper.activateDefaultTyping(
            LaissezFaireSubTypeValidator.instance,
            ObjectMapper.DefaultTyping.NON_FINAL);
        serializer.setObjectMapper(mapper);
        
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(serializer);
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(serializer);
        
        template.afterPropertiesSet();
        return template;
    }
    
    @Bean
    public RedisTemplate<String, String> stringRedisTemplate() {
        RedisTemplate<String, String> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        
        StringRedisSerializer serializer = new StringRedisSerializer();
        template.setKeySerializer(serializer);
        template.setValueSerializer(serializer);
        template.setHashKeySerializer(serializer);
        template.setHashValueSerializer(serializer);
        
        template.afterPropertiesSet();
        return template;
    }
    
    @Bean
    public CacheManager cacheManager() {
        RedisCacheConfiguration config = RedisCacheConfiguration
            .defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new GenericJackson2JsonRedisSerializer()));
        
        return RedisCacheManager.builder(redisConnectionFactory())
            .cacheDefaults(config)
            .build();
    }
}
```

### 4.2 캐시 무효화 전략

```java
@Service
public class CacheInvalidationService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    // 패턴 기반 키 삭제 (SCAN 사용 - 프로덕션 안전)
    public void deleteByPattern(String pattern) {
        Set<String> keys = new HashSet<>();
        ScanOptions options = ScanOptions.scanOptions()
            .match(pattern)
            .count(100)
            .build();
        
        try (Cursor<String> cursor = redisTemplate.scan(options)) {
            while (cursor.hasNext()) {
                keys.add(cursor.next());
            }
        }
        
        if (!keys.isEmpty()) {
            redisTemplate.delete(keys);
        }
    }
    
    // 소규모 환경에서만 사용 (주의: 프로덕션에서는 Redis 블로킹 가능)
    public void deleteByPatternUnsafe(String pattern) {
        Set<String> keys = redisTemplate.keys(pattern);
        if (keys != null && !keys.isEmpty()) {
            redisTemplate.delete(keys);
        }
    }
    
    // 사용자 관련 모든 캐시 삭제
    public void invalidateUserCache(String userId) {
        deleteByPattern("temp_mail:" + userId + "*");
        deleteByPattern("mail_list:" + userId + "*");
        deleteByPattern("chat_room:*" + userId + "*");
        deleteByPattern("notifications:" + userId + "*");
        deleteByPattern("recent_contacts:" + userId + "*");
        deleteByPattern("address_book:" + userId + "*");
    }
    
    // 메일 관련 캐시 삭제
    public void invalidateMailCache(String userId, String mailId) {
        // 목록 캐시
        deleteByPattern("mail_list:" + userId + "*");
        
        // 상세 캐시
        redisTemplate.delete("mail_detail:" + mailId);
        
        // 읽음 상태
        redisTemplate.delete("mail_read:" + mailId);
        redisTemplate.delete("mail_read_time:" + mailId);
    }
    
    // 채팅방 관련 캐시 삭제
    public void invalidateChatRoomCache(String roomId) {
        deleteByPattern("chat_room:" + roomId + "*");
        deleteByPattern("message_detail:*" + roomId + "*");
        deleteByPattern("search:" + roomId + "*");
    }
}
```

### 4.3 Redis Pub/Sub을 이용한 실시간 메시징

```java
@Configuration
public class RedisMessageConfig {
    
    @Bean
    public RedisMessageListenerContainer redisMessageListenerContainer(
            RedisConnectionFactory connectionFactory,
            MessageListenerAdapter listenerAdapter) {
        
        RedisMessageListenerContainer container = 
            new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.addMessageListener(listenerAdapter, 
            new PatternTopic("chat_channel:*"));
        
        return container;
    }
    
    @Bean
    public MessageListenerAdapter listenerAdapter(
            ChatMessageSubscriber subscriber) {
        return new MessageListenerAdapter(subscriber, "onMessage");
    }
}

@Service
public class ChatMessagePublisher {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public void publishMessage(String roomId, ChatMessage message) {
        String channel = "chat_channel:" + roomId;
        redisTemplate.convertAndSend(channel, message);
    }
}

@Service
public class ChatMessageSubscriber {
    
    @Autowired
    private SimpMessagingTemplate messagingTemplate;
    
    public void onMessage(Message message, byte[] pattern) {
        try {
            String messageBody = new String(message.getBody());
            ChatMessage chatMessage = objectMapper.readValue(
                messageBody, ChatMessage.class);
            
            // WebSocket으로 클라이언트에 전송
            messagingTemplate.convertAndSend(
                "/topic/chat/" + chatMessage.getRoomId(), 
                chatMessage);
                
        } catch (Exception e) {
            logger.error("Error processing message", e);
        }
    }
}
```

### 4.4 분산 락 (Distributed Lock)

```java
@Component
public class RedisLockManager {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    /**
     * 분산 락 획득
     * @param lockKey 락 키
     * @param requestId 요청 ID (UUID)
     * @param expireTime 만료 시간 (초)
     * @return 락 획득 성공 여부
     */
    public boolean tryLock(String lockKey, String requestId, int expireTime) {
        Boolean result = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, requestId, expireTime, TimeUnit.SECONDS);
        return Boolean.TRUE.equals(result);
    }
    
    /**
     * 분산 락 해제
     * @param lockKey 락 키
     * @param requestId 요청 ID
     * @return 락 해제 성공 여부
     */
    public boolean unlock(String lockKey, String requestId) {
        String script = 
            "if redis.call('get', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('del', KEYS[1]) " +
            "else " +
            "    return 0 " +
            "end";
        
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            Collections.singletonList(lockKey),
            requestId);
        
        return result != null && result == 1L;
    }
    
    /**
     * 락을 획득하고 작업 실행 (exponential backoff 적용)
     */
    public <T> T executeWithLock(String lockKey, 
                                  int expireTime, 
                                  Supplier<T> task) {
        String requestId = UUID.randomUUID().toString();
        long startTime = System.currentTimeMillis();
        int retryCount = 0;
        
        try {
            // 락 획득 시도 (최대 5초 대기, exponential backoff)
            while (!tryLock(lockKey, requestId, expireTime)) {
                if (System.currentTimeMillis() - startTime > 5000) {
                    throw new RuntimeException("Failed to acquire lock: " + lockKey);
                }
                
                // Exponential backoff with jitter (지수 백오프 + 지터)
                int backoff = Math.min(100 * (1 << retryCount), 1000);
                int jitter = new Random().nextInt(50);
                Thread.sleep(backoff + jitter);
                retryCount++;
            }
            
            // 작업 실행
            return task.get();
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Interrupted while acquiring lock", e);
        } finally {
            // 락 해제
            unlock(lockKey, requestId);
        }
    }
}

// 사용 예시
@Service
public class PaymentService {
    
    @Autowired
    private RedisLockManager lockManager;
    
    public void processPayment(String userId, double amount) {
        String lockKey = "payment_lock:" + userId;
        
        lockManager.executeWithLock(lockKey, 10, () -> {
            // 중복 결제 방지를 위한 비즈니스 로직
            Payment payment = createPayment(userId, amount);
            return payment;
        });
    }
}
```

### 4.5 Rate Limiting (속도 제한)

```java
@Component
public class RedisRateLimiter {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    /**
     * 슬라이딩 윈도우 방식의 Rate Limiting
     * @param key 제한 키 (예: "api_call:userId")
     * @param limit 제한 횟수
     * @param windowSeconds 시간 윈도우 (초)
     * @return 허용 여부
     */
    public boolean isAllowed(String key, int limit, int windowSeconds) {
        long now = System.currentTimeMillis();
        long windowStart = now - (windowSeconds * 1000L);
        
        // 오래된 요청 제거
        redisTemplate.opsForZSet().removeRangeByScore(key, 0, windowStart);
        
        // 현재 요청 수 확인
        Long count = redisTemplate.opsForZSet().size(key);
        
        if (count != null && count >= limit) {
            return false;
        }
        
        // 현재 요청 추가
        redisTemplate.opsForZSet().add(key, String.valueOf(now), now);
        redisTemplate.expire(key, windowSeconds, TimeUnit.SECONDS);
        
        return true;
    }
    
    /**
     * 토큰 버킷 방식의 Rate Limiting
     */
    public boolean checkRateLimit(String userId, int maxRequests, int windowSeconds) {
        String key = "rate_limit:" + userId;
        String countKey = key + ":count";
        String timestampKey = key + ":timestamp";
        
        Long currentCount = redisTemplate.opsForValue().increment(countKey, 0);
        String lastTimestamp = redisTemplate.opsForValue().get(timestampKey);
        
        long now = System.currentTimeMillis();
        
        if (currentCount == null || currentCount == 0 || lastTimestamp == null) {
            // 첫 요청
            redisTemplate.opsForValue().set(countKey, "1");
            redisTemplate.opsForValue().set(timestampKey, String.valueOf(now));
            redisTemplate.expire(countKey, windowSeconds, TimeUnit.SECONDS);
            redisTemplate.expire(timestampKey, windowSeconds, TimeUnit.SECONDS);
            return true;
        }
        
        long lastTime = Long.parseLong(lastTimestamp);
        long elapsed = now - lastTime;
        
        if (elapsed > windowSeconds * 1000L) {
            // 시간 윈도우 초과, 리셋
            redisTemplate.opsForValue().set(countKey, "1");
            redisTemplate.opsForValue().set(timestampKey, String.valueOf(now));
            redisTemplate.expire(countKey, windowSeconds, TimeUnit.SECONDS);
            redisTemplate.expire(timestampKey, windowSeconds, TimeUnit.SECONDS);
            return true;
        }
        
        if (currentCount >= maxRequests) {
            return false;
        }
        
        redisTemplate.opsForValue().increment(countKey);
        return true;
    }
}

// 사용 예시
@RestController
public class ChatController {
    
    @Autowired
    private RedisRateLimiter rateLimiter;
    
    @PostMapping("/api/chat/send")
    public ResponseEntity<?> sendMessage(@RequestBody ChatMessage message) {
        String userId = getCurrentUserId();
        
        // 분당 100개 메시지 제한
        if (!rateLimiter.isAllowed("chat_send:" + userId, 100, 60)) {
            return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
                .body("Too many messages. Please slow down.");
        }
        
        // 메시지 전송 로직
        chatService.sendMessage(message);
        
        return ResponseEntity.ok().build();
    }
}
```

---

## 5. 모범 사례 및 주의사항

### 5.1 캐시 전략

#### Cache-Aside Pattern
```java
public Object getData(String key) {
    // 1. Redis에서 조회
    Object cached = redisTemplate.opsForValue().get(key);
    if (cached != null) {
        return cached;
    }
    
    // 2. DB에서 조회
    Object data = database.query(key);
    
    // 3. Redis에 저장
    if (data != null) {
        redisTemplate.opsForValue().set(key, data, 1, TimeUnit.HOURS);
    }
    
    return data;
}
```

#### Write-Through Pattern
```java
public void saveData(String key, Object data) {
    // 1. DB에 저장
    database.save(data);
    
    // 2. Redis에 저장
    redisTemplate.opsForValue().set(key, data, 1, TimeUnit.HOURS);
}
```

#### Write-Behind (Write-Back) Pattern
```java
public void updateData(String key, Object data) {
    // 1. Redis에 먼저 저장
    redisTemplate.opsForValue().set(key, data);
    
    // 2. 비동기로 DB에 저장
    asyncExecutor.execute(() -> {
        database.save(data);
    });
}
```

### 5.2 메모리 관리

**1. TTL 설정:**
```java
// 모든 캐시 데이터에 TTL 설정
redisTemplate.expire(key, duration, timeUnit);

// 기본 TTL 설정
@Bean
public RedisCacheConfiguration cacheConfiguration() {
    return RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofHours(1));
}
```

**2. 메모리 정책 설정 (redis.conf):**
```
maxmemory 2gb
maxmemory-policy allkeys-lru
```

**3. 큰 데이터 저장 시 주의:**
```java
// 큰 리스트는 제한된 크기만 유지
public void addToList(String key, Object value, int maxSize) {
    redisTemplate.opsForList().leftPush(key, value);
    redisTemplate.opsForList().trim(key, 0, maxSize - 1);
}
```

### 5.3 성능 최적화

**1. SCAN 대신 KEYS 사용 금지 (프로덕션):**
```java
// ❌ 잘못된 방법 - KEYS는 Redis를 블로킹함
Set<String> keys = redisTemplate.keys("user:*");

// ✅ 올바른 방법 - SCAN은 비블로킹
Set<String> keys = new HashSet<>();
ScanOptions options = ScanOptions.scanOptions()
    .match("user:*")
    .count(100)
    .build();

try (Cursor<String> cursor = redisTemplate.scan(options)) {
    while (cursor.hasNext()) {
        keys.add(cursor.next());
    }
}
```

**2. Pipeline 사용 (일괄 처리):**
```java
public void batchOperations(List<String> keys, List<Object> values) {
    redisTemplate.executePipelined(new SessionCallback<Object>() {
        @Override
        public Object execute(RedisOperations operations) {
            for (int i = 0; i < keys.size(); i++) {
                operations.opsForValue().set(keys.get(i), values.get(i));
            }
            return null;
        }
    });
}
```

**3. Lua Script 사용 (원자성 보장):**
```java
public Long incrementIfExists(String key, long delta) {
    String script = 
        "if redis.call('exists', KEYS[1]) == 1 then " +
        "    return redis.call('incrby', KEYS[1], ARGV[1]) " +
        "else " +
        "    return nil " +
        "end";
    
    return redisTemplate.execute(
        new DefaultRedisScript<>(script, Long.class),
        Collections.singletonList(key),
        delta);
}
```

**4. 연결 풀 설정:**
```yaml
spring:
  redis:
    lettuce:
      pool:
        max-active: 20      # 최대 연결 수
        max-idle: 10        # 최대 유휴 연결 수
        min-idle: 5         # 최소 유휴 연결 수
        max-wait: 2000ms    # 최대 대기 시간
```

### 5.4 데이터 일관성

**1. DB와 Redis 동기화:**
```java
@Transactional
public void updateMailStatus(String mailId, String status) {
    // 1. DB 업데이트
    mailRepository.updateStatus(mailId, status);
    
    // 2. Redis 캐시 무효화
    redisTemplate.delete("mail_detail:" + mailId);
    
    // 또는 업데이트
    Mail mail = mailRepository.findById(mailId).orElse(null);
    if (mail != null) {
        redisTemplate.opsForValue().set(
            "mail_detail:" + mailId, 
            mail, 
            1, TimeUnit.HOURS);
    }
}
```

**2. 캐시 갱신 전략:**
```java
// 변경 시 즉시 무효화
public void onMailSent(Mail mail) {
    cacheInvalidationService.invalidateMailCache(
        mail.getSenderId(), 
        mail.getId());
}

// 또는 TTL이 짧은 캐시 사용
redisTemplate.opsForValue().set(key, value, 5, TimeUnit.MINUTES);
```

### 5.5 장애 대응

**1. Fallback 로직:**
```java
public List<Mail> getMailList(String userId) {
    try {
        // Redis 조회 시도
        return getMailListFromRedis(userId);
    } catch (Exception e) {
        logger.error("Redis error, falling back to DB", e);
        // Redis 장애 시 DB에서 직접 조회
        return getMailListFromDB(userId);
    }
}
```

**2. Circuit Breaker 패턴:**
```java
@CircuitBreaker(name = "redis", fallbackMethod = "getFallback")
public String getData(String key) {
    return redisTemplate.opsForValue().get(key);
}

public String getFallback(String key, Exception e) {
    logger.warn("Redis circuit breaker activated", e);
    return database.query(key);
}
```

**3. Redis Sentinel/Cluster 사용:**
```yaml
spring:
  redis:
    sentinel:
      master: mymaster
      nodes: 
        - sentinel1:26379
        - sentinel2:26379
        - sentinel3:26379
```

### 5.6 모니터링

**1. Redis 메트릭 수집:**
```java
@Component
public class RedisMetrics {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Scheduled(fixedRate = 60000) // 1분마다
    public void collectMetrics() {
        Properties info = redisTemplate.getConnectionFactory()
            .getConnection()
            .info();
        
        // 메모리 사용량
        String usedMemory = info.getProperty("used_memory_human");
        
        // 연결 수
        String connectedClients = info.getProperty("connected_clients");
        
        // 히트율
        String keyspaceHits = info.getProperty("keyspace_hits");
        String keyspaceMisses = info.getProperty("keyspace_misses");
        
        logger.info("Redis Metrics - Memory: {}, Clients: {}, " +
            "Hits: {}, Misses: {}", 
            usedMemory, connectedClients, keyspaceHits, keyspaceMisses);
    }
}
```

**2. 슬로우 쿼리 로깅:**
```
# redis.conf
slowlog-log-slower-than 10000  # 10ms 이상
slowlog-max-len 128            # 최대 128개 기록
```

### 5.7 보안

**1. 비밀번호 설정:**
```yaml
spring:
  redis:
    password: ${REDIS_PASSWORD}
```

**2. 네트워크 격리:**
```
# redis.conf
bind 127.0.0.1  # 로컬만 허용
```

**3. 민감 데이터 암호화:**
```java
public void saveSensitiveData(String key, String data) {
    String encrypted = encryptionService.encrypt(data);
    redisTemplate.opsForValue().set(key, encrypted, 1, TimeUnit.HOURS);
}
```

---

## 결론

Redis는 다음과 같은 경우에 특히 효과적입니다:

1. **빠른 응답 시간이 필요한 경우** (< 100ms)
2. **읽기가 많은 경우** (읽기:쓰기 = 10:1 이상)
3. **실시간 데이터 처리가 필요한 경우**
4. **세션/캐시 데이터 관리**
5. **분산 환경에서의 동기화**

단, 다음 사항을 주의해야 합니다:

- ❗ **영구 저장이 필요한 중요 데이터는 반드시 DB에도 저장**
- ❗ **TTL을 적절히 설정하여 메모리 관리**
- ❗ **Redis 장애 시 Fallback 로직 구현**
- ❗ **캐시 무효화 전략 수립**
- ❗ **적절한 모니터링 및 알림 설정**

이 가이드를 참고하여 각 기능에 맞는 Redis 활용 전략을 수립하시기 바랍니다.
