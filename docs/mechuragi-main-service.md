# 메추라기 프로젝트 - 김진아 포트폴리오 내용

## 프로젝트 개요

**메추라기(Mechuragi)** - AI 기반 메뉴/식당 추천 커뮤니티 플랫폼

- **기간**: 2024.XX ~ 2025.XX (진행 중)
- **팀 구성**: 4인 (디자이너 1명, 프론트엔드 1명, 백엔드 2명)
- **역할**: Backend Developer
- **GitHub**: [링크]

---

## 핵심 기술 스택

### Backend & Infrastructure
- **Framework**: Spring Boot, Spring Security, JPA/Hibernate
- **Authentication**: JWT, OAuth 2.0 (카카오 로그인)
- **Real-time Communication**: WebSocket (STOMP)
- **Database**: MySQL
- **Cache & Message Queue**: Redis (Pub/Sub)
- **Cloud & Infrastructure**: AWS EC2, S3, CloudFront, SES, Bedrock
- **Email Service**: AWS SES
- **Containerization**: Docker, NGINX
- **CI/CD**: GitHub Actions, Terraform, Ansible

---

## 주요 담당 업무 및 성과

### 1. 인증/인가 시스템 설계 및 구현 (JWT + Spring Security + OAuth 2.0)

**구현 내용:**
- JWT 기반 무상태(Stateless) 인증 시스템 설계 및 구현
- Spring Security를 활용한 접근 제어 및 보안 필터 체인 구성
- 카카오 OAuth 2.0 소셜 로그인 연동
- AWS SES를 활용한 이메일 인증 시스템 구축
- Refresh Token 기반 토큰 갱신 메커니즘 구현

**기술적 의사결정:**
- **JWT 토큰 기반 인증 선택 이유**:
  - RESTful API 구조에 적합한 Stateless 인증
  - 프론트엔드(React Native) 분리 환경에서 효율적
  - 확장성 있는 마이크로서비스 아키텍처 대비

- **Custom Filter 대신 Controller API 방식 채택**:
  - JWT 토큰 기반 환경에서 명시적인 엔드포인트 제공
  - OAuth2와의 통합 용이성
  - 프론트엔드와의 명확한 API 계약

**책임 분리 및 설계 원칙:**
- **단일 책임 원칙(SRP) 적용**: 회원가입 로직을 auth 도메인에서 member 도메인으로 이동
  - `MemberService`: 회원 생성 및 검증 핵심 로직 담당
  - `AuthService`: 인증 관련 후처리(이메일 발송) 담당
- **이중 방어 전략**: 중복 검증을 애플리케이션 레벨(빠른 피드백)과 DB 레벨(동시성 안전) 이중으로 구현

**주요 기능:**
- 일반 회원가입/로그인 (이메일 + 비밀번호)
- 카카오 OAuth 2.0 소셜 로그인
- AWS SES 기반 이메일 인증 (6자리 코드, 30분 유효)
- Access Token + Refresh Token 발급 및 갱신
- 회원 정보 수정, 비밀번호 변경, 회원 탈퇴(소프트 삭제)

**보안 강화:**
- BCrypt를 활용한 비밀번호 암호화
- 이메일/닉네임 중복 체크 (DB Unique 제약조건 + 애플리케이션 검증)
- 계정 상태 관리 (ACTIVE, INACTIVE, SUSPENDED)
- CORS 설정 및 CSRF 보호

**주요 클래스:**
- `JwtTokenProvider`: JWT 토큰 생성, 검증, 파싱
- `JwtAuthenticationFilter`: JWT 기반 인증 필터
- `CustomUserDetailsService`: Spring Security UserDetailsService 구현
- `CustomOAuth2UserService`: OAuth2 로그인 처리
- `OAuth2SuccessHandler`: OAuth2 성공 후 JWT 발급 및 리다이렉트
- `EmailService`: AWS SES 이메일 발송 및 인증 코드 검증

**API 엔드포인트:**
```
POST /api/auth/signup          - 회원가입
POST /api/auth/login           - 로그인
POST /api/auth/logout          - 로그아웃
POST /api/auth/refresh         - Access Token 재발급
POST /api/auth/email/send      - 이메일 인증 메일 발송
POST /api/auth/email/verify    - 이메일 인증 코드 확인
GET  /oauth2/authorization/kakao - 카카오 로그인 시작
```

**성과:**
- 안전하고 확장 가능한 인증 시스템 구축
- 일반 로그인과 소셜 로그인의 원활한 통합
- 이메일 인증을 통한 계정 신뢰성 향상

---

### 2. 실시간 알림 시스템 구축 (Redis Pub/Sub + WebSocket STOMP)

**구현 내용:**
- Redis Pub/Sub을 활용한 분산 환경 메시지 브로커 구현
- WebSocket STOMP 프로토콜을 통한 실시간 양방향 통신
- 투표 종료 및 종료 10분 전 개인화 알림 전송
- 알림 내역 영구 저장 및 읽음 처리 기능
- 스케줄러 기반 자동 알림 발송

**아키텍처 설계:**
```
VoteNotificationScheduler (매 1분 스케줄링)
        ↓
VotePostService (비즈니스 로직 + Redis 발행)
        ↓
Redis Pub/Sub (vote:end, vote:before10min 채널)
        ↓
VoteNotificationSubscriber (메시지 구독 + 역직렬화)
        ↓
VoteNotificationService (STOMP 전송)
        ↓
WebSocket Client (특정 사용자에게만 실시간 전달)
```

**핵심 기술적 구현:**

1. **분산 환경 지원**:
   - Redis Pub/Sub으로 여러 서버 인스턴스 간 이벤트 공유
   - 메시지 직렬화/역직렬화: GenericJackson2JsonRedisSerializer 사용

2. **개인화된 알림 타겟팅**:
   - STOMP의 `/user/{memberId}/queue/vote/end` 구조 활용
   - `VoteNotificationMessageDTO`에 memberId 포함
   - 투표 작성자에게만 선택적 알림 전송

3. **중복 방지 메커니즘**:
   - DB 레벨: `NotificationRepository.existsByMemberIdAndVoteIdAndType()` 체크
   - 투표 레벨: `VotePost.notified10MinBefore` 플래그 활용
   - 스케줄러 1분 주기 실행에도 중복 없음

4. **트랜잭션 안정성**:
   - `@TransactionalEventListener(phase = AFTER_COMMIT)` 사용
   - DB 커밋 후에만 Redis 메시지 발행
   - 트랜잭션 롤백 시 잘못된 알림 방지

5. **알림 설정 존중**:
   - Member 엔티티에 `voteNotificationEnabled` 필드 추가
   - 알림 OFF 사용자는 DB 저장 및 Redis 발행 모두 중단

**주요 기능:**
- 투표 종료 알림 (COMPLETED)
- 투표 종료 10분 전 알림 (ENDING_SOON)
- 알림 목록 조회 (페이징 지원)
- 특정 알림 읽음 처리
- 안 읽은 알림 개수 조회
- 사용자별 알림 설정 ON/OFF

**주요 클래스:**
- `VoteNotificationScheduler`: 매 1분마다 만료/임박 투표 확인
- `VoteEventListener`: VoteCompletedEvent 처리 및 Redis 발행
- `RedisCacheConfig`: Redis Pub/Sub Template 설정
- `RedisSubscriberConfig`: Redis 채널 구독 설정
- `VoteNotificationSubscriber`: Redis 메시지 수신 및 라우팅
- `VoteNotificationService`: STOMP 메시지 전송
- `NotificationService`: 알림 CRUD 및 중복 체크

**API 엔드포인트:**
```
GET   /api/notifications              - 알림 목록 조회
PATCH /api/notifications/{id}/read    - 알림 읽음 처리
GET   /api/notifications/unread-count - 안 읽은 알림 개수
```

**성과:**
- 낮은 지연시간의 실시간 알림 시스템 구축
- 분산 환경에서도 안정적인 메시지 전달
- 사용자 경험 향상 (투표 마감 임박 알림)
- 확장 가능한 이벤트 기반 아키텍처 구현

---

### 3. 실시간 인기 메뉴 집계 시스템 설계

**구현 내용:**
- Hot 투표의 메뉴 옵션을 집계하여 실시간 Top 10 인기 메뉴 제공
- Redis 기반 실시간 데이터 조회 및 투표율 재계산
- 메뉴명 정규화 및 동의어 처리로 정확도 향상
- 다차원 점수 계산 알고리즘 설계

**실시간성 확보:**

**문제점 분석:**
- 기존 `VotePostService.getHotVotes()`는 `@Cacheable`로 5분간 캐싱
- `VoteOption.getVotePercentage()`는 DB의 `participations` 컬렉션 기반
- → 실시간 데이터 반영 불가

**해결 방법:**
1. **캐시 우회**: `vote:hot` Sorted Set을 직접 조회하여 실시간 Hot 투표 목록 확보
2. **Redis 기반 투표율 재계산**:
   ```java
   int totalParticipants = redisTemplate.opsForValue()
       .get("vote:" + voteId + ":participants");
   double realtimePercentage = (double) voteCount / totalParticipants * 100.0;
   ```
3. **1분 TTL 캐싱**: 실시간성(최대 1분 지연)과 성능 균형 확보

**점수 계산 알고리즘:**

다차원 점수 시스템으로 공정하고 정확한 인기도 측정:

```java
최종 점수 = (언급 횟수 × 10.0)          // 여러 투표 등장 횟수
         + (평균 투표율 × 0.01)         // 각 투표에서 선택률
         + (평균 최근성 × 2.0)          // 최근 트렌드 반영
         + 1.0                         // 기본 점수
```

**최근성 계산:**
```java
long daysSinceCreated = ChronoUnit.DAYS.between(vote.getCreatedAt(), now);
double recencyScore = Math.max(0, 1.0 - (daysSinceCreated / 7.0));
```
- 오늘 생성: 1.0 → 7일 전 생성: 0.0
- 최근 트렌드에 더 높은 가중치 부여

**메뉴명 정규화 및 동의어 처리:**

1. **정규화 (Normalization)**:
   - 공백 제거 및 통일
   - 특수문자, 이모지 제거
   - 대소문자 통일
   - 예: "🍜 마라탕!!!" → "마라탕"

2. **동의어 처리 (Synonym Mapping)**:
   - 동일 의미 메뉴 통합: "파스타" ↔ "스파게티", "짜장면" ↔ "자장면"
   - 초기: Map 하드코딩 → 중기: YAML 외부화 → 장기: Admin API 동적 관리

**성능 최적화:**
- Hot 투표 50개 × 평균 옵션 5개 = 250개 옵션 처리
- Redis Pipeline 활용 고려 (여러 키 한 번에 조회)
- 1분 TTL 캐싱으로 응답 시간 ~10ms (캐시 히트) / ~100-300ms (캐시 미스)

**주요 클래스:**
- `PopularMenuService`: 인기 메뉴 집계 및 점수 계산
- `MenuNormalizer`: 메뉴명 정규화 유틸리티
- `MenuScore`: 점수 계산 및 데이터 누적 내부 클래스

**API 엔드포인트:**
```
GET /api/menus/popular - 실시간 인기 메뉴 Top 10 조회
```

**성과:**
- 실시간 데이터 반영으로 정확한 인기 메뉴 제공
- 정규화 및 동의어 처리로 데이터 품질 향상
- 다차원 점수 시스템으로 공정한 순위 산정
- 확장 가능한 집계 시스템 설계

---

### 4. 인프라 구축 및 운영

**담당 업무:**
- AWS EC2 기반 서버 인프라 구축 및 배포
- Docker 컨테이너화 및 NGINX 리버스 프록시 설정
- Redis 서버 구축 및 캐싱/Pub-Sub 설정
- AWS S3, CloudFront를 활용한 정적 파일 배포
- AWS SES 이메일 발송 서비스 연동
- CI/CD 파이프라인 구축 (GitHub Actions, Terraform, Ansible)

**인프라 아키텍처:**
```
Client
  ↓
CloudFront (CDN)
  ↓
NGINX (Reverse Proxy)
  ↓
Spring Boot (Docker Container)
  ↓
├── MySQL (AWS RDS)
├── Redis (Cache + Pub/Sub)
└── AWS S3 (파일 저장소)
```

**주요 기술:**
- **컨테이너화**: Docker를 활용한 일관된 배포 환경 구성
- **로드 밸런싱**: NGINX 리버스 프록시로 트래픽 분산
- **CDN**: CloudFront로 정적 파일 전송 속도 향상
- **IaC**: Terraform으로 인프라 코드화 및 버전 관리
- **자동화**: Ansible로 서버 설정 자동화

**성과:**
- 안정적이고 확장 가능한 인프라 구축
- CI/CD 자동화로 배포 시간 단축
- 인프라 코드화로 재현 가능성 및 유지보수성 향상

---

### 5. 코드 품질 관리 및 팀 협업

**코드 컨벤션 수립:**
- 패키지 구조, 네이밍 규칙, 어노테이션 규칙 등 상세 컨벤션 문서화
- Interface 기반 Service 계층 설계
- DTO ↔ Entity 변환 Mapper 패턴 도입
- 단일 책임 원칙(SRP) 적용

**주요 컨벤션:**
- **Service 계층**: Interface + Impl 분리
- **Mapper 패턴**: `service/mapper` 패키지에 `엔티티명RequestMapper`, `엔티티명ResponseMapper`
- **DTO 어노테이션 규칙**:
  - Request DTO: `@Getter`, `@NoArgsConstructor` (Jackson 역직렬화)
  - Response DTO: `@Getter`, `@Builder`, `@AllArgsConstructor` (빌더 패턴)
- **REST API URL**: 복수형 사용 (`/api/members`, `/api/notifications`)
- **메소드 네이밍**: CRUD 접두사 통일 (get, save, update, delete)

**문서화:**
- 주요 기능별 상세 설계 문서 작성:
  - `auth.md`: JWT 인증 및 OAuth 2.0 구현 계획
  - `notification-redis.md`: 실시간 알림 시스템 아키텍처
  - `popular-menus.md`: 실시간 인기 메뉴 집계 설계
  - `code-convention.md`: 코드 컨벤션 가이드

**성과:**
- 일관된 코드 스타일로 가독성 및 유지보수성 향상
- 명확한 문서화로 팀원 간 소통 효율성 증대
- 확장 가능한 아키텍처 설계

---

## 기술적 도전 과제 및 해결

### 1. 실시간 알림의 중복 발송 방지

**문제:**
- 스케줄러가 1분마다 실행되므로 동일 알림이 여러 번 발송될 위험

**해결:**
- DB 레벨 중복 체크: `NotificationRepository.existsByMemberIdAndVoteIdAndType()`
- 투표 레벨 플래그: `VotePost.notified10MinBefore`
- 트랜잭션 후 발행: `@TransactionalEventListener(AFTER_COMMIT)`

### 2. 분산 환경에서의 실시간 메시지 전송

**문제:**
- 여러 서버 인스턴스에서 알림을 처리할 때 일관성 유지

**해결:**
- Redis Pub/Sub을 메시지 브로커로 활용
- 모든 인스턴스가 동일 채널 구독
- STOMP를 통한 특정 사용자 타겟팅

### 3. 실시간 인기 메뉴의 데이터 정합성

**문제:**
- 캐싱된 데이터와 실시간 데이터 간 불일치

**해결:**
- `vote:hot` Sorted Set 직접 조회 (캐시 우회)
- Redis 기반 투표율 재계산
- 1분 TTL 캐싱으로 실시간성과 성능 균형

### 4. OAuth 2.0과 JWT 통합

**문제:**
- 소셜 로그인과 일반 로그인의 인증 플로우 통합

**해결:**
- `CustomUserDetails`에 `OAuth2User` 인터페이스 추가 구현
- `OAuth2SuccessHandler`에서 JWT 발급 후 프론트엔드로 리다이렉트
- 두 가지 redirect-uri 구분:
  - `spring.security.oauth2.redirect-uri`: 카카오 → 백엔드
  - `oauth2.redirect-uri`: 백엔드 → 프론트엔드

---

## 프로젝트를 통해 배운 점

### 기술적 성장
- **인증/인가**: JWT와 Spring Security의 깊은 이해, OAuth 2.0 통합 경험
- **실시간 통신**: WebSocket STOMP 프로토콜 및 Redis Pub/Sub 활용
- **분산 시스템**: 여러 서버 인스턴스 간 이벤트 공유 및 동기화
- **인프라**: AWS 클라우드 서비스 활용, Docker 컨테이너화, CI/CD 구축
- **성능 최적화**: Redis 캐싱 전략, 실시간성과 성능의 트레이드오프 고려

### 설계 및 아키텍처
- **도메인 주도 설계**: 도메인별 패키지 분리 및 책임 명확화
- **이벤트 기반 아키텍처**: Spring Events와 Redis Pub/Sub을 활용한 느슨한 결합
- **단일 책임 원칙**: Service 계층의 책임 분리 (MemberService vs AuthService)
- **확장 가능한 설계**: 인터페이스 기반 설계, Mapper 패턴 적용

### 협업 및 문서화
- **코드 컨벤션**: 팀 전체가 따를 수 있는 명확한 규칙 수립
- **기술 문서화**: 복잡한 시스템을 이해하기 쉽게 문서화
- **의사결정 기록**: 기술 선택의 이유와 트레이드오프 명시

---

## 프로젝트 성과

- **안전한 인증 시스템**: JWT + OAuth 2.0 기반 확장 가능한 인증/인가 구현
- **실시간 사용자 경험**: WebSocket + Redis를 활용한 저지연 알림 시스템
- **데이터 기반 추천**: 실시간 집계로 정확한 인기 메뉴 제공
- **안정적인 인프라**: AWS 클라우드 기반 확장 가능한 서버 환경 구축
- **높은 코드 품질**: 일관된 컨벤션과 명확한 문서화로 유지보수성 확보

