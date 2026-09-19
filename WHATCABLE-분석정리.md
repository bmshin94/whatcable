# WhatCable 프로젝트 분석 정리

> 작성일: 2026-09-19
> 분석 대상 저장소: **https://github.com/bmshin94/whatcable**
> 원본(upstream) 저장소: **https://github.com/darrylmorley/whatcable**
> 공식 웹사이트: **https://whatcable.uk**

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [쉽게 풀어쓴 설명](#2-쉽게-풀어쓴-설명)
3. [Q&A 7문답](#3-qa-7문답)
4. [수익화 아이디어 8선](#4-수익화-아이디어-8선)
5. [참고 링크](#5-참고-링크)

---

## 1. 프로젝트 개요

### 한 줄 요약

**"이 USB-C 케이블이 실제로 뭘 할 수 있는가"** 를 macOS 메뉴바에서 즉시 알려주는 앱.

### 기본 정보

| 항목 | 내용 |
| --- | --- |
| 이름 | WhatCable |
| 원작자 | Darryl Morley (영국) |
| 원본 저장소 | https://github.com/darrylmorley/whatcable |
| 이 저장소 | https://github.com/bmshin94/whatcable (원본 미러) |
| 웹사이트 | https://whatcable.uk |
| 종류 | macOS 메뉴바 앱 + CLI 바이너리 |
| 언어 | Swift / SwiftUI (약 49,798 LOC) |
| 최소 사양 | macOS 14 (Sonoma) 이상, **Apple Silicon 전용** |
| 라이선스 | MIT (단 `Sources/WhatCablePlugins/` 는 독점) |
| 빌드 | Swift Package Manager + XcodeGen(위젯) |

### 해결하는 문제

USB-C 케이블은 겉모습이 전부 동일하지만 내부 스펙은 천차만별이다.

```
겉모습: ━━━━━  (전부 동일)
속사정:
  A 케이블 → USB 2.0, 60W,  480Mbps   (사실상 충전 전용)
  B 케이블 → USB4,   240W, 40Gbps     (풀스펙)
  C 케이블 → 충전 전용, 데이터 라인 없음
```

그 결과 발생하는 문제: "충전이 느리다", "외장 SSD가 느리다", "모니터 해상도가 안 나온다".
WhatCable은 그 원인이 **맥북 / 케이블 / 충전기 / 기기 중 누구인지**를 평문으로 지목한다.

### 동작 원리 — 읽는 IOKit 서비스 4종

권한 요청, 사설 API, 헬퍼 데몬 없이 **읽기 전용**으로 동작한다.

| IOKit 서비스 | 얻는 정보 |
| --- | --- |
| `AppleHPMInterfaceType10/11/12/18`, `AppleTCControllerType10/11` | 포트별 연결 상태, 전송 방식, 플러그 방향, e-마커 유무 |
| `IOPortFeaturePowerSource` | 충전기가 광고하는 전체 PDO 목록 + 현재 선택된 PDO |
| `IOPortTransportComponentCCUSBPDSOP` / `SOPp` / `SOPpp` | PD Discover Identity VDO (기기 / 케이블 근단 / 원단 e-마커) |
| XHCI 컨트롤러 서브트리 | 연결된 USB 기기를 물리 포트에 매핑 |

> e-마커: 60W 초과 케이블 내부의 인증 칩. "나는 240W / 40Gbps 지원" 을 스스로 신고한다.
> WhatCable은 이 신고 내용을 읽어 사람 말로 번역한다.

### 저장소 구조 (실측)

```
whatcable/
├── Sources/                          Swift 49,798 LOC
│   ├── WhatCableCore/        23,774  ★ 두뇌: PD 비트 디코딩, 진단 로직, 포매팅
│   │   ├── Cable/                    케이블 분류 · 신뢰도(Trust) 검사
│   │   ├── PD/                       USB Power Delivery VDO/SOP 디코더
│   │   ├── Display/                  EDID 파싱 (해상도/주사율 진단)
│   │   ├── Thunderbolt/              TB 패브릭 · 터널 토폴로지
│   │   ├── Power/                    충전 진단 · SMC 계약 합성
│   │   ├── Devices/                  연결 기기 트리 · 허브 접기
│   │   ├── Output/                   Text / JSON / ANSI 포매터
│   │   └── Database/                 CableDB · VendorDB 조회
│   ├── WhatCableDarwinBackend/  9,784  IOKit 워처 (실제 하드웨어 읽기)
│   ├── WhatCable/               9,644  메뉴바 UI (SwiftUI 팝오버 · 설정)
│   ├── WhatCableNotifications/  4,288  연결/해제 알림
│   ├── WhatCableWidget/         1,341  WidgetKit 데스크탑 위젯
│   ├── WhatCableCLI/              475  CLI 바이너리
│   ├── WhatCableAppKit/           488  ★ PluginRegistry (확장 포인트)
│   └── WhatCablePlugins/            4  ★ Pro 기능 — 공개판은 빈 껍데기
├── Tests/                      223개 테스트 파일 (골든 픽스처 + 코퍼스 스윕)
├── probes/test-kit/            C 하드웨어 탐침 10종
├── data/known-cables.md        알려진 케이블 목록 (41KB)
├── docs/whatcable.db           케이블/벤더 SQLite DB (745KB) ★ 핵심 자산
├── src/ → docs/                Eleventy 정적 사이트 (GitHub Pages)
└── scripts/                    빌드 · 코드서명 · 공증 · 릴리즈 자동화
```

### 핵심 발견 — 오픈코어 전략

`Sources/WhatCablePlugins/Bootstrap.swift` 는 단 4줄이며 내용이 비어 있다.

```swift
import WhatCableAppKit

public func bootstrapPlugins(registry: PluginRegistry) {
}
```

즉 **유료(Pro) 기능의 소스는 공개하지 않았다.** 확장 포인트(`PluginRegistry`)만 오픈소스로
공개하고, 실제 Pro 구현은 비공개 저장소에서 주입한다.
→ 하나의 코드베이스로 무료판과 유료판을 동시에 빌드하는 전형적인 **오픈코어** 구조.

### 무료 vs Pro 기능 비교

| 무료 (MIT) | Pro (£9.99 · 일회성 · 2대) |
| --- | --- |
| 포트별 케이블 e-마커 정보 | 케이블 이력 관리 (이름 부여 후 성능 추적) |
| 충전 병목 진단 | 실시간 전력 계측 + Power Monitor 그래프 |
| 데이터 속도 진단 | Negotiation Diagnostics (협상 상세 비교) |
| 썬더볼트 패브릭/토폴로지 | Display Diagnostics (해상도/DSC/어댑터 추적) |
| USB-IF 인증 확인 | 포트 헬스 카운터 · 케이블 저항 추정 |
| 케이블 신뢰도 경고 카드 | 터미널 TUI 대시보드 (`--dashboard`) |
| 데스크탑 위젯 (S/M/L) | 핀 다이어그램 · 액체 감지 상태 |
| CLI (`--json`, `--watch`) | 라이브 계측 미지원 맥에서도 동작 |
| 19개 언어 (한국어 포함) | |

### 이 프로젝트가 주는 가치

**실용적 측면**
- 서랍 속 케이블의 실제 등급 파악 및 라벨링
- 충전 저속 원인이 케이블/충전기/맥북 중 무엇인지 즉시 판별
- 저가 케이블의 스펙 과장 여부 확인 (신뢰도 카드)
- 모니터 해상도 미달 원인 추적

**개발자 측면**
- 대규모 Swift/SwiftUI 프로젝트의 모듈 분리 레퍼런스
- 223개 테스트 + 골든 픽스처 기반 회귀 검증 설계 학습
- 오픈코어 + 플러그인 레지스트리 수익화 구조를 소스로 확인 가능
- `whatcable --json` 은 로컬 AI 에이전트의 도구로 즉시 활용 가능

---

## 2. 쉽게 풀어쓴 설명

### 비유 1 — 케이블 전문 병원

케이블은 말 못 하는 환자다. 다만 60W 초과 케이블에는 **주민등록증(e-마커 칩)** 이 들어있다.

```
케이블 주민등록증 (e-마커)
┌──────────────────────┐
│ 제조사   : Apple      │
│ 최대전류 : 5A         │
│ 최대전력 : 100W       │
│ 속도     : 40 Gbps    │
│ USB-IF 인증 : 있음    │
└──────────────────────┘
```

WhatCable은 이 증명서를 읽어 한국어로 통역한다.

### 비유 2 — 진단은 "3자 대질"

단순 조회가 아니라 **세 당사자의 주장을 비교**해서 병목을 지목한다.

```
충전기 : "나는 96W까지 공급 가능"
케이블 : "나는 60W까지만 가능"
맥북   : "나는 96W를 받고 싶음"
        ↓
판정   : 범인은 케이블 → "케이블이 충전 속도를 제한하고 있음"
```

다른 판정 예시:
- 셋 다 일치 → "96W로 잘 충전 중"
- 배터리 95% → "맥북이 30W만 요청 중 (충전기는 96W 가능)"
- 배터리 100% → "배터리 가득참, 충전하지 않는 중"

이것이 단순 리더기와 결정적으로 다른 점이다. **숫자가 아니라 책임 소재를 알려준다.**

### 비유 3 — 케이블 안의 여러 차선

```
   USB-C 케이블 단면
┌─────────────────────────────────────┐
│ 전원 도로      → 충전 (5V~20V)        │
│ USB2 도로      → 느린 데이터          │
│ USB3 도로      → 빠른 데이터          │
│ 썬더볼트 도로  → 초고속 (40Gbps)      │
│ DisplayPort    → 영상                 │
└─────────────────────────────────────┘
```

저가 케이블은 전원 도로만 깔려 있다. "충전은 되는데 데이터가 안 가는" 이유.
WhatCable은 **지금 어떤 도로가 살아있는지(Active transports)** 를 보여준다.

### 비유 4 — IOKit은 맥북 몸속 CCTV

```
macOS 내부 IOKit 레지스트리  (복잡한 raw 데이터로 이미 존재)
        ↓  WhatCable이 읽고 해석
"케이블이 충전 속도를 제한하고 있습니다"  ← 평문 번역
```

해킹이 아니다. macOS가 이미 알고 있지만 보여주지 않는 정보를 꺼내 정리할 뿐이다.
읽기 전용이라 안전하고, 자동 네트워크 전송도 없다.

### 비유 5 — 모듈 구조를 회사 조직으로

```
WhatCableDarwinBackend (현장 조사팀)  9,784 LOC
   → IOKit에서 raw 데이터 수집
        ↓
WhatCableCore (분석/두뇌팀)          23,774 LOC  ★ 최대 모듈
   → 비트 해석 + 병목 판정
        ↓
WhatCable (UI팀) / WhatCableCLI (터미널팀) / WhatCableWidget (위젯팀)
WhatCablePlugins (유료팀, 비공개)
```

**핵심 설계:** 두뇌(Core)가 UI와 완전히 분리되어 있어, 동일 엔진으로
GUI · CLI · 위젯을 모두 만든다. 한 번 만들어 세 번 재사용하는 구조.

### 왜 테스트가 223개인가

맥북 기종과 케이블 종류는 수천 가지인데 개발자가 전부 구매할 수 없다. 그래서:

```
1. 사용자가 "진단 데이터 기여" 버튼 클릭 (완전 자발적)
        ↓
2. 해당 맥북의 익명화된 IOKit 덤프 수집
        ↓
3. 테스트 코퍼스(corpus)로 저장
        ↓
4. 코드 수정 시마다 실제 하드웨어 케이스 전체로 회귀 검증
```

**사용자가 늘수록 앱이 정확해지는 구조** — 이 프로젝트의 진짜 경쟁력.

---

## 3. Q&A 7문답

### Q1. 설치 및 사용법은?

**사전 조건:** macOS 14 (Sonoma) 이상 + Apple Silicon (M1~M5). **인텔 맥 불가.**

> 인텔 맥이 불가능한 이유는 추측이 아니라 측정 결과다. 커뮤니티 진단 코퍼스에 모인
> 모든 인텔 맥이 USB-C 포트 컨트롤러 서비스를 빈 값으로 반환한다. 즉 애플이 해당
> 계층을 공개 IOKit 접근자로 노출하지 않아 소프트웨어적으로 우회가 불가능하다.
> (썬더볼트 패브릭 데이터는 인텔에서도 읽히지만, 포트 컨트롤러 계층이 비어 있다.)

**설치 방법**

```bash
# 1) Homebrew — 권장 (앱 + CLI, PATH 자동 연결)
brew install --cask darrylmorley/whatcable/whatcable

# 2) CLI만 설치
brew install darrylmorley/whatcable/whatcable-cli

# 탭 신뢰 경고가 뜰 경우
brew trust darrylmorley/whatcable
```

```
# 3) 수동 설치
https://github.com/darrylmorley/whatcable/releases/latest
→ WhatCable.zip 다운로드 → 압축 해제 → /Applications 로 드래그
```

Developer ID 서명 + Apple 공증이 되어 있어 Gatekeeper 경고가 뜨지 않는다.
수동 설치 시 CLI는 앱 번들 안에 있으므로 심볼릭 링크가 필요하다.

```bash
ln -s /Applications/WhatCable.app/Contents/Helpers/whatcable /usr/local/bin/whatcable
```

**GUI 사용법**

| 조작 | 기능 |
| --- | --- |
| 메뉴바 아이콘 클릭 | 포트별 상태 팝오버 |
| ⌥(Option) + 클릭 | 엔지니어 모드 (raw IOKit 속성 노출) |
| 우클릭 | 새로고침 / 창 고정 / 설정 / Pro 화면 |
| 톱니바퀴 아이콘 | 설정 (언어 · 표시 · 알림) |

설정 항목: 빈 포트 숨기기, 로그인 시 실행, Dock 앱 모드, 메뉴바 실시간 와트 표시,
폰트 크기/투명도, 19개 언어 전환, 연결/해제 알림, USB 깊은 탐색 비활성화(KVM/허브 호환).

**CLI 사용법**

```bash
whatcable                # 사람이 읽는 포트 요약
whatcable --json         # 구조화 JSON (jq 파이프용)
whatcable --watch        # 연결/해제 실시간 스트림
whatcable --raw          # IOKit 원본 속성 포함
whatcable --report       # 케이블 정보로 GitHub 이슈 사전 작성
whatcable --test-kit     # 진단 프로브 실행 + 익명 제보
whatcable --no-usb-probe # 깊은 USB 탐색 생략
whatcable --desktop      # GUI를 Dock 모드로 실행
whatcable --popover      # GUI를 메뉴바 모드로 실행
whatcable --version / --help

# Pro 전용
whatcable --monitor              # 실시간 전력 텔레메트리
whatcable --monitor-json         # NDJSON 스트림 (스크립팅)
whatcable --dashboard            # 풀스크린 TUI (Tab 전환, q 종료)
whatcable --activate XXXX-XXXX-XXXX-XXXX
whatcable --licence / --deactivate / --pro
```

**소스 빌드** (macOS에서만 가능, Swift 5.9+ / Xcode 15+)

```bash
git clone https://github.com/bmshin94/whatcable
cd whatcable
swift build            # 전체 컴파일
swift run WhatCable    # 앱 실행 (개발 모드, 위젯/번들 구조 없음)
swift run whatcable-cli
swift test             # 223개 테스트
./scripts/smoke-test.sh  # 배포용 .app 빌드 + 서명 + 공증
```

---

### Q2. 플러그인? 스킬? MCP?

**셋 다 아니다.** 독립 실행되는 네이티브 macOS 앱 + CLI 바이너리이며, Claude와 무관하다.

| 구분 | 여부 | 근거 |
| --- | :---: | --- |
| Claude 플러그인 | X | `.claude-plugin/` 없음 |
| Claude 스킬 | X | `SKILL.md` 없음 |
| MCP 서버 | X | MCP 프로토콜 구현 없음 |
| 네이티브 macOS 앱 | O | Swift/SwiftUI 앱 번들 |

**혼동 주의:** `WhatCablePlugins`, `PluginRegistry` 는 **앱 내부 전용 자체 플러그인 시스템**이다.
유료 기능을 탈착하기 위한 구조이며, 외부 플러그인 규격이 아니다.

```swift
@MainActor
public final class PluginRegistry {
    public static let shared = PluginRegistry()

    public private(set) var launchHooks: [() async -> Void] = []
    public private(set) var menuItems: [MenuPlacement: [PluginMenuItem]] = [:]
    public private(set) var headerButtonBuilders: [() -> AnyView] = []
    public private(set) var portCardTrailingBuilders: [(PortCardContext) -> AnyView?] = []
    public private(set) var cliCommands: [CLICommand] = []
    public private(set) var widgetDataContributors: [any WidgetDataContributor] = []
    public private(set) var cliOutputFooterContributors: [() -> String?] = []
    // ...
}
```

무료 배포판은 `bootstrapPlugins()` 가 비어 있어 아무것도 등록되지 않고, 결과적으로
Pro 메뉴/버튼/CLI 명령이 전부 사라진다. **단일 코드베이스 · 이중 배포**의 정석.

**다만 MCP로 감싸는 것은 매우 쉽다.** `--json` 출력이 깔끔하기 때문이다.

```js
// mcp-whatcable (개념)
import { execSync } from "child_process";
server.tool("get_cable_status", {}, async () => {
  const data = JSON.parse(execSync("whatcable --json").toString());
  return { content: [{ type: "text", text: JSON.stringify(data) }] };
});
```

---

### Q3. API 토큰이 필요한가?

**사용에는 전혀 불필요하다.** 앱 실행 · CLI · 진단 전부 로컬에서 완결된다.
인터넷을 끊어도 `whatcable --json` 은 정상 동작한다.

토큰이 등장하는 곳은 3군데뿐이며, 모두 일반 사용자와 무관하다.

| 상황 | 필요한 것 | 대상 |
| --- | --- | --- |
| 앱 직접 빌드/배포 | `DEVELOPER_ID` (애플 인증서), `NOTARY_PROFILE` (공증 자격) | 배포자만 |
| 릴리즈 스크립트 | `gh` CLI 인증 | 배포자만 |
| Pro 활성화 | `XXXX-XXXX-XXXX-XXXX` 라이선스 키 (API 토큰 아님) | 구매자 |

개발 목적의 `swift build` / `swift test` 에는 아무 자격증명도 필요 없다.

**네트워크를 사용하는 3가지 (모두 투명)**

| 동작 | 시점 | 인증 | 전송 정보 |
| --- | --- | :---: | --- |
| 업데이트 확인 | 6시간마다 자동 | 없음 (GitHub 공개 API) | 없음 |
| 케이블 제보 | 버튼 클릭 시에만 | 없음 (브라우저로 이슈 폼 오픈) | 케이블 VID/PID/VDO |
| 진단 데이터 기여 | 버튼 클릭 시에만 | 없음 | 익명화 IOKit 정보 |

뒤 두 가지는 사용자가 직접 클릭해야만 동작하며 자동 전송은 없다.

---

### Q4. 왜 GitHub에서 유명한가?

저장소 내부 증거로 확인한 8가지 이유.

1. **보편적 불편의 해결** — README 마지막 문장이 핵심을 요약한다.
   *"Inspired by every time someone has asked 'is this cable any good?'"*
2. **숫자가 아닌 평문 답변** — 경쟁 도구는 `VDO: 0x8C4841` 를 던지지만,
   WhatCable은 "케이블이 충전 속도를 제한하고 있음" 이라고 말한다.
   엔지니어 도구를 일반 사용자 도구로 번역한 것이 결정적 차별점.
3. **무료 + 오픈소스 + 완성도 높은 UI** — 셋을 동시에 갖춘 사례가 드물다.
4. **네트워크 효과** — 사용자 제보 → `whatcable.db`(745KB) / `known-cables.md`(41KB) 성장
   → 정확도 상승 → 사용자 증가. 30명 이상이 하드웨어 덤프를 기여했고,
   이 데이터는 후발주자가 복제할 수 없는 해자다.
5. **19개 언어 현지화** — `de, en, es, fr, hi, hy, it, ja, ko, lv, nb, nl, pl, pt-BR,
   ru, tr, uk, zh-Hans, zh-Hant`. 한국어는 @dohun0310 기여.
6. **Product Hunt 상위 노출 + 배포 편의성** — README에 PH top-post 배지.
   `brew install` 한 줄 설치, Apple 공증 완료로 진입 장벽이 사실상 0.
7. **품질에 대한 집착** — 223개 테스트, 골든 픽스처, `THIRD_PARTY_NOTICES.md` 에
   포팅 코드의 출처를 커밋 해시 단위로 명시.
8. **정직한 한계 고지 → 신뢰 확보** — "소프트웨어는 피복 안을 검증할 수 없다.
   칩이 240W라고 거짓말하면 그건 칩의 거짓말이다." 신뢰도 카드 문구도
   "가짜다"가 아니라 "이 값은 이례적으로 보인다" 로 신중하게 표현한다.

**생태계 확장:** Linux 포팅 [usbeehive](https://github.com/abrauchli/usbeehive) (Rust, 독립 구현),
형제 앱 [WhatBattery](https://www.whatbattery.app) · [WhatPort](https://www.whatport.app), GitHub Sponsors 후원자 존재.

> 인디 개발 황금공식: **작고 명확한 문제 + 평문 답변 + 오픈소스 신뢰 + 커뮤니티 데이터 수집**

---

### Q5. 로컬 에이전트 구축에 도움이 되는가?

**두 가지 의미에서 크게 도움이 된다.**

#### A. 에이전트 "도구(Tool)"로서 — 즉시 활용 가능

로컬 에이전트의 최대 약점은 물리 세계를 관측할 수단이 없다는 것인데, WhatCable이 그 눈이 된다.

| 도구 적합성 조건 | 충족 |
| --- | :---: |
| 구조화 출력 (JSON) | O `--json` |
| 이벤트 스트림 | O `--watch`, `--monitor-json` (NDJSON) |
| 부작용 없음 (읽기 전용) | O |
| 권한/인증 불필요 | O |
| 빠른 응답 (로컬) | O |
| 결정론적 출력 | O (골든 픽스처로 보장) |

> **읽기 전용 + 무인증 + JSON** 조합은 에이전트 도구로서 최상급이다.
> 에이전트가 실수로 시스템을 손상시킬 여지가 없다.

구현 가능한 것:
- **하드웨어 집사 에이전트** — `--watch` 로 연결 감지 → LLM이 원인 분석 →
  "지금 꽂은 케이블은 60W입니다. 서랍의 240W 케이블로 교체하세요" (개인 케이블 DB와 결합)
- **MCP 서버 래핑** — `get_cable_status` / `diagnose_charging` / `list_connected_devices`
  툴 3개만 노출해도 Claude가 직접 하드웨어를 진단할 수 있다.

#### B. 설계 교본으로서 — 장기적으로 더 값질 수 있음

1. **수집층 / 판단층 / 표현층 완전 분리**
   `DarwinBackend(수집) → Core(판단) → Output(표현)`.
   에이전트의 `관측 → 추론 → 행동` 구조와 정확히 대응한다.
2. **하나의 엔진, 여러 표면** — Core 하나로 GUI · CLI · 위젯을 만든다.
   에이전트도 로직은 한 곳, 인터페이스는 여러 개가 정석.
3. **PluginRegistry = 툴 레지스트리 패턴** — `register(cliCommand:)`, `register(launchHook:)`
   구조가 에이전트 도구 등록 방식과 동일하다.
4. **골든 픽스처 = eval 데이터셋** — `Tests/.../Fixtures/golden/*.json|.txt` 는
   입력→기대출력을 고정한 회귀 검증이며, LLM eval 설계와 개념이 같다.
5. **"모르면 모른다" 설계** — 애매한 값은 단정하지 않고 "이례적임" 으로만 표기.
   에이전트 환각 방지 설계의 좋은 참고 사례.

**한계:** macOS + Apple Silicon 전용(서버/리눅스 에이전트에는 부적합, 리눅스는 usbeehive),
Pro 기능은 라이선스 필요, 읽기 전용이라 "쓰기" 액션 불가.

---

### Q6. 수익화 아이디어가 있는가?

→ [4. 수익화 아이디어 8선](#4-수익화-아이디어-8선) 참조.

핵심 인사이트 하나만 먼저:

> **이 프로젝트의 진짜 수익 자산은 앱이 아니라 `whatcable.db` (케이블 지문 DB) 다.**
> 앱은 복제 가능하지만, 수천 명이 제보한 실제 케이블 지문 데이터는 복제할 수 없다.
> 수익화를 설계한다면 **데이터를 모으는 구조**를 가장 먼저 만들어야 한다.

---

### Q7. React나 PHP로 만들 수 있는가?

**핵심 진단 엔진은 불가능하고, 주변부는 전부 가능하다.**

#### 불가능한 부분 — 센서층

| 기술 | e-마커 읽기 | 사유 |
| --- | :---: | --- |
| React (브라우저) | X | 샌드박스. IOKit 접근 불가 |
| WebUSB API | X | USB 기기는 열람 가능하나 케이블 e-마커(PD 계층)는 미노출 |
| WebHID | X | 동일 |
| PHP | X | 서버 언어. 클라이언트 하드웨어 접근 불가 |
| Node.js + node-usb | △ | USB 기기는 가능하나 USB-PD SOP'/SOP'' 는 불가 |
| Swift / Obj-C / C | O | IOKit 직접 접근 |
| Rust | O | io-kit-sys 바인딩 (usbeehive 방식) |

케이블 e-마커는 `IOPortTransportComponentCCUSBPDSOP` 계열 IOKit 서비스에 존재하며,
**네이티브 코드만 접근 가능**하다. 웹 기술로는 우회할 수 없다.

#### 가능한 부분 — 전략: 네이티브는 센서, React/PHP는 화면과 두뇌

```
┌──────────────────────────────────────────────┐
│ 센서층 (네이티브 필수)                         │
│ Swift/Rust 데몬 또는 기존 whatcable CLI 그대로 │
└────────────────┬─────────────────────────────┘
                 │ JSON / NDJSON / WebSocket
                 ↓
┌──────────────────────────────────────────────┐
│ React 앱 (Tauri / Electron)                   │
│ 대시보드, 실시간 그래프, 케이블 관리 UI         │
└────────────────┬─────────────────────────────┘
                 │ HTTPS
                 ↓
┌──────────────────────────────────────────────┐
│ PHP 백엔드 (Laravel)                          │
│ 케이블 DB API, 라이선스 서버, 팀 대시보드       │
└──────────────────────────────────────────────┘
```

**React 영역:** Tauri/Electron 데스크탑 앱(`--json` exec), 실시간 전력 그래프
(`--monitor-json` → Recharts), 케이블 검색 웹사이트(Next.js), B2B 관리 대시보드.

**PHP(Laravel) 영역:** 케이블 DB REST API, 라이선스 발급/검증 서버,
진단 데이터 수집 엔드포인트, B2B 자산관리 백엔드, 제휴 커머스 연동.

#### 추천 조합

| 안 | 구성 | 기간 |
| --- | --- | --- |
| 1안 (현실적) | 기존 whatcable CLI + Tauri/React 프론트 | 1~2주 |
| 2안 (수익형) | 최소 네이티브 센서 + React + Laravel SaaS | 1~2개월 |
| 3안 (웹 전용) | `whatcable.db` 파싱 → Next.js 케이블 검색 사이트 | 2~3일 |

#### 라이선스 확인 결과 (실제 `LICENSE` 파일 기준)

```
[가능] MIT 부분 (Core / CLI / Backend / AppKit 등)
   → 사용, 복사, 수정, 병합, 배포, 서브라이선스, 판매 모두 허용
   → 조건: 저작권 표시 + MIT 라이선스 전문 포함

[불가] Sources/WhatCablePlugins/ (Pro)
   → proprietary, all rights reserved (공개 저장소에는 빈 껍데기만 존재)

[주의] THIRD_PARTY_NOTICES.md
   → EDID 관련 3개 파일은 v4l-utils(edid-decode)에서 포팅됨. 해당 라이선스 준수 필요
```

**결론: MIT이므로 상업적 이용 가능.** 단 (1) 저작권 고지 유지, (2) 출처 명시,
(3) "WhatCable" 상표를 그대로 사용하지 말 것.

---

## 4. 수익화 아이디어 8선

### 대전제

```
진짜 해자(moat)는 "앱" 이 아니라 "데이터" 다.
  앱          → 약 3개월이면 복제 가능
  케이블 지문 DB → 수천 명의 제보 없이는 불가능
```

**시장 타이밍**
- USB-C가 EU 규제 등으로 사실상 전 세계 법적 표준화
- USB4 v2 / 240W / TB5 등장으로 케이블 혼란은 오히려 심화
- 저가 케이블 범람 → "실제 스펙 검증" 수요 증가

---

### 1. B2B 케이블 자산관리 SaaS  (최우선 추천)

| 수익성 | 난이도 | 기간 | 스택 |
| :---: | :---: | --- | --- |
| ★★★★★ | ★★★★ | 2~3개월 | Swift 에이전트 + React + Laravel |

**문제:** 맥북 100대 규모 조직에서 "왜 저 팀만 충전이 느린가"를 IT팀이 일일이 확인해야 한다.

**해결:** 각 맥에 경량 에이전트 배포 → 중앙 대시보드에서 전사 케이블 상태 집계.

```
관리자 대시보드
├─ 위험: 3대에서 과전류 감지 → 케이블 교체 필요
├─ 비효율: 27대가 60W 케이블로 96W 맥북 충전 중
├─ 노후: 12개월 이상 사용 케이블 45개
└─ 구매 제안: 240W 케이블 27개 (예상 비용 자동 산출)
```

**수익 구조:** 맥 1대당 월 2,000~3,000원.
100대 고객사 = 월 20~30만원. 50개사 확보 시 월 1,000~1,500만원.

**진입 전략:** Jamf / Kandji 같은 MDM의 **연동 모듈**로 포지셔닝하면 영업이 쉬워진다.
IT 자산관리는 이미 예산이 배정된 시장이다.

---

### 2. 케이블 지문 DB API 구독

| 수익성 | 난이도 | 기간 | 스택 |
| :---: | :---: | --- | --- |
| ★★★★ | ★★ | 3~4주 | Laravel 또는 Node + Postgres |

VID / PID / VDO(케이블 지문)를 브랜드·모델·실제 스펙으로 변환하는 API.

```http
GET /api/v1/cable?vid=0x05AC&pid=0x1234
→ {
    "brand": "Apple",
    "model": "Thunderbolt 4 Pro Cable (1m)",
    "verified": true,
    "max_power": "240W",
    "max_speed": "40Gbps",
    "trust_score": 98,
    "reports": 1247
  }
```

**고객:** 이커머스(상품 스펙 자동 검증), 리뷰 사이트/유튜버, 케이블 제조사, 앱 개발자.

**가격 티어:** Free 100 req/월 · Hobby 9,900원/월(10,000 req) · Business 99,000원/월(무제한).

가장 자산형에 가까운 모델. DB를 한 번 구축하면 지속적으로 수익이 발생한다.

---

### 3. 진단 → 커머스 제휴  (가장 빠른 시작점)

| 수익성 | 난이도 | 기간 | 스택 |
| :---: | :---: | --- | --- |
| ★★★★ | ★ | 1~2주 | React 단일 페이지 |

```
진단: "케이블이 충전을 60W로 제한 중"
  ↓
제안: "이 맥북은 96W까지 수용 가능합니다"
  ↓
추천: [100W 인증 케이블 보기] → 제휴 링크
```

**전환율이 높은 이유:** 일반 광고는 "사세요" 지만, 이것은 "당신의 케이블에 실제 문제가
확인되었고 이 제품이 해결합니다" 이다. 개인화된 진단이 구매 결정에 선행한다.

**수익 추정:** 쿠팡 파트너스 3% / 아마존 4%.
케이블 평균 2만원 × 월 500건 × 3% ≈ 월 30만원 (트래픽에 선형 비례).

**필수 준수:** 제휴 링크임을 명시할 것. 진단 결과를 판매 목적으로 왜곡하면
프로젝트의 유일한 자산인 신뢰가 무너진다.

---

### 4. MCP 서버 / AI 에이전트 도구 판매

| 수익성 | 난이도 | 기간 | 스택 |
| :---: | :---: | --- | --- |
| ★★★ | ★★ | 2~3주 | Node / TypeScript (MCP SDK) |

"맥북 하드웨어 진단 MCP" — AI 에이전트가 직접 하드웨어를 진단하게 하는 도구.

```
사용자: "내 맥북 왜 이렇게 느려?"
에이전트: [get_cable_status] [diagnose_charging] [list_devices]
에이전트: "포트 2의 케이블이 USB 2.0이라 SSD가 480Mbps로 동작 중입니다.
          포트 1의 썬더볼트 케이블로 교체하면 40Gbps가 나옵니다."
```

**확장:** 배터리 · 디스플레이 · 네트워크까지 묶어 "맥북 종합 진단 MCP 스위트" 로 확대.

**가격:** 개인 $5/월, 팀 $20/월, 또는 일회성 $29.

**타이밍:** MCP 생태계 초기이며 **하드웨어 접근 도구가 거의 없다.** 선점 가능한 영역.

---

### 5. 수리점 / AS센터용 진단 키오스크

| 수익성 | 난이도 | 기간 | 스택 |
| :---: | :---: | --- | --- |
| ★★★★ | ★★★ | 1~2개월 | 네이티브 + React 키오스크 UI |

```
매장 체험존 "케이블 무료 진단"
  → 고객이 본인 케이블 연결
  → 진단서 출력 또는 메신저 전송
  → 매장에서 적합한 케이블 판매 (업셀)
```

**타겟:** 사설 수리점, 통신사 매장, 기업 헬프데스크.
**수익:** 매장당 월 5만원 라이선스, 또는 하드웨어+SW 패키지 50만원/대.
매장 입장에서는 고객 유입과 판매 근거를 동시에 확보하므로 ROI가 명확하다.

---

### 6. Windows / Android 포팅  (빈 시장)

| 수익성 | 난이도 | 기간 | 스택 |
| :---: | :---: | --- | --- |
| ★★★★ | ★★★★★ | 3~6개월 | C++/Rust + WinRT, Kotlin/NDK |

```
macOS   → WhatCable (존재)
Linux   → usbeehive (존재)
Windows → 없음   ← 기회
Android → 없음   ← 기회
```

Windows 사용자 규모는 macOS의 약 10배다.
기술적으로는 `UCSI`(USB Type-C Connector System Interface) 드라이버를 경유해 PD 정보에
접근할 수 있으나, 제조사별 구현 편차가 커서 난이도가 매우 높다.
**아무도 하지 않은 이유이자, 동시에 기회인 지점.**

**현실적 접근:** 전체 지원을 노리지 말고 인기 모델(삼성 갤럭시북, LG 그램 등)
몇 종만 먼저 지원한 뒤 확장.

**가격:** 일회성 15,000원. 사용자 규모를 고려하면 macOS판보다 매출이 클 수 있다.

---

### 7. 콘텐츠 + 검증 배지 비즈니스

| 수익성 | 난이도 | 기간 | 스택 |
| :---: | :---: | --- | --- |
| ★★★ | ★★ | 지속형 | Next.js / React |

```
1) 콘텐츠 (유입)
   "3천원 케이블 vs 5만원 케이블 실측", "저가 240W 케이블 10종 진위 검증"
        ↓
2) 검증 DB 사이트 (신뢰)
   케이블 스펙 백과사전 → SEO 트래픽 확보
        ↓
3) 수익화
   ├─ 제휴 커머스 (3번과 결합)
   ├─ "검증됨" 배지 유료 발급 (판매자 대상, 건당 10만원)
   └─ 제조사 컨설팅 / 테스트 의뢰
```

**한국 시장 특화:** 해외 직구 저가 케이블 범람과 스펙 과장이 심각해
"진위 감별" 콘텐츠의 수요가 크다.

---

### 8. 개인용 Pro 모델 + 한국 현지화

| 수익성 | 난이도 | 기간 |
| :---: | :---: | --- |
| ★★ | ★★ | 1개월 |

원작과 동일한 오픈코어 + 유료 플러그인 방식. 다만 원작과 직접 경쟁하므로 차별화 필요.

```
한국 시장 특화 요소
├─ 케이블을 "책상 서랍 2번칸 검정 케이블" 식으로 물리적 위치 기반 관리
├─ 국내 쇼핑몰 가격 비교 내장
├─ 카카오톡 알림 연동
└─ 국내 유통 케이블 DB 집중 구축
```

**가격:** 12,000원 일회성 (2대).
**주의:** MIT라 합법이지만 이름 변경 · 출처 명시는 필수이며, 원작자에게 사전 고지하는 것이 바람직하다.

---

### 추천 실행 로드맵

```
STEP 1 (1~2주)  빠른 시장 검증
  3번(제휴 커머스) + 7번(콘텐츠)
  → 기존 whatcable CLI 활용, React 프론트만 제작. 초기 비용 0원

STEP 2 (1~2개월)  자산 구축
  2번(케이블 DB API, Laravel)
  → STEP 1에서 수집한 데이터로 DB 구축 → 해자 형성

STEP 3 (3개월~)  본격 수익화
  1번(B2B SaaS, React + Laravel)
  → STEP 2의 DB가 그대로 핵심 경쟁력이 됨
  병행: 4번(MCP 서버) — 선점 효과
```

**리스크 낮은 순:** 3 → 7 → 4 → 2 → 5 → 1 → 8 → 6

가장 권장하는 시작점은 **3번(제휴 커머스)** 이다. 2주 내 출시해 시장 반응을 확인할 수 있고,
여기서 쌓이는 데이터가 나머지 모든 아이디어의 전제 조건이 된다.

---

## 5. 참고 링크

| 구분 | 주소 |
| --- | --- |
| **이 저장소** | https://github.com/bmshin94/whatcable |
| 원본 저장소 | https://github.com/darrylmorley/whatcable |
| 릴리즈 (다운로드) | https://github.com/darrylmorley/whatcable/releases/latest |
| 공식 웹사이트 | https://whatcable.uk |
| Pro 구매 페이지 | https://whatcable.uk/pro |
| Linux 포팅 (usbeehive) | https://github.com/abrauchli/usbeehive |
| Linux GNOME UI (usbee) | https://github.com/abrauchli/usbee |
| 형제 앱 — WhatBattery | https://www.whatbattery.app |
| 형제 앱 — WhatPort | https://www.whatport.app |

### 저장소 내 주요 문서

| 파일 | 내용 |
| --- | --- |
| `README.md` | 전체 기능 · 설치 · 원리 · 한계 설명 (29KB) |
| `LICENSE` | MIT + 독점 부분 이중 라이선스 고지 |
| `THIRD_PARTY_NOTICES.md` | 포팅 코드 출처 (v4l-utils edid-decode 등) |
| `TRANSLATIONS.md` | 번역 기여 가이드 · 용어 정책 |
| `CONTRIBUTORS.md` | 기여자 목록 |
| `.env.example` | 빌드/배포용 환경변수 템플릿 |
| `data/known-cables.md` | 알려진 케이블 목록 (41KB) |
| `docs/whatcable.db` | 케이블/벤더 SQLite DB (745KB) |
