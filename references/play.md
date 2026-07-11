# Google Play Console — 제출 절차/함정

## 흐름 (App Content 선언 → 스토어 등록정보 → 프로덕션 릴리스 → 제출)
1. **앱 생성**: 이름/무료/게임 or 앱/패키지명.
2. **App Content(앱 콘텐츠) 선언 전부** — "주의 필요 0건" 만들 때까지:
   - 콘텐츠 등급(IARC) 설문 — 저장 먼저 눌러야 "다음" 활성화되는 경우 있음.
   - 타겟층 및 콘텐츠 — 아동지향 회피하려면 13세+만 선택, Designed for Families 미가입.
   - 데이터 안전성 — 수집항목을 AAB 권한과 **일치**시켜라(불일치=리젝). 삭제 URL, 익명(계정생성 없음) 명시.
   - 광고 ID(애널리틱스 AD_ID 자동병합이면 예), 정부앱/금융/건강 선언.
3. **카테고리 + 연락처**(이메일/웹사이트).
4. **스토어 등록정보(그래픽)**: 512 아이콘 + 1024×500 피처그래픽 + 스크린샷. 파일 업로드는 네이티브 파일창 → SKILL §3 / automation.md.
5. **프로덕션 릴리스**: AAB 업로드(파일창, 크기 무제한) → 출시명 + 출시노트(en/ko) → **출시국=대한민국** 등 + 무료.
6. **제출/검토 전송**: 게시 개요 → "검토를 위해 변경사항 전송" / 프로덕션 트랙 "출시 검토". 되돌릴 수 없는 게시라 유저 승인 후. 성공 시 "출시 버전 1(1.0.0) 검토 중".

## 함정 (검증됨)
- **Play Console SPA 폼 라우트는 `document_idle`를 못 채워**(상시 폴링) screenshot/read_page **45s 무한실패** → **새 MCP 탭 생성 후 navigate → 8s 대기 → screenshot**이 유일 안정 경로. 기존 탭이 degraded되면 새 탭으로.
- form 라우트 **직접 deep-link URL은 app-list로 리다이렉트** → overview에서 "선언 시작" 클릭으로 SPA 진입.
- 다이얼로그 막 열린 직후 클릭 미등록(렌더 전) → 1번 더 클릭.
- "수집됨" 체크 시 하위질문(임시처리/필수/목적)이 펼쳐져 좌표 시프트 → 펼친 후 재좌표.
- "저장" 버튼이 find로 "삭제(discard)" 버튼에 잘못 잡힐 수 있음 — 파란 "저장"은 footer 맨우측 좌표로 직접. discard 다이얼로그 뜨면 "취소"(keep).
- 이미 "검토 중"이면 **중복 제출 금지** — 프로덕션 트랙 상태("검토 중")부터 확인.

## 파일 업로드(AAB·그래픽)
Play도 네이티브 파일창이라 SKILL §3 / automation.md의 cliclick 캘리브레이션이 동일하게 통한다. (예전엔 AX search 필드 set value+AXConfirm 경로도 썼지만, cliclick 좌표 캘리브레이션이 더 일반적·안정적.)

## 리젝 유력사유
AAB merged manifest의 민감권한(ACCESS_FINE_LOCATION/RECORD_AUDIO/READ_PHONE_STATE 등 플러그인 transitive)이 데이터안전성 선언과 불일치. → `tools:node="remove"`로 불필요 권한 제거 후 재빌드(단 런타임 의존 실기기 1회 검증), AD_ID 제거 시 광고ID 선언도 수정.
