# Apple App Store Connect — 제출 절차/규격/숨은필수

## 사전 (빌드 전 SKILL.md §0 프리플라이트 먼저)
- 키체인에 **Apple Distribution: <ORG>** 인증서 있는지(`security find-identity -v -p codesigning`). 자동서명+Xcode 로그인 세션이면 App ID·프로파일은 빌드 중 자동생성.
- Bundle ID, Team ID 확인(`grep PRODUCT_BUNDLE_IDENTIFIER / DEVELOPMENT_TEAM ios/Runner.xcodeproj/project.pbxproj`).

## 제출 순서 (이 순서로 해야 막힘 없음)
1. **ASC 앱 레코드 생성** (반드시 업로드 *전에*). ASC → 앱 → ➕ "신규 앱": 플랫폼 iOS, 이름, 기본언어, **번들 ID 드롭다운**(자동서명이 등록해줬으면 뜬다), SKU, 전체 액세스. select들은 native라 `form_input` 또는 JS native-setter+change로 세팅(SKILL §3).
2. **빌드 업로드**: `xcodebuild -exportArchive ... -exportOptionsPlist <destination=upload> -allowProvisioningUpdates`. ~15분 처리.
3. **버전 메타** (배포 inflight 페이지): 프로모션텍스트·**설명**·키워드·지원URL·저작권. ⚠️ 설명에 `■ → 🐾 ·` 같은 특수문자 넣으면 "유효하지 않은 문자"로 저장 거부 → `[ ] - ,` 등 안전문자로.
4. **앱 정보**: 부제(선택)·기본 카테고리·추가 카테고리·콘텐츠 권한(타사콘텐츠 없으면 DOES_NOT_USE).
5. **연령 등급**: 7단계 위저드(라디오). 콘텐츠 항목 전부 NONE, UGC만 예(앱이 사진 공유하면). 가챠=무료면 시뮬도박 NONE/lootBox false. 7단계 "해당 없음"(=어린이용 카테고리 미선택).
6. **App Privacy(앱이 수집하는 개인정보)**: isAnyDataCollected=true → 수집 데이터유형 체크(예: COARSE_LOCATION, PHOTOS_OR_VIDEOS, DEVICE_ID, PRODUCT_INTERACTION, OTHER_USER_CONTENT) → 유형별 설정(용도 체크 + isLinked=false + isTracked=false) → **게시**. PrivacyInfo.xcprivacy와 일치시켜라. 처리방침 URL도 입력.
7. **가격**: 가격 추가 → 기준국가 → $0.00 선택 → 다음/확인 → 저장.
8. **앱 심사 정보**: 연락처(이름/성/전화/이메일=내부전용→개인OK), 메모(가챠무료·환전0·로그인없음·UGC모더레이션), **로그인 필요 OFF**(익명앱).
9. **스크린샷**: iPhone 6.5" 1284×2778 1~3장(+iPad 13" 2048×2732 1장 if 범용). 파일창=SKILL §3/automation.md.
10. **빌드 첨부**: "빌드 추가" → 처리완료된 빌드 선택.
11. **제출**: "심사에 추가" → 우측 "제출 초안(N)" 패널 → **"심사를 위해 제출"**. 되돌릴 수 없는 게시 결정이니 유저 명시 승인 후. 성공 시 "N개의 항목 제출됨" + 상태 "심사 대기 중".

## 자동화 라디오/위저드 패턴 (JS로 빠르게)
라디오 그룹은 `name`별 value로 식별. 안전답변 일괄 클릭 + "다음"까지 한 콜:
```js
function setSafe(){const b={};[...document.querySelectorAll('input[type=radio]')].forEach(r=>{(b[r.name]=b[r.name]||[]).push(r)});
 Object.entries(b).forEach(([n,rs])=>{if(n==='userGeneratedContent')return; (rs.find(r=>r.value==='NONE')||rs.find(r=>r.value==='false')||rs[0]).click()});}
```
연령등급 위저드엔 라디오 없는 "설명/예시" 단계가 끼므로(다음만 누름) 루프로: setSafe → "다음" 클릭 → 700ms 대기 → 반복, "저장" 나오면 종료.

## 심사 후 리젝 (제출은 통과했는데 본심사에서 거부 — 실사고)
- **2.5.1 HealthKit**: 앱이 HealthKit/CareKit API를 쓰는데 **UI에서 그 기능을 명확히 식별 안 하면 거부**. Flutter `health` 플러그인만 넣어도 HealthKit이 링크돼 트리거됨(걸음수 등). 해결 = ①해당 화면에 "Apple 건강(Apple Health/HealthKit)"이라고 명시한 카드/섹션+설명+버튼 추가(은근한 ♥아이콘만으론 부족, 글자로 "Apple 건강" 박아야)→재빌드→재제출, 또는 ②실기기 화면녹화로 회신. 빌드전 점검: health/CareKit 플러그인 있으면 그 UI에 명시 식별 넣었나. (pawcha 2026-06-26 실리젝→walk_screen에 Apple 건강 카드 추가, 빌드+1 재제출로 해결.)
  - ⚠️ **같은 가이드라인으로 2회 리젝되면 "더 잘 설명"이 아니라 "원인 제거"로 전환하라(실사고 2026-06-29).** pawcha는 UI 카드 추가(빌드4)에도 2.5.1 재리젝 — 게임처럼 **앱 주제가 건강/피트니스가 아니면** HealthKit 식별을 아무리 넣어도 심사관 재량 리스크가 남는다. 결정론적 해결 = **HealthKit 프레임워크 자체를 제거**: `health` 패키지 제거 → 걸음수는 의료DB 아닌 모션센서로(iOS **CMPedometer**/Core Motion `queryPedometerData`, Android **TYPE_STEP_COUNTER**+ACTIVITY_RECOGNITION). 2.5.1은 HealthKit 링크 앱에만 적용 → 링크 0이면 심사대상서 빠짐(기능·걸음수 동일). Info.plist NSHealth* 제거, NSMotion 유지. **검증은 바이너리에서**: 빌드된 `.app`에 `otool -L Runner | grep -i health` = 0건(소스 grep만으로 끝내지 말 것). 안드로이드 센서는 history 없음→그날 첫읽기 baseline으로 오늘 걸음 산출.
- **재제출 플로우(콘솔)**: 리젝된 버전 페이지 → 빌드 섹션 기존빌드 "삭제"→"빌드 추가"로 새 빌드(처리완료=TestFlight "제출 준비 완료") 라디오 선택→완료→저장 → "심사 업데이트"→reviewsubmissions 페이지 "앱 심사에 다시 제출" → 상태 "심사 대기 중" 확인. 메타 수정(상표 등)도 같이 저장돼 함께 재심사.
- **메타 상표 동시정리**: description의 "포켓몬/Pokémon"(5.2.1)·"LoL식"(Riot 상표) 같은 3rd파티 상표는 어차피 리젝 벡터 → 재빌드 김에 제거. React textarea는 prototype value setter+input/change 이벤트로 커밋(저장버튼 활성화로 확인).

## 숨은 필수항목 (제출 검증에서야 뜸 — §0에서 예방)
- **ITMS-90683**: 위치 SDK가 always API 참조 → `NSLocationAlwaysAndWhenInUseUsageDescription`(+legacy) 추가, 재빌드.
- **ITMS-91053**: 앱 타겟 PrivacyInfo.xcprivacy 없음 → required-reason API 선언.
- **iPad 13" 스크린샷 필수**: 빌드가 iPad 지원(범용)이라서. 폰앱은 iPhone-only로 빌드하면 사라짐.
- **사용자 이름/암호 필수**: "로그인 필요" 체크 상태 → 익명앱이면 OFF(실클릭으로 커밋).

## 함정
- 같은 디렉토리에서 dev 서버 켠 채 prod export/build 금지(.next/충돌류는 다른 앱 얘기지만, 빌드 산출물 충돌 일반).
- 검증 에러 표시는 stale 가능 → 새로고침으로 서버값 확인(SKILL §4).
- ASC 안 뜨면 콘솔부터(확장 충돌).
