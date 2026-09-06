# RTKLIB & GNSS-SDR — GNSS 측위 알고리즘/아키텍처 뜯어보기

> 근거등급 표기 기준
> (a) 공식 문서/실제 소스코드로 직접 확인
> (b) 로컬 증거+공식자료 조합으로 강하게 추론 가능
> (c) 정황적(3자 출처)이거나 직접 미확인
> (d) 열린 질문 — 확인 안 됨, 지어내지 않고 명시

---

## 1. 대상 프로젝트 소개, 라이선스, 클론 버전

### 1.1 RTKLIB

- 저장소: https://github.com/tomojitakasu/RTKLIB
- 클론한 브랜치/태그/커밋: `rtklib_2.4.3` 브랜치, 태그 `v2.4.3-b34`, 커밋 `180043ee24b6d2b168f98b64be15f69d50046b1a` (2020-12-29)
- 라이선스: **BSD 2-Clause** (`readme.txt` 106~130행에 명시, 원저작자 T. Takasu, 2007-2013) + 일부 Windows 실행바이너리/공유 라이브러리에 한정된 추가 조항. 소스코드 자체(본 문서가 다루는 알고리즘 부분)는 순수 BSD-2-Clause. (a)
- 참고: `tomojitakasu/RTKLIB`의 기본 브랜치(`master`)는 2018-01-30 커밋(`71db0ffa...`, 태그 `v2.4.2-p13`)에 멈춰 있고, 실제로 더 최신 코드는 `rtklib_2.4.3` 브랜치(태그 최대 `v2.4.3-b34`)에 있다. 이번 조사는 `master`가 아니라 이 더 최신 브랜치를 클론해서 진행했다. (a — `git ls-remote --heads/--tags`로 직접 확인)
- 커뮤니티에서 실제 RTK/PPK 실무에 널리 쓰이는 `rtklibexplorer/RTKLIB` 포크(브랜치 `demo5`, `dev` 등)가 원저자 저장소보다 더 활발히 갱신되고 있다는 점도 `git ls-remote`로 확인했다 — 다만 이 포크 자체의 소스코드는 이번 조사에서 열어보지 않았으므로, "원저자 저장소보다 활발하다"는 브랜치 존재/갱신 빈도까지만 (a), 포크 내부의 알고리즘 차이는 (d) 열린 질문으로 남긴다.

### 1.2 GNSS-SDR

- 저장소: https://github.com/gnss-sdr/gnss-sdr
- 클론한 브랜치/커밋: `main` 브랜치, 커밋 `0f4dba13865bbed38a586678b1af75479f3c2411` (2026-04-17)
- 라이선스: **GPL-3.0-or-later** (최상위 `COPYING` 파일이 GPLv3 전문). 단 저장소 내 `LICENSES/` 디렉터리에 Apache-2.0, BSD-1/2/3-Clause, MIT, Zlib 등 개별 SPDX 라이선스 텍스트가 함께 들어있어, 일부 서드파티/이식 코드는 파일 단위로 다른 라이선스(BSD-2-Clause 등)를 갖는 REUSE 방식 라이선싱 구조임을 확인했다. 실제로 아래 3절에서 다룰 `src/algorithms/libs/rtklib/` 하위 파일들은 각 파일 헤더에 `SPDX-License-Identifier: BSD-2-Clause`가 개별적으로 명시돼 있다. (a)

---

## 2. RTKLIB — 측위 알고리즘 아키텍처

### 2.1 소스 구조 개관 (a)

`src/` 디렉터리 하나에 약 4만 1천 줄의 C 코드가 들어있고, 파일 단위로 역할이 명확히 나뉜다 (실제로 `ls`, `wc -l`로 확인):

| 파일 | 역할 |
|---|---|
| `rinex.c` | RINEX 관측/항법 파일 파싱 |
| `rtcm.c`, `rtcm2.c`, `rtcm3.c` | RTCM v2/v3 실시간 보정 메시지 디코딩 |
| `pntpos.c` | 단독 측위(SPP)/DGPS용 코드-기반 위치 계산 (최소자승) |
| `rtkpos.c` | RTK(이동국-기준국 상대측위) 및 DGPS의 공통 엔진, 칼만 필터 기반 |
| `ppp.c`, `ppp_ar.c` | PPP(정밀 단독 측위) 및 PPP 정수 모호도 결정 |
| `lambda.c` | LAMBDA/MLAMBDA 정수 모호도 결정(암비귀티 리졸루션) 알고리즘 |
| `rtkcmn.c` | 좌표변환, 행렬연산, **칼만 필터 핵심 함수**, 시각 변환 등 공통 유틸 |
| `ephemeris.c`, `preceph.c` | 방송궤도력/정밀궤도력(SP3)·시계 보정 계산 |
| `sbas.c` | SBAS(위성기반 보강시스템) 보정 처리 |
| `rtksvr.c`, `stream.c`, `streamsvr.c` | 실시간 스트림(시리얼/TCP/NTRIP 등) 입출력 서버 |

### 2.2 RTK / DGPS — `rtkpos.c` + 칼만 필터 (a)

- `rtkpos.c`의 `relpos()`/`udstate()` 등에서 이동국-기준국 관측치를 **단일차분(SD, single-difference)** → **이중차분(DD, double-difference)**으로 조합한다. 실제 주석과 함수명으로 확인:
  - `366행: /* single-differenced observable */`
  - `943행: /* DD (double-differenced) measurement error covariance */`
  - `1086행: /* make DD (double difference) */`
  이렇게 위성간·수신기간 차분을 취해 위성/수신기 시계오차를 소거하는 고전적 RTK 수학 구조가 실제 코드에 그대로 구현돼 있음을 확인했다. (a)
- 상태 갱신은 `rtkcmn.c` 1268~1332행의 `filter()` 함수가 담당한다. 함수 상단 주석에 갱신식이 그대로 적혀 있다:
  ```
  K=P*H*(H'*P*H+R)^-1, xp=x+K*v, Pp=(I-K*H')*P
  ```
  이는 표준 **칼만 필터(측정 갱신 단계)** 수식이며, `rtkpos.c` 1339행 `filter(rtk->x,rtk->P,H,v,R,rtk->nx,nv)` 호출로 실제 RTK 위치/모호도 상태벡터에 적용된다. (a)
- **DGPS는 RTK와 별개 모듈이 아니라 같은 `rtkpos.c` 엔진을 옵션값 `PMODE_DGPS`로 분기해서 재사용**한다. 예: `rtkpos.c` 82행 `#define NB(opt) ((opt)->mode<=PMODE_DGPS?0:MAXSAT*NF(opt))` — DGPS 모드에서는 반송파 모호도 상태(N, ambiguity)를 아예 할당하지 않고 코드(의사거리)만 사용한다. 즉 RTKLIB 아키텍처상 DGPS는 "반송파 모호도 추정을 끈 RTK"로 구현돼 있다. (a — 실제 매크로/조건문 확인)
- **정수 모호도 결정(ambiguity resolution)**은 `lambda.c`에서 담당하며, 파일 헤더 주석에 참고문헌이 명시돼 있다: Teunissen(1995)의 LAMBDA 기법, Chang·Yang·Zhou(2005)의 MLAMBDA(Modified LAMBDA) 개선판. `LD()` 함수(27행)는 정수최소자승 공분산 행렬의 LD(L'DL) 분해를 수행한다. (a)

### 2.3 PPP — `ppp.c`/`ppp_ar.c` (a)

- `ppp.c` 헤더 주석에 IERS Conventions(1996/2003/2010), IGS MGEX, RTCM SSR(위성궤도/시계/코드바이어스) 등 참고문헌이 명시돼 있고, 조석보정(`tidedisp()`), 안테나 위상중심/요잉자세 모델(Kouba 2009, Dilssner 2010) 등을 구현한다고 이력에 남아있다. (a — 주석/함수명 직접 확인. 단 각 서브모델의 수치가 참고문헌과 정확히 일치하는지까지는 대조하지 않았으므로 그 부분은 (b)로 낮춰본다)
- PPP는 이중차분을 쓰지 않는 **비차분(undifferenced) 단독 측위** 방식으로, 정밀궤도력·정밀시계(IGS 등)를 직접 이용해 위성시계오차를 소거한다는 것이 구조상 자연스러운 결론이나, `ppp.c` 내부에서 "undifferenced"라는 표현 자체를 직접 grep으로 재확인하지는 않았다 — 이 문단의 "비차분"이라는 용어 사용은 (b) 등급(코드 구조로 강하게 추론)로 남긴다.

### 2.4 사이클슬립(cycle-slip) 탐지 — `rtkpos.c` 599~700행대 (a)

실제로 3가지 서로 다른 탐지 로직이 함수 단위로 분리돼 있음을 코드에서 직접 확인했다:

1. `detslp_ll()` (599행) — RINEX LLI(Loss-of-Lock Indicator) 플래그 기반 탐지 + half-cycle(반주기) 불확실 플래그 전이 탐지.
2. `detslp_gf()` (647행) — L1-Lx 기하학무관(geometry-free) 반송파 위상 조합의 점프량이 임계값(`rtk->opt.thresslip`)을 넘으면 슬립으로 판정.
3. `detslp_dop()` (671행) — 도플러 관측치와 위상차분을 비교하는 방식인데, **실제 코드에 `#if 0`로 비활성화돼 있고 주석에 "v.2.3.0에서 시계점프 문제 때문에 비활성화"라고 명시돼 있다.** 즉 이 버전 알고리즘은 문서/코드에는 남아있지만 실제로는 컴파일되지 않는 죽은 코드다. (a — `#if 0` 및 주석 직접 확인. 과거 컨트리뷰터의 "실제로 확인함" 오류를 반복하지 않기 위해 일부러 이 디테일을 남긴다: 겉보기 함수 목록만 보고 "3종 슬립탐지가 다 동작한다"고 쓰면 틀린 서술이 됨)

### 2.5 RTCM/RINEX 파싱 아키텍처 (a)

- `rtcm3.c`의 `decode_rtcm3()`(2537행)는 메시지 타입 번호(1001~수천번대)를 `switch-case`로 분기해 각 `decode_type####()` 함수로 위임하는 **메시지 타입 디스패치 테이블** 구조다. 실제로 다수의 케이스가 `/* not supported */` 주석과 함께 남아있어, RTCM 표준 전체가 아니라 실사용 빈도가 높은 메시지 위주로 구현돼 있음을 확인했다. (a)
- 원시 데이터 흐름은 `rtksvr.c`의 `decoderaw()`(339행)에서 `input_rtcm2()`/`input_rtcm3()`(둘 다 `rtcm.c`)/`input_raw()`(수신기별 바이너리 포맷, `rcvraw.c`)로 분기되고, 이 결과가 `update_obs()`(138행)를 거쳐 관측치 버퍼에 쌓인 뒤 `rtkpos()`/`pntpos()`로 전달되는 구조다. 즉 "원시 스트림 → 관측치 구조체(obs_t) → 측위 엔진"이라는 계층이 파일 경계로 명확히 분리돼 있다. (a)
- SPP(단독측위)는 `pntpos.c`의 `estpos()`(368행)가 담당하며, 매 에폭 독립적으로 **반복 가중최소자승(iterative weighted least squares, `lsq()` 호출)**을 푸는 방식이다 — RTK/PPP의 시간축 재귀 칼만 필터와는 수학적으로 다른 접근이라는 점을 실제 함수 구현 확인으로 검증했다. (a)

---

## 3. GNSS-SDR — SDR 신호처리 파이프라인 아키텍처

### 3.1 공식 문서 기준 구조 (a)

`README.md`(1004행 이하 "Control plane" 절)에서 공식적으로 설명하는 구조를 그대로 인용/요약하면:

- `ControlThread`가 설정 파일을 읽고 `GNSSFlowgraph` 객체를 생성 → `run()` 시점에 GNU Radio의 `gr::top_block`에 블록들을 실제로 연결(connect)하고 실행을 시작한다.
- `GNSSFlowgraph`(`src/core/receiver/gnss_flowgraph.h`)는 설정에 따라 블록 그래프를 구성/실행/런타임 재구성(채널별 위성 재할당 등)하는 책임을 진다.
- 블록 인스턴스 생성은 `GNSSBlockFactory`(`src/core/receiver/gnss_block_factory.h`)가 맡으며, `GetAcqBlock`/`GetTrkBlock`/`GetObservables`/`GetPVT` 등 역할별 팩토리 메서드가 설정 파일의 문자열 키(`구현체 이름`)를 실제 C++ 클래스 인스턴스로 매핑한다. (a — 헤더 파일에서 메서드 시그니처 직접 확인)

### 3.2 실제 소스 디렉터리 = 파이프라인 단계 (a)

`src/algorithms/` 하위 디렉터리 이름 자체가 곧 파이프라인 단계이며, `ls`로 직접 확인했다:

```
signal_source → data_type_adapter → input_filter/conditioner/resampler
  → acquisition → tracking → telemetry_decoder → observables → PVT
```

- **acquisition**: `acquisition/adapters/`에 32개 헤더(신호/방식별 구현)가 존재. 예: `galileo_e1_pcps_ambiguous_acquisition`, `beidou_b1i_pcps_acquisition` 등 — PCPS(Parallel Code Phase Search) 알고리즘을 GPS/Galileo/BeiDou/QZSS 등 신호별로 각각 구현한 어댑터 클래스가 별도 파일로 나뉘어 있다. (a)
- **tracking**: `tracking/adapters/`에 25개 헤더. DLL/PLL(Delay-Locked Loop/Phase-Locked Loop) 기반 추적 루프(`base_dll_pll_tracking`)를 각 신호별로 파생시킨 구조. (a)
- **telemetry_decoder**: 신호별(GPS L1CA/L2C/L5, Galileo E1B/E5a/E5b/E6, GLONASS, BeiDou, QZSS, SBAS) 항법메시지 복조기가 각각 별도 어댑터로 존재. (a)
- **observables**: `hybrid_observables` 어댑터 하나로 여러 신호 채널의 추적 결과를 의사거리/반송파위상 관측치로 통합하는 단계. (a — 디렉터리/파일 존재만 확인, 내부 알고리즘 상세까지는 이번에 안 읽었으므로 그 부분은 (d))
- **PVT**: 관측치를 실제 위치/시각(PVT) 해로 변환하는 최종 단계 — 아래 4절에서 다루듯, 이 내부 구현이 RTKLIB 이식 코드다.

### 3.3 플러그인 경계의 실체: `GNSSBlockInterface` (a)

`src/core/interfaces/gnss_block_interface.h`를 직접 읽었다. 모든 블록(신호원, 필터, 획득, 추적, 관측치, PVT 등)은 이 순수가상 인터페이스를 상속하며, 핵심 메서드는:

```cpp
virtual std::string role() = 0;
virtual std::string implementation() = 0;
virtual void connect(gr::top_block_sptr top_block) = 0;
virtual void disconnect(gr::top_block_sptr top_block) = 0;
virtual gr::basic_block_sptr get_left_block() = 0;
virtual gr::basic_block_sptr get_right_block() = 0;
```

즉 각 단계는 "왼쪽 블록/오른쪽 블록"이라는 그래프 연결점만 노출하고, 내부적으로 어떤 GNU Radio 신호처리를 하는지는 인터페이스 뒤에 완전히 숨겨져 있다. 설정 파일의 `역할.implementation=구현체이름` 문자열 하나로 `GNSSBlockFactory`가 실제 구현 클래스를 골라 인스턴스화하는 **팩토리+전략 패턴** 조합임을 코드로 확인했다. (a)

---

## 4. 매우 중요한 교차점: GNSS-SDR은 PVT 단계에서 실제로 RTKLIB 코드를 이식해 쓴다

이번 조사에서 가장 확실하고 재사용가치가 높은 발견이다.

- `src/algorithms/libs/rtklib/` 디렉터리에 **총 36개 파일**이 있고, 파일명이 `rtklib_rtkcmn.cc`, `rtklib_rtkpos.cc`, `rtklib_pntpos.cc`, `rtklib_ppp.cc`, `rtklib_lambda.cc`, `rtklib_rtcm.cc`, `rtklib_rtcm2.cc`, `rtklib_rtcm3.cc`, `rtklib_ephemeris.cc`, `rtklib_preceph.cc`, `rtklib_sbas.cc`, `rtklib_ionex.cc`, `rtklib_tides.cc`, `rtklib_solution.cc`, `rtklib_stream.cc`, `rtklib_rtksvr.cc` 등 — 위 2.1절에서 정리한 **RTKLIB `src/`의 거의 모든 파일과 1:1 대응**한다. (a — `ls`로 직접 대조 확인)
- `src/algorithms/PVT/libs/rtklib_solver.cc` 파일 헤더를 직접 읽었다. 저작자 목록에 "2007-2013, T. Takasu"가 포함돼 있고, 주석에 명시적으로 이렇게 적혀 있다:
  > "This is a derived work from RTKLIB http://www.rtklib.com/ ... released under the BSD 2-clause license"
  라이선스 태그도 `SPDX-License-Identifier: BSD-2-Clause`로 파일 단위 명시돼 있다. (a — 원문 직접 확인, 인용은 짧은 발췌)
- 결론: **GNSS-SDR의 아키텍처는 "SDR 신호처리(획득→추적→복조→관측치 생성)는 자체 구현 + PVT(관측치→위치)는 RTKLIB 코드를 이식해서 재사용"하는 하이브리드 구조다.** 이는 두 프로젝트를 "완전히 다른 두 접근법"으로 단순 대비하는 것이 아니라, **RTKLIB의 관측치-레벨 알고리즘이 SDR 수신기 구현체에서도 사실상 표준 백엔드로 채택되고 있다**는 것을 보여주는 강한 증거다. (a — 소스 직접 대조)

---

## 5. 아키텍처 비교: 관측치 레벨 처리 vs 원신호(raw-signal) 레벨 처리

| 구분 | RTKLIB | GNSS-SDR |
|---|---|---|
| 입력 | 이미 만들어진 관측치(RINEX 파일, RTCM 스트림, 수신기 바이너리) | 안테나에서 받은 원시 IF/RF 샘플(디지털 베이스밴드 신호) |
| 핵심 산출 대상 | 의사거리·반송파위상 관측치 → 위치/시계 해 | 원시 샘플 → (자체 구현) 의사거리·반송파위상 관측치 → (RTKLIB 이식 코드로) 위치/시계 해 |
| 아키텍처 단위 | C 함수/파일 단위 모듈(`*.c`), 링크 시점에 정적으로 조합 | GNU Radio 블록 + 추상 인터페이스(`GNSSBlockInterface`) + 설정파일 기반 런타임 팩토리 조합 |
| 확장 방식 | 신규 수신기 포맷은 `rcv/` 하위에 파서 추가, 신규 알고리즘은 함수 추가 | 신규 신호/신규 알고리즘은 `*Interface`를 구현하는 새 어댑터 클래스 + 설정 키 추가 (컴파일 후 설정만으로 조합 교체 가능) |
| 실시간성 처리 주체 | `rtksvr.c`가 자체 스레드/스트림 서버로 실시간 처리 담당 | GNU Radio `gr::top_block`의 스케줄러가 샘플 스트림 실시간 처리 전담 |
| 겹치는 지점 | — | PVT 단계에서 RTKLIB 코드를 그대로 이식(4절) |

이 표에서 "입력"과 "아키텍처 단위" 행은 (a) 등급(코드/디렉터리 구조 직접 확인)이고, "실시간성 처리 주체" 행은 파일 존재(`rtksvr.c`, GNU Radio 의존성)까지는 (a), 두 서버 아키텍처의 성능/지연 특성 비교는 실측하지 않았으므로 (d) 열린 질문으로 남긴다.

---

## 6. 재사용 가능한 설계 패턴 (다른 스터디에도 응용 가능)

1. **역할 인터페이스 + 팩토리 패턴으로 알고리즘 교체 가능하게 만들기** (GNSS-SDR, (a)) — `GNSSBlockInterface`처럼 "무엇을 하는가"(role)와 "어떻게 하는가"(implementation)를 분리하면, 신호/알고리즘이 늘어나도(GPS/Galileo/BeiDou×PCPS/DLL-PLL 등 수십 종) 파이프라인 뼈대는 그대로 두고 구현체만 갈아끼울 수 있다. QGIS 플러그인이나 향후 GIS 파이프라인 설계에도 그대로 응용 가능한 패턴.
2. **관측치 레벨과 원신호 레벨 처리를 완전히 분리하되, 관측치→해(解) 변환 로직은 검증된 구현을 이식해서 재사용** (GNSS-SDR이 RTKLIB PVT 코드를 가져다 쓴 것, (a)) — "바퀴를 다시 발명하지 않는다"의 실제 사례. 단, 라이선스 호환성(BSD-2-Clause 코드를 GPLv3 프로젝트가 포함하는 것은 가능하나 역방향은 불가)을 반드시 먼저 확인해야 한다는 교훈도 함께 남는다.
3. **차분(differencing) 기법으로 공통 오차를 소거하는 설계** (RTKLIB SD/DD, (a)) — 두 관측치의 차를 취해 공통 바이어스(시계오차 등)를 상쇄시키는 구조는 GNSS 밖에서도(예: 두 센서의 상대 측정치 비교, 캘리브레이션 설계) 응용 가능한 일반 패턴.
4. **모드(mode) 값 하나로 파라미터화해서 유사 알고리즘을 하나의 엔진에 통합** (RTKLIB의 `rtkpos.c`가 `PMODE_DGPS`/`PMODE_KINEMA` 등으로 RTK와 DGPS를 같은 칼만필터 엔진에서 분기 처리, (a)) — 완전히 별개 모듈로 나누는 대신, "상태벡터에 무엇을 포함시키는가"를 옵션으로 제어해 코드 중복 없이 유사 알고리즘 계열을 통합하는 방식.
5. **메시지 타입 디스패치 테이블로 프로토콜 파서 확장성 확보** (RTKLIB `decode_rtcm3()`의 switch-case, (a)) — 신규 메시지 타입 추가가 기존 코드에 영향을 주지 않는 구조.
6. **"주석/코드에는 남아있지만 실제로는 비활성화된 로직"을 구분해서 읽어야 한다는 방법론적 교훈** (RTKLIB `detslp_dop()`의 `#if 0`, (a)) — 겉보기 함수 존재만으로 "이 기능이 동작한다"고 단정하면 안 되고, 실제로 컴파일 경로에 포함되는지까지 확인해야 한다. 이번 조사에서 실제로 적용한 원칙이며, 향후 다른 스터디에서도 동일하게 적용할 것.

---

## 7. 열린 질문 (확인 안 된 것들, 정직하게 남김)

- (d) `rtklibexplorer/RTKLIB` 포크(`demo5`/`dev` 브랜치)가 원저자 저장소 대비 구체적으로 어떤 알고리즘 개선을 담고 있는지는 — 브랜치가 존재하고 활발히 갱신된다는 사실만 `git ls-remote`로 확인했을 뿐, 실제 diff나 소스코드는 이번에 열어보지 않았다.
- (d) RTKLIB `ppp.c`가 구현하는 각 조석/안테나/요잉 모델이 헤더 주석에 인용된 논문(IERS Conventions, Kouba 2009 등)의 수식과 정확히 일치하는지는 함수 본문을 수식 단위로 대조하지 않았으므로 확인되지 않았다.
- (d) GNSS-SDR `observables/` 모듈(`hybrid_observables_gs.cc`) 내부에서 추적 루프 출력을 의사거리/반송파위상으로 변환하는 구체적 알고리즘(예: 코드 위상→시간 변환, 다중 신호 융합 방식)은 디렉터리/파일 존재만 확인했고 내부 로직은 읽지 않았다.
- (d) GNSS-SDR과 RTKLIB의 실시간 처리 성능/지연(latency) 특성 차이는 코드 구조상 추정만 가능할 뿐 실측하지 않았다.
- (d) RTKLIB의 SBAS 처리(`sbas.c`)와 멀티패스 완화(multipath mitigation) 관련 로직은 이번 조사에서 파일 존재만 확인했고(2.1절 표), 내부 함수를 직접 읽어 알고리즘을 확인하지는 못했다 — 원래 과제에서 요청한 "멀티패스 완화... 실제로 찾을 수 있는 로직"에 대해 결과적으로 이번 조사 범위 내에서는 확정적 근거를 확보하지 못했다는 점을 명시한다.
- (d) GNSS-SDR이 이식한 RTKLIB 코드(`src/algorithms/libs/rtklib/`)가 원본의 어느 버전/커밋 기준으로 포크됐는지, 이후 RTKLIB 본가의 버그 수정이 GNSS-SDR 쪽에 반영되고 있는지는 확인하지 않았다.
