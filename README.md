# app-store-submit

**Apple App Store / Google Play 심사 사전 검사 + 제출 자동화 Claude Code 스킬**
A [Claude Code](https://claude.com/claude-code) skill that pre-checks App Store Review Guideline reject vectors and automates App Store Connect / Google Play Console submission (browser + desktop automation).

앱을 제출하기 *전에* 리젝 사유를 먼저 잡아내는 게 목적입니다. 스크린샷 규격, 숨은 필수 항목(ITMS-90683/91053), Privacy 선언 불일치, UGC 신고/차단 미구현(1.2), HealthKit 오용(2.5.1) 같은 흔한 리젝 벡터를 코드/콘솔에서 직접 grep·확인하는 체크리스트와, 실제로 검증된 macOS 네이티브 파일창 자동화 기법(cliclick 좌표 캘리브레이션)까지 포함합니다.

## 왜 필요한가

앱스토어 제출은 한 번 삽질하면 비싸게 삽질합니다 — 막판 재빌드, 파일창 자동화 실패, "리젝 사유만 고치고 재제출했다가 다른 사유로 또 리젝" 같은 패턴이 반복됩니다. 이 스킬은 실제 앱 제출/재제출 경험에서 나온 체크리스트와 기법을 순서대로 정리한 것입니다:

1. **빌드 전 프리플라이트** — 재빌드가 가장 비싼 실수이므로 빌드 *전에* 막습니다 (기기 패밀리, 권한 설명, PrivacyInfo.xcprivacy, 수출규정 등)
2. **자산 규격** — 스크린샷/아이콘 정확한 픽셀 사이즈
3. **검증된 콘솔 자동화 기법** — React 컨트롤드 입력, macOS 네이티브 파일창(NSOpenPanel) cliclick 좌표 캘리브레이션
4. **진단 룰** — SPA 렌더 실패, stale 에러 표시 등 흔한 함정
5. **PII/공개정보 규칙** — 스토어 공개 필드 vs 내부 전용 필드
6. **⛔ 제출 전 필수 GATE** — 리젝 사유 전 벡터(1.2 UGC, 2.1 실행, 3.1.1 IAP, 4.8 Sign in with Apple, 5.1 App Privacy, 연령등급, 스크린샷, Play 데이터안전성)를 코드 grep으로 하나씩 대조하는 적대적 체크리스트

## 키워드 / 트리거

App Store Connect, Google Play Console, 앱스토어 제출, 앱 심사, 심사 제출, 앱 출시, 리젝 방지, app store rejection, app review guideline, .ipa 업로드, AAB 업로드, 스크린샷 업로드, ASC 앱 레코드 생성, 연령등급, App Privacy, PrivacyInfo.xcprivacy, ITMS-90683, ITMS-91053, HealthKit 2.5.1, UGC 1.2, Sign in with Apple 4.8, Flutter 앱 출시, cliclick 파일창 자동화

## 설치

Claude Code를 쓴다면 이 저장소를 스킬 디렉토리에 클론하세요:

```bash
git clone https://github.com/bugbug9999/app-store-submit.git ~/.claude/skills/app-store-submit
```

프로젝트 전용으로 쓰려면 `.claude/skills/app-store-submit/`에 클론해도 됩니다.

설치 후 Claude Code에서 "앱스토어 제출해줘", "심사 넣어줘", "스크린샷 업로드", "App Store Connect" 같은 말을 하면 자동으로 이 스킬이 로드됩니다.

## 구성

- `SKILL.md` — 전체 플레이북 (프리플라이트 → 자산 → 콘솔 입력 → 진단 → PII → 제출 전 GATE)
- `references/apple.md` — App Store Connect 상세 절차, 숨은 필수 항목, 심사 후 리젝 사례(HealthKit 2.5.1 등)
- `references/automation.md` — macOS 네이티브 파일창 cliclick 좌표 캘리브레이션, React 컨트롤드 입력 우회
- `references/play.md` — Google Play Console 절차/함정

## 주의

이 스킬은 브라우저(Chrome DevTools MCP류)와 macOS 데스크탑 자동화(`cliclick`, `osascript`)를 함께 씁니다. `cliclick`은 `brew install cliclick`로 설치하세요. Apple/Google 계정 로그인 세션이 필요합니다(API 키 없이 Xcode/브라우저 세션 재사용).

**"심사를 위해 제출"은 되돌릴 수 없는 액션입니다.** 실제 제출 전 반드시 사람이 최종 확인하세요.

## 라이선스

MIT
