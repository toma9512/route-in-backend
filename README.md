# ⚙ Route-In Backend

> Route-In 서비스의 백엔드 서버

Spring Boot 기반 REST API + WebSocket을 통해 채팅, 알림, 게시글, 러닝코스, AI 추천 기능을 제공합니다.

🔗 [Frontend 레포](https://github.com/toma9512/route-in-frontend)　|　🔗 [배포 주소](https://routein.store)

---

## 🛠 Tech Stack

| 구분 | 기술 |
| --- | --- |
| Framework | Spring Boot |
| ORM | MyBatis |
| Database | MySQL |
| 실시간 통신 | WebSocket (STOMP) |
| 인증 | JWT (Access / Refresh Token) · Spring Security · OAuth2 |
| Infra | Docker · Nginx · GCP · GitHub Actions |

---

## 📌 본인 담당 기능

> 3인 팀 **팀장**으로 참여, 백엔드 설계·구현 전반 담당

- JWT 인증 (Access / Refresh Token 전략)
- WebSocket 실시간 채팅 · Presence 상태 관리
- 알림 시스템 · unread count API
- Kakao Map 러닝코스 CRUD
- 커서 기반 페이징 API (채팅 메시지 · 게시글)
- 일정 관리 및 역할 분담 주도

---

## ⭐ 주요 기능

### 💬 실시간 채팅
- WebSocket (STOMP) 기반 메시지 송수신
- `room_read_tb` 기반 사용자별 읽음 상태 관리
- `last_read_message_id` 기준 unread count 계산
- `message_id + create_dt` 복합 커서 기반 무한스크롤 페이징
- Presence(접속 상태) 서버 기준 관리 — `ENTER_ROOM / LEAVE_ROOM` 이벤트

### 🔔 알림
- Notification DB 저장 + WebSocket 동시 Push
- 사용자별 `/queue` 경로 개별 전송
- 활성 채팅방 사용자 알림 자동 제외

### 📰 게시글
- 운동루틴 · 러닝코스 게시글 CRUD
- 댓글 / 대댓글 (`parent_id` 계층형 구조)
- 1인 1추천 (DB UNIQUE 제약으로 중복 방지)
- 태그 · 지역 · 거리 기반 필터 검색 (MyBatis 동적 SQL)
- 커서 기반 무한스크롤 페이징

### 🗺 러닝코스
- Kakao Map 기반 코스 저장 · 조회 · 수정 · 삭제
- 거리 자동 계산 · 즐겨찾기
- 게시글 코스 복사 후 개인 데이터 분리 저장

### 📅 주간 루틴
- 요일별 루틴 등록 · 수정
- 체크 완료 상태 서버 동기화

### 🤖 AI 운동 추천
- 날씨 API + 사용자 운동 기록 기반 프롬프트 생성
- 질문 / 답변 DB 저장 및 히스토리 관리
- 실패 시 fallback 처리

### 📊 인바디
- 날짜별 체중 · 골격근량 · 체지방 기록

### ✅ 출석
- `user_id + date` UNIQUE 제약으로 하루 1회 강제
- 서버 기준 날짜 처리

---

## 🔑 기술적 의사결정

### 1. 실시간 채팅 — WebSocket(STOMP) 선택

| 방식 | 검토 내용 |
| --- | --- |
| HTTP Polling | 구현 단순하나 불필요한 요청 반복, 서버 부하 증가 |
| SSE | 서버→클라이언트 단방향만 가능 — 채팅에 부적합 |
| **WebSocket (STOMP)** ✅ | 양방향 지속 연결, `/pub` · `/sub` 엔드포인트 분리로 채팅방별 구독 구조화 |

채팅은 송·수신이 동시에 일어나는 양방향 통신이 필수였기 때문에 SSE와 Polling은 고려 대상에서 제외했습니다.

---

### 2. 채팅 메시지 페이징 — 커서 기반 선택

| 방식 | 검토 내용 |
| --- | --- |
| Offset 기반 | 구현 단순하나 데이터 증가 시 SKIP 비용 선형 증가 |
| **커서 기반 (create_dt + message_id)** ✅ | 인덱스 범위 스캔으로 데이터 증가와 무관하게 일정한 조회 성능 유지 |

`create_dt` 단독 커서는 동일 시각 메시지가 복수일 때 정렬이 불안정합니다.
`message_id`를 보조 커서로 추가해 **정렬 안정성과 성능을 동시에 확보**했습니다.

```sql
WHERE (create_dt < #{lastCreateDt})
   OR (create_dt = #{lastCreateDt} AND message_id < #{lastMessageId})
ORDER BY create_dt DESC, message_id DESC
LIMIT #{pageSize}
```

---

### 3. 읽음 상태 관리 — room_read_tb 분리 설계

| 방식 | 검토 내용 |
| --- | --- |
| message 테이블에 is_read 컬럼 추가 | 사용자 수 증가 시 메시지마다 UPDATE 비용 급증 |
| **room_read_tb 분리** ✅ | 채팅방별 last_read_message_id만 관리 → unread count를 단순 COUNT 쿼리로 처리 |

메시지와 읽음 상태의 생명주기를 분리해 **데이터 정합성과 실시간성을 동시에 만족**하는 구조를 설계했습니다.

```
room_read_tb
├── room_id
├── user_id
└── last_read_message_id

unread count = COUNT(*) WHERE message_id > last_read_message_id
```

---

### 4. Presence 상태 — 서버 기준 관리

클라이언트 기준으로 상태를 관리하면 탭 닫기·새로고침·중복 접속 상황에서 엣지 케이스가 발생합니다.
`CONNECT / DISCONNECT` 이벤트로 online user를 서버에서 관리하고, `ENTER_ROOM / LEAVE_ROOM` 이벤트를 추가해 `userId → activeRoomId` 매핑을 서버가 보유하도록 설계했습니다.

```
서버가 반드시 알고 있어야 할 상태
├── Connection State  (연결 여부)
├── User State        (온라인 여부)
└── Room State        (현재 활성 채팅방)
```

이 중 하나라도 없으면 알림 중복 · unread 오류 · 1인방 버그 · 재접속 문제가 연쇄적으로 발생합니다.

---

## 🔍 트러블슈팅

### 1. unread count 불일치

**문제** : 채팅방 입장 시 unread 개수가 실제와 다르게 표시

**원인** : `room_read` 구조가 마지막 읽음 기준을 안정적으로 표현하지 못해 메시지 정렬·페이징·재요청 타이밍에 따라 누락/중복 발생

**해결** : `last_read_message_id` 기준으로 구조 변경 + `message_id + create_dt` 복합 커서 적용 → 타이밍과 무관하게 항상 정확한 값 반환

---

### 2. 채팅방에 있어도 알림이 계속 뜨는 문제

**문제** : 사용자가 이미 채팅방 화면에 있는데도 알림 발생

**원인** : 서버가 사용자의 현재 활성 채팅방을 알지 못하는 구조

**해결** : `ENTER_ROOM / LEAVE_ROOM` 이벤트 추가 → 서버에서 `userId → activeRoomId` 매핑 관리 → 메시지 전송 시 activeRoomId와 비교해 알림 분기 처리

---

### 3. 멀티 디바이스 팝업 동기화 오류

**문제** : A기기에서 팝업을 닫아도 B기기에서 다시 뜨는 현상

**원인** : `localStorage`는 브라우저(기기) 단위 저장이라 기기 간 동기화 불가

**해결** : `attendance_tb`에 `popup_shown` 컬럼 추가 → `userId + attendance_date` 기준으로 서버가 단일 기준 판단

**배운 점** : 멀티 디바이스 환경에서 사용자 상태는 반드시 서버(DB)가 기준이어야 일관성 유지 가능

---

### 4. 채팅방에 혼자 있을 때 메시지 갱신 안 되는 문제

**문제** : 1인 채팅방에서 새 메시지 갱신 로직이 동작하지 않음

**원인** : 클라이언트 갱신 트리거를 상대방 이벤트 수신에 의존

**해결** : 서버 기준으로 "메시지 저장 성공" 시점에 방 토픽으로 갱신 이벤트를 항상 발행 → 클라이언트는 참가자 수와 무관하게 이벤트 처리

---

## 🗄 Database

### 주요 테이블

| 테이블 | 설명 |
| --- | --- |
| user_tb | 사용자 정보 · OAuth2 · 팔로우 수 |
| room_tb · room_participant_tb | 채팅방 · 참가자 관리 |
| room_read_tb | 사용자별 마지막 읽은 메시지 ID |
| message_tb | 채팅 메시지 |
| notification_tb | 알림 저장 |
| board_tb | 게시글 (루틴 · 러닝코스) |
| comment_tb | 댓글 / 대댓글 (parent_id 계층형) |
| recommend_tb | 게시글 추천 (UNIQUE 제약) |
| course_tb · course_point_tb | 러닝코스 경로 · 좌표 |
| routine_tb | 요일별 운동 루틴 |
| ai_question_tb · ai_recommend_tb | AI 질문·답변 이력 |
| in_body_tb | 인바디 기록 |
| attendance_tb | 출석 체크 |
| follow_tb | 팔로우 관계 |
