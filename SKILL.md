---
name: app-store-submit
description: >
  Apple App Store Connect / Google Play Console에 모바일 앱(특히 Flutter)을 콘솔 브라우저 자동화(Chrome MCP)+데스크탑 자동화(osascript/cliclick)로 직접 제출/출시하는 플레이북. 스토어 제출·심사 제출·앱 출시·.ipa/AAB 업로드·스크린샷 업로드·ASC 앱레코드 생성·연령등급/개인정보(App Privacy)/가격/심사정보 입력·리젝 방지를 다룬다. "앱스토어 올려/제출해", "플레이스토어 출시", "App Store Connect", "Play Console", "심사 제출", "스크린샷 업로드", "ipa 업로드", "스토어 등록정보" 같은 말이 나오면 반드시 먼저 이 스킬을 펼쳐서 ①빌드 전 프리플라이트 ②자산 규격 ③검증된 자동화 기법(특히 macOS 파일창)을 적용하라. 콘솔 폼 입력/네이티브 파일창/SPA 렌더 문제를 맨몸으로 시도하다 토큰 태우지 말 것.
---

# 모바일 스토어 제출 — 검증된 플레이북

이전에 pawcha 양 스토어 제출을 완주하며 **파일창 자동화 15회+, 막판 재빌드, 콘솔 진단 헤맴**으로 막대한 토큰을 태웠다. 그 함정들을 순서대로 막는 게 이 스킬의 전부다. **순서가 곧 효율**: ①빌드 전 프리플라이트 → ②자산 정확히 → ③검증된 기법으로 콘솔 입력.

> 상세: `references/automation.md`(자동화 기법·파일창), `references/apple.md`(ASC 절차·규격·숨은필수), `references/play.md`(Play 절차). 작업 시작 시 `references/automation.md`부터 읽어라 — 거기서 제일 많이 막혔다.

---

## 0. 빌드 전 프리플라이트 (가장 비싼 실수 = 막판 재빌드. 빌드 *전에* 막아라)

바이너리를 빌드하기 *전에* 아래를 확정한다. 이걸 안 하면 "제출 검증"에서야 숨은 필수항목이 떠서 재빌드→재업로드→재처리(30분+) 사이클이 돈다.

- [ ] **기기 패밀리**: 폰 전용 앱이면 `TARGETED_DEVICE_FAMILY = 1`(iPhone only)로 빌드. 범용(1,2)이면 **iPad 스크린샷이 강제**된다. 카메라/만보기 같은 폰 기능 앱은 거의 iPhone-only가 맞다. (정설: 바이너리의 UIDeviceFamily로 ASC가 요구 스샷을 정함. ⚠️ **이 경로는 pawcha 세션에선 미실행** — 당시엔 iPad 스샷 업로드로 우회함. iPhone-only로 처음 빌드하는 게 더 깔끔하나, 처음 적용 시 1회 검증.)
- [ ] **권한 사용설명 전부**: 코드가 참조하는 모든 권한에 `NSxxxUsageDescription`. 특히 위치 SDK(geolocator 등)는 *always* API를 참조해서 `NSLocationAlwaysAndWhenInUseUsageDescription`이 없으면 **ITMS-90683**. (Camera/Photo/Motion/Health/Microphone도 점검)
- [ ] **PrivacyInfo.xcprivacy**(앱 Runner 타겟에 추가, 단순 Pods 말고): required-reason API(UserDefaults CA92.1, FileTimestamp C617.1, DiskSpace E174.1, BootTime 35F9.1) 미선언 시 **ITMS-91053**.
- [ ] **`ITSAppUsesNonExemptEncryption=false`** (Info.plist) — 표준 HTTPS만이면. 없으면 업로드마다 수출규정 질문.
- [ ] **버전/빌드번호 증가** — 재업로드 시 빌드번호 +1 필수(중복 거부).
- [ ] 익명 앱이면 ASC "로그인 필요"는 **끌 것**(데모계정 username/password 강제 회피).

검증: 빌드 산출물에서 `unzip -l *.ipa | grep PrivacyInfo`(Runner.app 루트), `plutil -p Payload/Runner.app/Info.plist | grep -iE "Encryption|UsageDescription"`로 실제 반영 확인.

---

## 1. 자산 규격 (정확히 — 틀리면 재생성)

- **iPhone 6.5" 스크린샷**: `1284×2778` 또는 `1242×2688` (둘 다 6.5/6.7 호환은 1284×2778). 1290×2796은 6.5 슬롯에서 거부.
- **iPad 13" 스크린샷**: `2048×2732`(12.9") 또는 `2064×2752`(13"). iPad 지원 빌드면 최소 1장 필수.
- **아이콘**: .ipa에 1024 임베드되면 ASC가 자동추출(별도 업로드 불필요). Play는 512 아이콘+1024×500 피처그래픽 별도.
- 소스 1장에서 규격별로 **fit+pad**(배경색=가장자리 픽셀)로 PIL 일괄 생성. `references/automation.md`의 스니펫.

---

## 2. 빌드/업로드 (Apple)

- iOS 빌드는 이 맥에서 **된다**. "MLKit arm64 = 빌드불가"는 오해(arm64 경고는 *시뮬레이터* 한정). `flutter build ipa --release`로 기기/배포 archive 정상.
- **업로드 = API키/비번 없이 Xcode 세션으로**: `xcodebuild -exportArchive -archivePath ...xcarchive -exportOptionsPlist <plist with destination=upload, method=app-store-connect, teamID> -allowProvisioningUpdates`. Xcode 로그인 세션을 그대로 쓴다.
- ⚠️ **ASC 앱 레코드가 먼저 있어야** 업로드 성공(없으면 `missingApp` exit 70). 그러니 ASC에서 "신규 앱" 먼저 만들고 → 업로드 → 처리(~15분) → 빌드 첨부.
- dSYM 업로드 경고(서드파티 프레임워크)는 비차단.

---

## 3. 콘솔 입력 — 검증된 자동화 기법 (작업 전 `references/automation.md` 정독)

이번에 제일 많이 막힌 지점. 맨몸으로 시도 금지.

- **macOS 네이티브 파일창(스샷/에셋 업로드)**: file_upload MCP(세션공유 파일만)·request_directory(unsupervised 차단)·goto 타이핑(자동완성이 먹음)·goto AX-set(불안정) 전부 **사장길**. **정답 = cliclick 좌표 캘리브레이션**: `cliclick m:<logical>` 후 computer-use `cursor_position`으로 스크린샷 좌표를 받아 스케일 역산(**로지컬 = 스크린샷 × (실제로지컬폭/스크린샷폭), 보통 ×1.0714**). 폴더는 cliclick **더블클릭**으로 진입, 파일은 **cmd+클릭 다중선택** 후 **선택된 항목을 더블클릭**해 확정(열기 *버튼* 단일클릭은 cliclick이 안 먹힘). 사이드바 행은 AXOpen으로도 됨. → 전체 절차 `references/automation.md`.
- **React 컨트롤드 입력**: select/checkbox/긴 textarea는 JS `.value=`/`.click()`로 화면값만 바뀌고 **저장 안 됨**(저장버튼 비활성·검증실패·값 리셋). **실제 키스트로크/실클릭**(Chrome MCP `computer` type/left_click)으로 onChange 발화. 짧은 필드는 JS-set 후 **"문자 1개 입력→백스페이스"**로 트리거 가능.
- **입력 전 항상 Cmd+A → Delete**: 기존값 있는 필드에 그냥 타이핑하면 "명근명근"처럼 **이어붙는다**.

---

## 4. 진단 룰 (헤매지 말고 바로 적용)

- **SPA 본문이 빈 화면/렌더 안 됨** → 가시성 게이팅이라 단정 말고 **`read_console_messages` 먼저**. 크립토 지갑 등 브라우저 확장이 inapp.js를 주입해 모듈로더를 깨뜨린다(`i.memoize is not a function`류). → 그 확장 끄기(내가 못 끄면 유저에게 정확히 그 확장명 지정).
- **저장 버튼이 계속 비활성 / 값이 저장 후 되돌아감** → React 미커밋. JS-set 말고 실입력(위 §3).
- **"제출" 눌렀더니 에러가 떴다** → 그 에러 표시가 **stale일 수 있다**. 새로고침(navigate reload)으로 **서버 실제값**부터 확인. (실제로 "심사에 추가"는 에러 떠도 이미 성공해 있었음.) helper 텍스트·탭 라벨을 에러로 오탐하지 말 것.
- **MCP 탭이 백그라운드 창이라 파일창이 안 보임** → `osascript`로 해당 URL 가진 Chrome 창을 `set index ... to 1` + activate로 앞으로.

---

## 5. PII / 공개정보 규칙

- **스토어에 공개되는 필드(저작권·판매자명 등)에는 유저 개인 이름 금지 → 회사명**((주)X / X INC.).
- **Apple App Review 연락처(이름/전화/이메일)는 내부 전용**이라 개인정보 OK. 입력 전 "이게 스토어 공개냐 vs 콘솔 내부전용이냐" 구분.

---

## 6. ⛔ 제출 전 필수 GATE — 전 리젝벡터 적대적 전수 점검 (인용된 사유만 고치고 끝내지 마라)

**심사관은 "고친 2개"가 아니라 앱 전체를 본다. 리젝 사유 N개를 고쳤어도, 제출 *전에* 아래 전 벡터를 코드/콘솔로 직접 확인하고 통과해야 "완료"다. (사고 2026-06-29: 2.5.1·2.3.10만 고쳐 재제출 후, 1.2 UGC 신고/차단 미구현을 유저가 "심사관 돼서 봐"라고 시켜서야 발견 — 제출 전 grep 5분이면 잡혔다.)**

- **1.2 UGC (가장 흔히 놓침)**: 앱이 *남의* 사용자 콘텐츠(사진/글/카드)를 보여주면(피드·가챠풀·거래·공유) 1.2 풀적용 → **①업로드 필터 ②콘텐츠 신고 ③유저 차단 ④연락처**가 **UI에 실제로 동작**해야 함. `grep -rniE "report|block|신고|차단"`으로 **"준비 중"·TODO·placeholder가 아니라 실제 호출되는지** 확인. **메모에 "신고/차단 있음"이라 쓰려면 진짜 있어야** — 없는데 쓰면 심사관이 모순 잡아 더 큰 리젝.
- **2.1 실행**: 리뷰 기기에서 끝까지 도나? 익명/로그인 진입, 카메라/사진, 핵심기능(가챠 등) 크래시 없나. 실기기/TestFlight 1회 통과 권장(시뮬만 검증=불충분).
- **3.1.1 IAP**: `in_app_purchase` 링크됐는데 상품 0이면 무해하나, **깨진/숨은 구매 UI 없나** 확인. 포인트(앱내화폐) 거래는 실결제 아님 → IAP 아님.
- **4.8 Sign in with Apple**: Google/3rd파티 로그인을 노출하면 **Apple 로그인도 같이** 노출 필수. (익명/로그인 없음이면 미적용.)
- **5.1 App Privacy**: 콘솔 "앱이 수집하는 개인정보" + PrivacyInfo.xcprivacy + Play 데이터안전성 **3개 일치** + 실제 수집(로그인 이메일·사진·위치·deviceID)과 일치. **방법(놓치기 쉬움)**: 라이브 SDK를 코드로 *전수 grep*(`logEvent`/`logAppOpen`=Usage Data·DeviceID, `getToken`/FCM=Identifiers, `getCurrentPosition`=Location, 사진/카메라=Photos) → 그 수집목록을 위 3곳 선언과 1:1 대조. SDK가 pubspec에 링크돼 *실행*되면 "선언 안 함"은 5.1.1(c) 메타데이터 리젝. ⚠️ 일관성 함정 2개: ①PrivacyInfo `NSPrivacyTracking`/항목별 `Tracking`이 `true`인데 ATT 프롬프트 없으면 모순(=리젝) — 추적 안 하면 전부 `false`여야. ②위치 `LocationAccuracy`가 best/high면 Precise인데 매니페스트가 CoarseLocation이면 불일치 — 코드 정확도와 선언을 맞춰라. (analytics가 `logEvent` 라이브인데 ASC 라벨 미선언이 흔한 누락.)
- **연령등급/loot box**: 가챠가 **무료(현금구매 불가)**면 시뮬도박 아님(loot box는 "for purchase" 정의). 아동지향 회피=Made-for-Kids 미선택.
- **스크린샷 2.3.10**: OS 상태바·기기 정확. 안드로이드 캡처에 iOS바 합성은 Flutter(크로스플랫폼 동일 렌더)에서만 안전선 — Cupertino 위젯 쓰면 실 iOS 캡처 권장.
- **Play 유력사유**: AAB 민감권한(FINE_LOCATION/RECORD_AUDIO)이 데이터안전성 미선언과 불일치 → 권한정리 재빌드.

**GATE 통과 = 위 벡터 전부 ✓ 또는 "해당없음" 근거. 하나라도 불확실하면 제출 전 해소.** 통과 후 §3 절차로 제출.

---

원본 서사/디테일: `~/Desktop/Evanwiki/하네스/로그/pawcha-스토어-제출.md`. 양 스토어 전 단계 체크리스트는 그 프로젝트의 `docs/LAUNCH-TODO.md` 패턴 참고.
