# Copilot Chat Backup

Copilot Chat 대화를 이슈로 제출하면, GitHub Actions가 매일 자동으로 Markdown 파일로 백업하고 이슈를 닫습니다.

## 사용 방법
1. Copilot Chat에서 중요한 Q/A를 복사합니다.
2. 저장소의 [New issue]를 만들 때 템플릿 "Copilot Chat Backup"을 선택합니다.
3. 질문/답변을 붙여넣고 제출합니다. 라벨 `copilot-chat`이 자동 부여됩니다.
4. 매일 03:00 UTC에 워크플로가 돌아가며, 이슈 내용을 `backups/YYYY/MM/DD/...md`로 저장하고 이슈를 닫습니다.
   - 필요 시, Actions 탭에서 `Run workflow`로 수동 실행도 가능합니다.

## 백업 파일 구조
```
backups/
  YYYY/
    MM/
      DD/
        YYYY-MM-DD-<제목슬러그>-#<이슈번호>.md
```

각 파일 상단에는 이슈 메타데이터(YAML Front Matter)가 포함되어 있고, 본문에는 이슈 내용 전체가 저장됩니다.

## 문서 (Documentation)

이 저장소에는 백업된 Q&A 외에도 유용한 기술 문서가 포함되어 있습니다:

- **[Redis 성능 최적화 가이드](docs/redis-optimization-guide.md)** - 이메일 및 채팅 시스템에서 Redis를 활용한 성능 최적화 전략
  - 이메일 관련 기능 (임시저장, 조회, 예약/알림, 수신확인, 첨부파일)
  - 채팅 관련 기능 (메시지 전송/수신, 파일 첨부, 알림, 상세조회)
  - 구체적인 Spring Boot 구현 예시
  - 모범 사례 및 주의사항

- **[Redis 빠른 참조 가이드](docs/redis-quick-reference.md)** - Redis 활용 패턴의 핵심 내용 요약

## 주의사항
- 워크플로는 라벨 `copilot-chat`이 붙은 오픈 이슈만 처리합니다.
- 처리된 이슈에는 `archived` 라벨이 추가되고 자동으로 닫힙니다.
- 저장소 권한은 Actions가 푸시할 수 있도록 `contents: write`, `issues: write`로 설정되어 있습니다.

