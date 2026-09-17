# RKN Block Checker 프로젝트 분석 정리 (한국어)

> 이 문서는 프로젝트를 처음 접한 사람이 **"이게 뭐고, 나한테 왜 쓸모 있는지"**를
> 빠르게 파악할 수 있도록 정리한 분석 노트입니다.

## 관련 링크

| 구분 | 주소 |
|---|---|
| 원본 저장소 (upstream) | https://github.com/MayersScott/rkn-block-checker |
| 이 저장소 (fork) | https://github.com/bmshin94/rkn-block-checker |
| PyPI 패키지 | https://pypi.org/project/rkn-block-checker/ |
| 이슈 트래커 | https://github.com/MayersScott/rkn-block-checker/issues |

- 라이선스: MIT
- 버전: 0.6.0
- 요구사항: Python 3.10+
- 원본 저장소 지표(2026-09 기준): 스타 약 1.6k, 포크 63, 커밋 57

---

## 1. 한 줄 요약

**현재 사용 중인 인터넷 회선이 RKN/TSPU 검열 구간에 있는지, 그리고 "어떤 방식으로"
막혀 있는지를 네트워크 계층별로 진단하는 파이썬 CLI 도구.**

RKN은 러시아 연방 통신감독청(Роскомнадзор), TSPU는 ISP 망에 설치된 DPI 검열 장비를
가리킵니다.

## 2. 핵심 아이디어

브라우저는 실패하면 "연결할 수 없음" 한 줄만 보여줍니다. 이 도구는 **어느 계층에서
끊겼는지**를 특정합니다.

접속 과정을 택배 배달에 비유하면:

1. **주소 찾기 (DNS)** — 잘못된 주소를 알려줌 = DNS 오염
2. **찾아가기 (TCP)** — 길목이 막힘 = IP 차단 / RST 주입
3. **초인종 누르기 (TLS)** — 이름 대자마자 쫓겨남 = SNI 기반 DPI
4. **물건 받기 (HTTP)** — 물건 대신 "차단되었습니다" 안내문 = ISP 스텁 페이지

`rkn_checker/core.py`의 `check_url()`이 이 순서로 내려가며 **처음 깨지는 지점에서
멈추고 그것을 판정으로 확정**합니다.

| 단계 | 검사 방법 | 실패 시 판정 |
|------|-----------|--------------|
| DNS | 시스템 DNS vs Cloudflare DoH 결과 비교 | `DNS_BLOCK` |
| TCP | 443 포트 연결 시도 | `TCP_RESET` |
| TLS | ClientHello 후 핸드셰이크 관찰 | `TLS_BLOCK` |
| HTTP | 본문에서 차단 안내 문구 매칭 | `HTTP_STUB` |

### DNS 오염을 잡는 방법

- 시스템 DNS(= ISP 리졸버)에 질의 → 결과 A
- Cloudflare DoH(암호화 DNS)에 질의 → 결과 B
- **A와 B가 다르면 ISP 리졸버가 거짓말하고 있다는 신호**

`--proxy` 사용 시에도 **시스템 DNS 조회만큼은 의도적으로 프록시를 태우지 않습니다.**
프록시로 보내버리면 "우리 ISP가 DNS를 오염시키는가"를 확인할 수 없기 때문입니다
(`rkn_checker/network.py:24`의 주석 참고).

## 3. 확신도(Confidence) 모델 — 이 프로젝트의 차별점

`rkn_checker/models.py`에서 모든 판정에 신뢰도를 부여합니다.

| 표기 | 수준 | 의미 |
|---|---|---|
| `✗` | HIGH | 독립적 신호 2개 이상 일치 (DoH로 확인된 DNS 오염, HTTP 451, 알려진 스텁 문구) |
| `~ LIKELY` | MEDIUM | 검열 패턴과 일치하나 서버 측 문제 가능성을 배제 못 함 (TCP RST, TLS 중단) |
| `?` | LOW | 증상이 모호함 (타임아웃 등) |

정상 사이트(화이트리스트)를 **대조군**으로 함께 검사하여, 화이트리스트까지 실패하면
"검열이 아니라 회선 자체의 문제"라고 판정합니다 (`output.py`의 `_summary_verdict`).

증거가 뒷받침하는 만큼만 주장하는 이 설계가 이 프로젝트의 가장 큰 미덕입니다.

## 4. 폴더 구조

```
rkn-block-checker/
├── rkn_checker/          # 본체 (약 1,407줄)
│   ├── cli.py     (372)  # argparse 기반 CLI 진입점
│   ├── core.py    (211)  # 계층별 판정 로직 + 스레드풀 병렬 실행
│   ├── output.py  (209)  # 컬러 터미널 리포트 + 최종 서머리 판정
│   ├── network.py (127)  # TCP/TLS 프로브, SOCKS/HTTP 프록시 지원
│   ├── lists.py   (100)  # 커스텀 타겟 리스트(.txt/.json) 로더
│   ├── http.py     (94)  # HTTP 요청 + 스텁 페이지 문구 매칭
│   ├── dns.py      (76)  # 시스템 DNS vs DoH 비교
│   ├── models.py   (77)  # Verdict/Confidence Enum, CheckResult 데이터클래스
│   ├── targets.py  (62)  # 내장 화이트리스트 19개 / 블랙리스트 15개
│   ├── web.py      (71)  # 로컬 웹 대시보드 (NDJSON 스트리밍)
│   └── index.html        # 웹 UI 프론트엔드
├── tests/                # pytest 9개 파일
├── .github/workflows/    # CI(Python 3.10~3.12 매트릭스) + 릴리스 자동화
├── Dockerfile            # non-root 유저로 실행하는 슬림 이미지
└── docker-compose.yml
```

---

## 5. 설치 및 사용법

### 설치

```bash
# PyPI에서 설치
pip install rkn-block-checker

# 프록시 기능 포함
pip install 'rkn-block-checker[proxy]'

# 소스에서 개발 모드로 설치
pip install -e .
pip install -r requirements-dev.txt   # 테스트 도구 포함
```

의존성은 `requests` 하나뿐입니다. 프록시 사용 시에만 `PySocks`가 추가됩니다.

Docker 사용:

```bash
docker compose build
docker compose run --rm rkn-check --url https://example.com
```

### 기본 사용

```bash
rkn-check                            # 전체 스캔 (기본 34개 사이트)
rkn-check --url naver.com            # 단일 대상
rkn-check --url a.com --url b.com    # 여러 대상 (반복 지정)
rkn-check --white                    # 대조군만
rkn-check --black                    # 차단 대상군만
rkn-check --json                     # JSON 출력
rkn-check --timeout 10 --workers 20  # 타임아웃/동시 실행 수 조절
rkn-check -vv                        # 디버그 로그
rkn-check --no-self-info             # 공인 IP 조회 생략 (프라이버시)
rkn-check --identify                 # 자기 식별 User-Agent 사용
rkn-check --proxy socks5://127.0.0.1:1080
rkn-check startweb --port 8080       # 로컬 웹 대시보드
```

### 커스텀 목록으로 검사 (실무에서 가장 유용)

`my-list.txt`:

```txt
# 이름 = 주소  (주소만 적어도 됨)
우리api     = https://api.mycompany.com
사내위키    = https://wiki.internal.corp
github.com
```

```bash
rkn-check --black-file my-list.txt --white-file control.txt
```

JSON 형식(`{"이름": "주소"}`)도 지원합니다.

> 참고: `startweb`은 `--help`에 표시되지 않습니다. `cli.py:232`에서 argparse보다 먼저
> `argv[0] == "startweb"`을 가로채는 구조이기 때문입니다.

---

## 6. 자주 나오는 질문 정리

### 플러그인? 스킬? MCP?

**셋 다 아닙니다.** 순수한 독립 실행 파이썬 CLI 도구이며 AI와 무관합니다.

| 구분 | 정체 | 실행 주체 | 해당 여부 |
|---|---|---|---|
| CLI 도구 | 터미널 명령어 | 사람 | **해당** |
| MCP 서버 | AI가 쓰는 도구 | AI 에이전트 | 해당 없음 |
| 스킬 | Claude용 설명서(.md) | Claude | 해당 없음 |
| 플러그인 | 특정 앱의 확장 | 해당 앱 | 해당 없음 |

다만 셋 모두로 감쌀 수 있습니다. 특히 MCP 래핑이 쉽습니다 (아래 참고).

### API 토큰이 필요한가?

**전혀 필요 없습니다.** 코드 전체에 토큰/키 관련 항목이 없습니다.

대신 무료 공개 서비스 3개를 호출합니다.

| 서비스 | 용도 | 키 | 제한 |
|---|---|---|---|
| `cloudflare-dns.com` | DoH 대조 조회 | 불필요 | 사실상 무제한 |
| `ipinfo.io/json` | 내 IP/통신사 확인 | 불필요 | 하루 약 1,000회 |
| `ip-api.com/batch` | 접속 대상 국가코드 | 불필요 | 분당 약 45회 |

주의사항 2가지:

1. **`ip-api.com`은 `http://`로 호출됩니다** (`cli.py:135`). 무료 플랜이 HTTPS를
   지원하지 않기 때문인데, 평문 통신이라 조회 내역이 중간에 노출됩니다. 프라이버시가
   중요하면 이 기능을 비활성화하는 편이 좋습니다.
2. **자동화 실행 시 `--no-self-info`를 권장합니다.** `ipinfo.io` 일일 한도를 아끼고,
   결과 JSON에 공인 IP가 남지 않습니다.

### 왜 GitHub에서 인기가 있을까

원본 저장소는 스타 약 1.6k, 포크 63개, 기여자 5명 이상입니다. 소규모 프로젝트치고
매우 높은 수치이며, 이유는 다음과 같이 분석됩니다.

1. **실재하는 대규모 수요** — 러시아 인터넷 검열은 현재진행형 이슈이고, 수천만 명이
   매일 겪는 문제입니다.
2. **3초 안에 이해되는 포지셔닝** — "사이트 X가 안 열린다는 건 브라우저도 알려준다.
   중요한 건 어디서 깨졌는가다."라는 README 첫 문단.
3. **진입 장벽 0** — 설정·회원가입·API 키 없이 두 줄로 끝납니다.
4. **겸손한 확신도 설계** — 단정하지 않는 태도가 기술 커뮤니티의 신뢰를 얻습니다.
5. **README 상단의 실제 출력 화면** — 컬러 리포트 예시가 설득력을 만듭니다.
6. **실제로 좋은 코드 품질** — 모듈 분리, 테스트, CI 매트릭스, Docker, PyPI 배포까지
   갖춰져 기여하기 좋은 형태입니다.

### 로컬 AI 에이전트 구축에 도움이 되는가

크게 세 가지 층위에서 도움이 됩니다.

**(1) MCP 서버로 감싸기** — 구조가 이미 최적화되어 있습니다. 명확한 입력(URL),
구조화된 출력(`to_dict()`), 부작용 없음, 짧은 실행 시간.

```python
from mcp.server.fastmcp import FastMCP
from rkn_checker.core import check_url

mcp = FastMCP("network-doctor")

@mcp.tool()
def diagnose(url: str) -> dict:
    """사이트 접속 실패 원인을 DNS/TCP/TLS/HTTP 계층별로 진단"""
    return check_url(name=url, url=url).to_dict()
```

이러면 에이전트가 "안 됩니다" 대신 **"TLS 핸드셰이크에서 끊겼고 인증서 CN이
불일치합니다"**라고 답할 수 있습니다.

**(2) 에이전트 도구 설계의 교과서**

- 반환값을 구조화한다 (문자열이 아닌 dict)
- **`confidence` 필드를 반드시 넣는다** — LLM은 도구 출력을 무조건 사실로 믿는
  경향이 있습니다. `MEDIUM`과 근거 note를 함께 주면 에이전트가 "~로 보입니다"라고
  적절히 표현합니다. 할루시네이션 억제에 직접적인 효과가 있습니다.
- `notes`에 사람이 읽을 근거를 담는다
- 대조군을 둔다 (화이트리스트의 역할)

**(3) 실전 시나리오** — 자가 진단 후 재시도 판단, 인프라 감시 및 계층별 원인 알림,
배포 후 엔드포인트 검증 등.

### React나 PHP로 만들 수 있는가

**React(브라우저)만으로는 불가능합니다.** 브라우저 보안 샌드박스가 핵심 기능을
모두 차단합니다.

| 필요 기능 | 브라우저 | 이유 |
|---|---|---|
| DNS 직접 조회 | 불가 | 해당 API 없음 |
| raw TCP 연결 | 불가 | 소켓 API 없음 (WebSocket은 별개) |
| TLS 핸드셰이크 관찰 | 불가 | 브라우저가 결과를 노출하지 않음 |
| 응답 본문 읽기 | 불가 | CORS 차단 |
| **실패 원인 구분** | **불가** | **전부 `TypeError: Failed to fetch`로 뭉개짐** |

마지막 항목이 결정적입니다. 원인을 구분할 수 없으면 이 도구의 존재 이유가 사라집니다.
따라서 **React는 화면(프론트엔드)만 담당하고, 진단은 백엔드가 수행**해야 합니다.
이 프로젝트가 이미 그 구조입니다 (`web.py` + `index.html`).

**Node.js는 잘 됩니다.** `net`, `tls`(servername으로 SNI 지정 가능), `dns` 모듈이
모두 제공되고, 비동기가 기본이라 스레드풀 없이 `Promise.all`로 병렬 처리가 됩니다.

**PHP는 가능하지만 어색합니다.** `dns_get_record`, `stream_socket_client`,
`stream_socket_enable_crypto`로 구현은 되지만, 스레드가 없어 병렬 처리가 까다롭고
(`curl_multi`/Swoole/ReactPHP 필요), 웹서버 실행 시 타임아웃 문제가 있으며,
실시간 스트리밍 구현이 번거롭고, CLI 배포 생태계가 약합니다.

권장 구성:

```
┌─────────────────┐    HTTP / NDJSON 스트리밍    ┌──────────────────┐
│  React 프론트   │ ◄──────────────────────────► │  진단 백엔드     │
│  (대시보드 UI)  │                              │  Python or Node  │
└─────────────────┘                              └──────────────────┘
                                                          │
                                            DNS / TCP / TLS / HTTP 프로브
```

| 상황 | 권장 |
|---|---|
| 현재 프로젝트를 확장 | Python 백엔드 유지 + React 프론트 (가장 빠름) |
| JS/TS로 통일 | Node.js 백엔드 + React |
| PHP 서버뿐 | PHP CLI로 스캔 후 JSON 저장 → 웹은 그 JSON만 표시 |

`web.py`에 `/api/init`, `/api/self-info`, `/api/scan` 세 개의 API가 이미 있고
NDJSON 스트리밍도 구현되어 있어, React에서 그대로 호출하면 됩니다.

---

## 7. 이 도구가 실무에 주는 가치

원래 용도(러시아 검열 진단)는 국내 사용자에게 직접적 효용이 적습니다. 타겟 목록이
전부 `.ru` 도메인이고 차단 문구도 러시아어입니다. 그러나:

### 목록만 교체하면 범용 네트워크 진단기

```bash
rkn-check --json --no-self-info --black-file our-services.txt > "snap/$(date -I).json"
```

- 회사 방화벽/망분리 환경에서 어떤 서비스가 어느 계층에서 막히는지 진단
- 장애 시 DNS 문제인지, 서버 문제인지, 인증서 문제인지 즉시 구분
- 인증서 만료/CN 불일치 모니터링 (`tls_cert_cn` 수집 중)
- 크론 등록 후 JSON 스냅샷을 축적하면 시계열 가용성 기록

### 코드 자체가 학습 자료

- 계층별 조기 종료(early-exit) 진단 패턴
- `Enum` + `dataclass` 기반 결과 모델링과 JSON 직렬화
- 탐지와 확신도의 분리 설계 (알림 시스템 설계에도 그대로 적용 가능)
- `ThreadPoolExecutor` + `as_completed` 스트리밍 제너레이터 패턴
- 패키징 풀세트: `pyproject.toml`, 콘솔 스크립트, optional-dependencies,
  Docker, GitHub Actions 매트릭스 CI, 릴리스 자동화

> 이 도구는 차단을 우회하는 도구가 아니라 **관측하는 도구**입니다.
> 체온계이지 약이 아닙니다.

---

## 8. 수익화 전략 분석

### 출발점: 냉정한 전제 3가지

1. **이 도구 자체는 팔 수 없다.** MIT 라이선스라 누구나 무료로 쓰고, 경쟁자도
   같은 코드로 동일한 것을 만들 수 있습니다. 코드는 해자가 아닙니다.
2. **"RKN 차단 진단"은 시장이 너무 작다.** 돈 낼 사람이 적고, 법적 리스크가 크고,
   애초에 무료여야 하는 성격의 도구입니다.
3. **팔 수 있는 것은 "계층별 원인 진단"이라는 관점 하나뿐이다.**

```
기존 모니터링:  "사이트 다운됨"                   <- 레드오션
이 프로젝트:    "TLS 핸드셰이크에서 끊김.          <- 아직 빈틈 있음
                인증서 CN 불일치 의심"
```

### 시장 현실 (2026년 조사 기준)

| 구분 | 가격대 | 시사점 |
|---|---|---|
| 업타임 모니터링 (UptimeRobot) | 무료 50개 → $7~12/월 | 가격이 이미 바닥. 진입 비권장 |
| 인시던트 플랫폼 (Better Stack) | $29/월 per seat | 팀 협업에 과금 |
| 소기업 플랜 일반 | $10~50/월 | |
| 프로 플랜 | $50~200/월 | |
| 엔터프라이즈 | $200+/월 | 마진 구간 |
| SSL 인증서 모니터링 | $5~20/월, 개당 $0.72~ | 니치, 경쟁 낮음 |
| 웹 접근성 모니터링 | $100~500/월 | "규정 준수" 프레임의 단가 효과 |

핵심 인사이트 두 가지:

- **"모니터링"으로 포지셔닝하면 $7짜리 싸움**에 끌려갑니다.
- **"규정 준수 / 사고 예방"으로 포지셔닝하면 $100~500** 구간이 열립니다.
  같은 기술인데 10배 차이입니다.

### 아이디어 순위

#### 1위 — "서비스 사망 원인 분석기" (Open Core SaaS)

> "당신 서비스가 죽었다는 건 다들 알려줍니다. 우리는 왜 죽었는지 알려줍니다."

CLI는 무료 오픈소스로 유지(= 유입 깔때기), 유료는 시간·협업·자동화에 과금.

| 무료 (CLI/셀프호스팅) | 유료 (SaaS) |
|---|---|
| 1회 진단 | 히스토리 및 추이 그래프 |
| 터미널 출력 | 슬랙/이메일 알림 |
| 로컬 실행만 | 여러 지역에서 자동 실행 |
| 개인 | 팀 계정, 권한, 공유 링크 |
| — | 인시던트 리포트 자동 생성 |

- 장점: 오픈소스 유입, 차별점 유지, 개발자 타겟 마케팅 용이, 1인 창업 가능
- 리스크: 무료 CLI로 충분해서 전환이 안 될 수 있음 →
  **"혼자 쓰면 무료, 팀이 쓰면 유료"** 경계를 처음부터 뾰족하게 그어야 함
- 과금: 개인 무료 / 팀 $29~49/월 / 기업 $199~/월

#### 2위 — 한국 기업 망분리·방화벽 진단 (B2B)

금융·공공·대기업 망분리 환경에서 "어느 구간에서 막혔는지 증거를 제시"하는 도구.

- 장점: 단가가 높음(온프레미스 라이선스), 교체 비용이 커서 이탈 적음,
  외산 SaaS가 진입 못 하는 폐쇄망 영역, 보안 심사 자체가 진입장벽
- 리스크: 영업 사이클 6~18개월, 레퍼런스 1호 확보가 난관,
  일의 80%가 영업·제안서·보안심사
- 과금: 구축비 + 연 유지보수 20%
- **관련 업계 인맥이 있다면 1위로 올라갈 수 있는 아이템**

#### 3위 — 인증서·도메인 만료 파수꾼 (Micro-SaaS)

가장 작고, 가장 빨리 만들 수 있고, 가장 확실하게 검증되는 아이템.

- 시장: 월 검색량 약 390회, 경쟁 낮음, 목표 MRR $5K~15K
- 기존 강자: TrackSSL (인증서당 $0.72~)
- **유리한 점: `check_tls()`가 이미 인증서 CN을 수집 중** (`network.py:_extract_cn`).
  만료일 필드만 추가하면 거의 완성. "인증서 + DNS + 도달성"을 한 번에 보는 조합은
  기존 서비스가 하지 않음
- 진짜 타겟은 개발사가 아니라 **에이전시·호스팅사·프리랜서**.
  고객사 인증서 만료는 평판이 걸린 사고이므로 지불 의향이 높음
- 리스크: 단가가 낮아 볼륨 필요, 무료 대안 존재 → "여러 고객사 일괄 관리"로 차별화
- 과금: 무료 5개 / $9 50개 / $29 200개 / $99 무제한 + 화이트라벨

#### 4위 — 글로벌 도달성 체크 (조사 후 순위 하향)

무료 경쟁자가 너무 많아 "1회 체크"는 완전히 상품화되었습니다.

| 서비스 | 제공 | 가격 |
|---|---|---|
| ViewDNS.info | 중국 다지점 테스트 | 무료 |
| GreatFire | 실시간 중국 접근성 | 무료 |
| 21YunBox | 300개 지점 + 서드파티 의존성 분석 | 무료 |
| vpnMentor | 실시간 다지점 | 무료 |
| WebsitePulse | 30일 무료 모니터링 | 무료 |

살아남는 각도:

- 한국어 + 국내 결제 + 국내 CS
- 1회 체크가 아닌 **상시 감시 + 변화 알림** ("지난주엔 됐는데 어제부터 막힘")
- 진단에서 끝나지 않고 처방까지 (AppInChina의 모델)
- 특정 산업 집중 (게임사 해외 서비스 등)

과금: 도메인당 월 3~10만원, 리포트 건당 50~200만원

#### 5위 — AI 에이전트용 네트워크 진단 MCP

- 장점: 신생 시장 선점, 기술적으로 가장 빨리 구현 가능
- 리스크: 현재 MCP 단독으로 과금하는 문화가 없음
- **결론: 단독 상품이 아니라 1위 아이템의 차별화 기능으로 편입하는 것이 적절**

### 종합 비교

| | 시장 크기 | 난이도 | 수익까지 | 경쟁 | 1인 가능 | 총평 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| 1위 원인 분석 SaaS | 중 | 중 | 6~12개월 | 중 | 가능 | 최우선 |
| 2위 망분리 B2B | 소·고단가 | 상 | 12~18개월 | 하 | 어려움 | 인맥 있으면 1순위 |
| 3위 인증서 파수꾼 | 소 | 하 | 2~4개월 | 중 | 가능 | 검증용 최적 |
| 4위 글로벌 도달성 | 중 | 중 | 6~12개월 | 상 | 가능 | 차별화 필수 |
| 5위 MCP | 미형성 | 하 | 불확실 | 하 | 가능 | 기능으로 편입 |

### 90일 실행 플랜

**1~30일 — 가장 작은 것부터 출시 (3위 아이템)**

- 인증서 만료일 수집 기능 추가
- 도메인 등록 + 랜딩 페이지 1장
- 무료 진단 웹페이지 공개 → 이메일 수집
- 목표: 이메일 100개

**31~60일 — 유료 전환 실험**

- 알림 기능 + 대시보드
- 목표: 유료 고객 10명 (월 1만원이라도)
- 여기서 0명이면 아이템을 바꿉니다. 만들기 전에 알 수 있는 것이 이득입니다.

**61~90일 — 확장 결정**

- 성과가 있으면 1위 아이템(원인 분석 SaaS)으로 확장
- 반응이 없으면 2위(B2B) 쪽 인맥 탐색으로 전환
- 목표: MRR 100만원

> 핵심 원칙: 코드를 더 짜서 해결하려 하지 말고, 돈 내는 사람을 먼저 찾습니다.
> 이 프로젝트의 코드는 이미 충분히 좋습니다. 부족한 것은 코드가 아니라 고객입니다.

### 리스크 5가지

1. **라이선스** — MIT라 상업적 이용은 자유지만 저작권 표시와 라이선스 전문을
   반드시 포함해야 합니다.
2. **검열 진단의 상업화는 지양** — 특정 국가 대상 서비스는 현지 법 위반 소지가
   있습니다. 진단 대상에서 제외하는 편이 안전합니다.
3. **프로브 인프라가 원가의 대부분** — 다지역 서버 비용. 초기에는 저가 VPS로
   3~4개 지역만 시작합니다.
4. **"모니터링"이라는 단어를 피할 것** — 그 순간 $7짜리 서비스와 비교됩니다.
   "원인 분석", "장애 진단", "사고 예방"으로 부릅니다.
5. **파는 것은 기술이 아니라 시간** — 고객이 사는 것은 진단 알고리즘이 아니라
   "원인 찾느라 날린 3시간"입니다.

---

## 9. 코드에서 발견한 개선 포인트

분석 과정에서 확인된 사항들입니다. 모두 즉시 장애를 일으키지는 않습니다.

1. **`rkn_checker/core.py` — `Optional` import 누락**
   35, 175, 202행에서 `Optional[str]`을 사용하지만 `from typing import Iterator`만
   있습니다. `from __future__ import annotations` 덕분에 런타임 오류는 없지만,
   mypy나 `typing.get_type_hints()` 사용 시 실패합니다.

2. **`rkn_checker/output.py:199` — 오타**
   `"Likely in an an RKN-blocked zone"` — `an`이 중복되었습니다.

3. **`rkn_checker/web.py` — `/api/scan`이 `proxy_url`을 전달하지 않음**
   CLI에는 `--proxy`가 있으나 웹 UI에서는 프록시를 사용할 수 없습니다.

4. **`startweb` 서브커맨드가 `--help`에 노출되지 않음**
   argparse보다 먼저 가로채는 구조라 문서를 보지 않으면 존재를 알기 어렵습니다.

5. **`ip-api.com` 호출이 평문 HTTP** (`cli.py:135`)
   무료 플랜 제약이지만, 검열 진단 도구로서는 프라이버시상 아쉬운 지점입니다.

---

## 10. 참고 자료

### 시장 조사 출처

- [10 Best Website Uptime Monitoring Tools in 2026 — Better Stack](https://betterstack.com/community/comparisons/website-uptime-monitoring-tools/)
- [Better Stack vs UptimeRobot 비교](https://betterstack.com/community/comparisons/better-stack-vs-uptimerobot/)
- [UptimeRobot Pricing and Review 2026 — CubeAPM](https://cubeapm.com/blog/uptimerobot-pricing-and-review/)
- [Test Your Website in China — AppInChina](https://appinchina.co/test-your-site-in-china/)
- [China Firewall Test — 21YunBox](https://www.21cloudbox.com/tools/china-firewall-test.html)
- [Chinese Firewall Test — ViewDNS.info](https://viewdns.info/chinesefirewall/)
- [SSL Certificate Monitoring 마이크로SaaS 분석 — MicroSaaSHQ](https://microsaashq.com/saas-ideas/ssl-certificate-monitoring-807)
- [TrackSSL](https://trackssl.com/)
- [5 Best SSL Certificate Monitoring Services in 2026](https://firstsiteguide.com/best-ssl-certificate-monitoring-services/)
- [웹 접근성 모니터링 가격 분석 — TestParty](https://testparty.ai/blog/website-accessibility-cost-2025)

### 프로젝트 문서

- [README.md](../README.md) — 원본 영문 문서
- [docs/sample-output.json](./sample-output.json) — JSON 출력 예시
- [docs/sample-output.svg](./sample-output.svg) — 터미널 출력 예시
