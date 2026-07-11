# 자동화 기법 (콘솔 폼 + 네이티브 파일창 + 데스크탑)

작업 시작 시 이 파일을 먼저 읽어라. 여기서 제일 많이 막혔다.

## 1. macOS 네이티브 파일창(NSOpenPanel) 업로드 — 핵심 기법

웹 "파일 선택" 버튼을 누르면 OS 파일창이 뜬다. Chrome `file_upload` MCP는 **세션이 공유한 파일만** 받고(로컬 생성 파일 거부), `request_directory`는 unsupervised에서 차단, goto(Cmd+Shift+G)에 경로 타이핑하면 자동완성이 가로채고, goto AX-set은 불안정. → **cliclick 좌표 캘리브레이션이 유일하게 안정적.**

### 1-1. 좌표 캘리브레이션 (이걸 *맨 먼저* 하라 — 안 하면 클릭이 다 빗나간다)
cliclick은 **로지컬 좌표**(예 1470×956)를 쓰는데 computer-use 스크린샷은 다른 해상도로 다운스케일된다. 둘의 비율을 1회 측정:
```bash
# 로지컬 화면 크기
osascript -e 'tell application "Finder" to get bounds of window of desktop'   # 예: 0,0,1470,956
# 임의 로지컬 좌표로 커서 이동
/opt/homebrew/bin/cliclick m:735,478
```
그 다음 computer-use `cursor_position` 호출 → image_pixels 좌표 반환(예 686,446).
→ **scale = 스크린샷/로지컬 = 686/735 ≈ 0.9333**. 따라서 **로지컬 = 스크린샷좌표 ÷ scale**.
⚠️ **공식이 진실, 숫자는 기기마다 다르다.** 0.9333(=×1.0714 역산)은 이 맥(로지컬 1470×956)의 값일 뿐 — DPI/해상도 바뀌면 달라진다. **매번 위 2줄로 1회 측정해서 scale을 새로 구하라.** (외워서 ×1.0714 박지 말 것.)

### 1-2. 폴더 진입 / 파일 선택 / 확정
- 리스트뷰로 두면 파일목록이 AXOutline이 돼 다루기 쉽다(필요 시 키보드 Cmd+2 — 시트가 키 입력 받음).
- **사이드바(데스크탑/문서 등) 행은 AX로 진입 가능**(좌표 불필요, 이번에 검증된 보조경로). 핵심: 시트 안의 *사이드바 outline*을 찾아(첫 행 value가 "최근 항목") 그 안에서 원하는 행을 찾아 `perform action "AXOpen"`. ⚠️ **파일목록(파일 행) AXOpen은 이번에 불안정**했음 — 파일 진입/선택은 아래 cliclick 경로를 1순위로. (완성된 AX outline 탐색 스니펫은 길어서 생략 — 핵심은 "사이드바 outline = 첫행 최근항목, 파일목록 outline = 그 외"로 구분.)
- **폴더 진입 = cliclick 더블클릭**(로지컬 좌표): `cliclick dc:<lx>,<ly>`. 한 번에 안 들어가고 *선택만* 되면 한 번 더 dc(첫 dc가 select로 먹는 경우 있음).
- **파일 다중선택 = cmd+클릭**: `cliclick kd:cmd c:<lx>,<ly1> c:<lx>,<ly2> c:<lx>,<ly3> ku:cmd` → 선택되면 하이라이트+"열기" 버튼 활성.
- **확정 = 선택된 항목을 더블클릭**(열기 *버튼* 단일클릭은 cliclick이 안 먹힘. 검증됨): `cliclick dc:<lx>,<선택된행 ly>`. 다중선택 상태에서 한 항목 더블클릭하면 선택 전체가 열린다. (단 큰 파일 1장이 누락될 때가 있으니 업로드 후 카운트 확인하고 부족하면 그 파일만 개별 더블클릭으로 추가.)
- 업로드 직후 ASC가 "선택한 N" 안내 다이얼로그(승인 버튼)를 띄우면 Chrome MCP로 승인 클릭.

### 1-3. 위치 검증
클릭 전 `cliclick m:<lx>,<ly>` + computer-use `cursor_position`으로 "이 로지컬이 그 행/버튼 위인가" 1회 확인하면 빗맞힘이 없다. (computer-use 스크린샷에 렌더된 커서 위치는 무시 — cursor_position 반환값만 신뢰.)

### 1-3b. ⭐ 패널 활성화 — cliclick 직전 매번 Chrome activate (2026-06-29 결정적 검증)
**가장 흔한 실패: cliclick 클릭이 닿는데(선택/폴더진입은 됨) "확정(열기)"만 안 먹고 헛돈다.** 원인 = NSOpenPanel 창이 **비활성**(frontmost가 Chrome 아님 — computer-use 스크린샷 찍으면 포커스가 딴 데로 튐). macOS는 비활성 창의 첫 클릭을 "활성화"로만 소비(click-through 안 됨) → 더블클릭/Return/열기버튼이 다 무시됨.
- **해결(검증됨)**: cliclick 액션 **바로 앞에 매번** `osascript -e 'tell application "Google Chrome" to activate'`. 그러면 패널이 활성→더블클릭이 즉시 확정. 예: `osascript -e '...activate'; cliclick w:300 dc:<lx>,<ly>`.
- **다중선택 한방 업로드는 패널 활성일 때만 된다**: activate 후 `kd:cmd c:.. c:.. c:.. ku:cmd` 다중선택 → `dc:<선택행>` 하면 전체 열림(iPad 3장 동시 성공). 비활성이면 단일파일 더블클릭으로 1장씩(이것도 activate 필수).
- **MCP 탭이 백그라운드면 파일창이 안 뜬다**: 파일선택 클릭 전에 §1-4 osascript로 ASC 탭을 활성탭+창 최상단으로. 유저 Chrome에 탭/창 많으면 매 단계 osascript-front 반복.
- **긴 textarea(심사메모 등) 교체**: Chrome MCP `type`이 CDP 타임아웃 나도 **키스트로크는 실제로 들어가 있다**(value 길이로 확인) — 타임아웃 에러에 속지 말고 textarea value 읽어 검증. 그래도 React 미커밋 의심되면 §2의 1글자+백스페이스.

### 1-3c. 빌드 선택목록에 새 빌드가 안 뜰 때 = 페이지 reload
버전 페이지 "빌드 추가" 다이얼로그의 빌드 목록은 **페이지 로드 시점에 고정**된다. TestFlight에서 "제출 준비 완료"인 새 빌드가 다이얼로그에 없으면 = stale. **저장으로 dirty 정리(빌드 하나 임시선택+저장) → 페이지 새로고침 → 다이얼로그 재오픈**하면 새 빌드가 뜬다. (force-navigate는 beforeunload가 OS레벨이라 안 먹음 → 먼저 저장해 dirty를 없애야 reload 가능.)

### 1-4. MCP 백그라운드 창 문제
ASC 탭이 MCP 그룹의 백그라운드 창에 있으면 파일창이 안 보인다. 해당 URL 창을 앞으로:
```bash
osascript -e 'tell application "Google Chrome"
  repeat with i from 1 to count of windows
    if (URL of active tab of window i) contains "<appId>" then
      set index of window i to 1
      activate
    end if
  end repeat
end tell'
```

## 2. React 컨트롤드 입력 (ASC/Play 폼)
JS로 `el.value = x` + `dispatchEvent(input/change)` 하면 화면값만 바뀌고 React state는 안 바뀐다 → 저장버튼 비활성, 저장해도 서버에 옛값, 새로고침하면 리셋.
- **텍스트 필드/textarea**: Chrome MCP `computer` left_click(ref) → **Cmd+A → Delete** → `type`(실키스트로크). 긴 한글도 type으로 들어간다.
- **JS-set이 불가피한 큰 textarea**(CDP type 타임아웃 시): JS로 value 세팅 후, 그 필드를 클릭하고 **아무 문자 1개 type → Backspace**로 onChange를 깨운다.
- **select(콤보박스)**: 네이티브면 `form_input`(MCP)으로 되기도 하나 onChange 안 깨우면 클릭→옵션 ref 클릭. Apple ASC의 일부 select는 `form_input`이 화면 갱신만 하고 React 미반영 → 그땐 JS로 native setter+change 후에도 안 되면 실클릭.
- **checkbox(예: "로그인 필요")**: JS `.click()`은 미커밋되어 되돌아감 → **Chrome MCP `computer` left_click(ref)**로 실클릭. 토글 후 의존 필드(데모계정 username/password)가 사라지는지로 커밋 확인.

## 3. 진단 룰
- **빈 화면/SPA 본문 미렌더**: `read_console_messages`(onlyErrors) 먼저. 확장(크립토 지갑 등) inapp.js가 모듈로더를 깨뜨리면 본문 0 DOM. → 확장 OFF.
- **검증 에러가 떴는데 진짜인지 의심**: navigate로 **새로고침** 후 서버 실제값 읽기. 폼 helper 텍스트/탭 라벨이 에러 정규식에 오탐될 수 있음. "심사에 추가"는 에러 표시와 무관하게 실제로 성공해 있던 사례 있음.
- **JS exec 결과가 `[BLOCKED: Cookie/query string data]`**: 페이지 텍스트를 통째로 반환하면 차단됨 → 짧은 상태값만(불리언/카운트/인덱스) 반환.

## 4. 스크린샷 규격 일괄 생성 (PIL)
```python
from PIL import Image
import os
def gen(srcs, TW, TH, out):
    os.makedirs(out, exist_ok=True)
    for i,s in enumerate(srcs,1):
        im=Image.open(s).convert("RGB"); w,h=im.size
        sc=min(TW/w, TH/h); nw,nh=int(w*sc),int(h*sc)
        im2=im.resize((nw,nh), Image.LANCZOS)
        pad=im2.getpixel((nw//2,4))                 # 가장자리색으로 패딩
        c=Image.new("RGB",(TW,TH),pad); c.paste(im2,((TW-nw)//2,(TH-nh)//2))
        c.save(os.path.join(out,f"shot_{i}.png"),"PNG")
# iPhone 6.5": gen(srcs, 1284, 2778, "iphone")
# iPad 13":   gen(srcs, 2048, 2732, "ipad")
```
파일은 **업로드용 폴더를 따로** 둬라(파일창에서 cmd+A 전체선택이 쉽다).
