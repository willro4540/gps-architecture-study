# 04. HuggingFace GPS/지리공간 AI 서베이 + QGIS GPS 통합 크로스레퍼런스

> **요약**: 이 문서는 두 파트로 구성됩니다. **Part A**는 HuggingFace Hub(models/datasets)에서
> GPS 궤적 예측·지오로케이션(이미지→위치)·위성영상 파운데이션 모델·GNSS 센서 퓨전, 이 네
> 카테고리를 실제로 검색해서 "진짜 존재하고 실제로 쓸 수 있는 것"만 골라낸 서베이입니다.
> 결론부터 말하면 **네 카테고리의 성숙도는 완전히 다릅니다** — 위성영상 파운데이션 모델은
> NASA/IBM/AllenAI 같은 실제 기관이 논문·코드·수 GB 가중치까지 공개한 **진짜 생태계**인 반면,
> "GPS 궤적 예측"이나 "GNSS 센서 퓨전" 전용 모델은 HuggingFace 검색 결과가 **사실상 0건**이라
> 지금은 존재하지 않는다고 보는 게 맞습니다. **Part B**는 이미 완료된 자매 저장소
> [`qgis-architecture-study`](../../qgis-architecture-study)가 정리한 QGIS 플러그인
> 아키텍처(`classFactory` 관례)를 그대로 인용하고, 그 위에 "실제 GPS/GNSS 데이터를 QGIS로
> 흘려넣는 오픈소스 프로젝트가 진짜 있는지"를 GitHub에서 직접 코드까지 열어 검증합니다.

## 근거 등급 표기 원칙

이 문서 전체에서 다음 4단계를 사용합니다([[reference_research_verification_skill]] 표준):

| 등급 | 의미 |
|---|---|
| (a) | 실제 모델카드/코드/공식문서로 직접 확인 |
| (b) | 강한 추론(모델카드+코드 조합) |
| (c) | 정황적(3자 출처, 블로그 등)이거나 직접 미확인 |
| (d) | 열린 질문 — 확인 안 됨 |

"직접 확인"은 이번 조사에서 실제로 해당 HuggingFace 모델카드 페이지 또는 GitHub 저장소 파일을
WebFetch로 열어서 대조한 경우에만 씁니다. 검색 결과 요약만 보고 등급 (a)를 준 항목은 없습니다.

---

## Part A — HuggingFace GPS/지리공간 AI 서베이

### A-1. GPS 궤적(trajectory) 예측 / 이상탐지 모델

**결론 먼저: 이 카테고리는 HuggingFace 위에서 사실상 비어 있습니다.**

- `huggingface.co/models?search=gps%20trajectory`를 직접 열어 확인한 결과, **검색 결과 0건**입니다
  (2026-09-05 확인, 등급 a — 직접 URL을 열어 페이지 자체를 대조함).
- `huggingface.co/datasets?search=gps%20trajectory`도 마찬가지로 **0건**입니다(등급 a).
- 반면 학술 논문(arXiv) 수준에서는 GPS 궤적 이상탐지 연구가 활발합니다 — 예: GADformer(그룹
  단위 궤적 이상탐지 트랜스포머, arXiv 2303.09841), "Beyond Regularity" 등 2508~2509년대
  논문들이 next-location prediction을 다룹니다. 하지만 **이 논문들의 코드/가중치가
  HuggingFace에 실제로 배포되어 있는 경우는 발견하지 못했습니다**(등급 d — 논문은 실재하나
  HF 배포 여부는 이번 조사에서 못 찾음. "없다"를 증명하는 것은 어렵기 때문에 열린 질문으로
  남깁니다).
- **주의할 만한 "가짜 친구"(false friend)**: HuggingFace `transformers` 라이브러리에는
  `Trajectory Transformer`([공식 문서](https://huggingface.co/docs/transformers/model_doc/trajectory_transformer))
  라는 모델이 실제로 있습니다(등급 a). 이름만 보면 GPS 궤적 예측처럼 보이지만, 실제로는
  "오프라인 강화학습을 하나의 시퀀스 모델링 문제로 취급"하는 모델로, 여기서 "trajectory"는
  **로봇/에이전트의 state-action-reward 시퀀스**를 뜻하지 GPS 좌표가 아닙니다. GPS/지리공간
  AI를 검색하다가 이런 이름 충돌에 낚이지 않도록 기록해 둡니다.
- 인접 분야로 **자율주행 차량 경로 예측**은 실제로 성숙한 생태계입니다 — 예: `nvidia/Alpamayo-R1-10B`
  (VLA 모델, 월 11,127 다운로드·432 좋아요, 논문 arXiv 2511.00088, 코드는 Apache-2.0/가중치는
  OpenMDW-1.1, 등급 a — 모델카드 직접 확인). 다만 이 모델이 예측하는 것은 **차량 좌표계
  기준 6.4초 후 64개 웨이포인트(x,y,z + 회전행렬)**이지 위경도(GPS lat/lon)가 아닙니다. 즉
  "차량 미래 경로 예측"은 실재하지만 "GPS 좌표 자체를 다루는 궤적 모델"과는 다른 문제입니다.

**정리**: GPS 위경도 시퀀스를 입력·출력으로 다루는 다음위치예측/이상탐지 모델은 이번 조사
기준 HuggingFace에 **다운로드 가능한 형태로는 없다고 보는 게 정직한 결론**입니다(등급 d,
지금 시점 기준). 자율주행 경로예측(좌표계가 다름)과 학술 논문 코드(HF 미배포)를 GPS 궤적
모델로 착각하지 않는 것이 이 카테고리에서 가장 중요한 함정 회피입니다.

### A-2. 지오로케이션 — 이미지 → 위치 추정

이 카테고리는 **진짜로 존재하고, 신뢰도 스펙트럼이 넓게 퍼져 있습니다.** 발견한 것을 신뢰도
순으로 정리합니다.

| 모델 | 조직 | 실제 근거 | 등급 |
|---|---|---|---|
| [`geolocal/StreetCLIP`](https://huggingface.co/geolocal/StreetCLIP) | Lukas Haas 등(스탠퍼드 연구자) | 논문 [arXiv 2302.00275](https://arxiv.org/abs/2302.00275), 110 likes, 월 8,381 다운로드, 실제 데모 스페이스 28개가 이 모델을 사용(예: `Plonk`, `Location_Predictor`), CC-BY-NC-4.0 | **(a)** — 모델카드 직접 확인 |
| [`sdan/geospot-base`](https://huggingface.co/sdan/geospot-base) | 개인(sdan) | SigLIP2-so400m 백본 + GeoCLIP식 대조학습, safetensors 실제 가중치 존재, 학습 스텝(20.6만)까지 명시 | **(b)** — 아키텍처 주장은 코드/카드로 뒷받침되나 월 15다운로드·2좋아요·논문 없음이라 "실사용 정확도"는 정황적(c) |
| [`ghost233lism/GeoAgent`](https://huggingface.co/ghost233lism/GeoAgent) | 난카이대학·서북공업대학 연구팀(Modi Jin 등) | Qwen2.5-VL-7B 기반, 논문 [arXiv 2602.12617](https://arxiv.org/abs/2602.12617), 자체 데이터셋(GeoSeek, 3.3만장) 공개, 데모 스페이스 존재, SFT+GRPO 학습 파이프라인 전체 공개 | **(a)** — 모델카드 직접 확인, 실제 연구팀 배경 확인 |
| [`ArjunMehan/pretrained-geolocation-clip-model`](https://huggingface.co/ArjunMehan/pretrained-geolocation-clip-model) | 개인 | **모델카드 없음("No model card")**, 월 14다운로드·0좋아요, PIGEON/PIGEOTTO 재구현이라는 GitHub 설명과 실제 HF 저장소 내용이 대조 불가 | **주의 예시** — 이게 정확히 이 도메인의 "겉보기엔 그럴듯하나 실체 없음" 사례입니다. 페이지는 실재하지만(등급 a) 내용물(모델 설명·성능)은 (d) |

**하이프 대 실체의 핵심 사례 — PIGEON/PIGEOTTO**: CVPR 2024에 게재된 유명 논문([arXiv 2307.05845](https://arxiv.org/abs/2307.05845), 대중 매체에도 "AI가 지오게서를 이긴다"는 식으로 소개됨)이지만,
**공식 모델 가중치는 프라이버시 이유로 공개되지 않았고 코드만 공개**되어 있습니다(등급 c —
3자 기사와 논문 서술 기반, 원 저자 배포 페이지에서 "weights not released"라는 명시적 문구까지는
이번 조사에서 원문 대조하지 못함). 위 표의 `ArjunMehan` 저장소처럼 "PIGEON 기반"을 표방하는
비공식 재구현들이 떠 있지만 실체 검증이 안 되는 경우가 많다는 것이 이 도메인 전반의 패턴입니다
— **유명세와 실제 재사용 가능성은 별개**라는 걸 보여주는 좋은 사례입니다.

### A-3. 위성영상 파운데이션 모델 / 데이터셋 — **이번 서베이에서 가장 실체 있는 카테고리**

지형 정밀도·GIS 방향에 가장 직결되는 카테고리이고, 결과도 가장 풍부합니다.

| 모델 | 조직 | 실제 근거 | 등급 |
|---|---|---|---|
| [`ibm-nasa-geospatial/Prithvi-EO-1.0-100M`](https://huggingface.co/ibm-nasa-geospatial/Prithvi-EO-1.0-100M) | IBM + NASA | NASA HLS(Harmonized Landsat Sentinel-2) V2 L30 데이터로 사전학습된 시공간 ViT(MAE 방식), 논문 [arXiv 2310.18660](https://arxiv.org/abs/2310.18660), Apache-2.0, 월 1,049 다운로드·289 좋아요, 데모 스페이스(`Prithvi-EO-1.0-demo`) 및 GitHub([`NASA-IMPACT/hls-foundation-os`](https://github.com/NASA-IMPACT/hls-foundation-os)) 모두 확인 | **(a)** |
| `ibm-nasa-geospatial/Prithvi-EO-2.0-*`(300M/600M/tiny 등 다수 체크포인트) | IBM + NASA | 같은 조직, 실제 GB급 가중치 파일(`.pt`, 최대 2.65GB) 존재를 파일 목록에서 직접 확인, BurnScars 등 파인튜닝 버전도 별도 공개 | **(a)** — 파일 트리 직접 확인 |
| [`allenai/OlmoEarth-v1_2-Base`](https://huggingface.co/allenai/OlmoEarth-v1_2-Base) | Allen AI(Ai2) | Sentinel-1/Sentinel-2/Landsat 통합 처리 ViT-Base(114M), 논문 arXiv 2605.20804, 월 6,606 다운로드(Prithvi보다 다운로드는 많지만 좋아요는 7개뿐 — 아직 커뮤니티 반응은 이른 단계), 자체 플랫폼(`olmoearth.allenai.org`)에 실제 서비스 배포 | **(a)** — 단, 라이선스가 "OlmoEarth Artifact License"라는 **커스텀 라이선스**라 재사용 조건(상업적 이용 가능 여부 등)은 원문을 별도로 읽지 않아 **(d) 확인 안 됨**으로 남김 |
| [`tacofoundation/SEN2SR`](https://huggingface.co/tacofoundation/SEN2SR) | tacofoundation(오픈 커뮤니티 프로젝트로 추정) | 검색 결과에만 등장, 모델카드 직접 열람은 못함 | **(c)** — 존재는 실재하나 상세 미확인 |
| `MBZUAI/TerraFM` | MBZUAI(무함마드 빈 자이드 인공지능대) | 검색 결과에만 등장, ViT 백본·Sentinel-1/2 18.7M 타일 학습이라는 설명은 검색 요약 기반 | **(c)** — 상세 미확인 |

**판단**: 이 카테고리는 "마케팅만 있고 실체 없는" 패턴이 아니라, **실제 정부/연구기관(NASA,
IBM)과 대형 AI 연구소(Allen AI)가 논문+코드+수백MB~수GB 가중치를 공개 배포하는 진짜
생태계**입니다. Prithvi 계열은 이미 산불 흔적(BurnScars)·홍수 매핑 등 실사용 파인튜닝
사례까지 갖춘 성숙 단계이고, OlmoEarth는 더 최근(2026년) 진입자로 다운로드는 많지만
커뮤니티 신호(좋아요)는 아직 적은, "이제 막 주목받기 시작한" 단계로 보입니다.

### A-4. 시계열/센서 퓨전 모델 중 GNSS 데이터를 다루는 것

- HuggingFace에 **"GNSS 전용 파운데이션 모델"이라 부를 만한 것은 발견하지 못했습니다**(등급 d).
- 다만 **실제 RTK급 GNSS 원시 데이터가 포함된 데이터셋**은 실재합니다:
  [`madhurbehl/RACECAR_DATA`](https://huggingface.co/datasets/madhurbehl/RACECAR_DATA) —
  IROS 2023 논문(arXiv 2306.03252) 기반, 6개 대학 자율주행 레이싱팀(Indy Autonomous
  Challenge 2021-22)의 실제 센서 데이터로, Novatel PwrPak RTK GNSS 수신기 2대 + IMU 데이터가
  포함됩니다(20Hz "best available GNSS solution" 토픽 등). HF에 올라간 파일 자체는 26.2KB
  메타데이터뿐이고 실제 센서 데이터는 AWS S3에 별도 호스팅되는 구조입니다. CC-BY-NC-4.0,
  월 184 다운로드. **등급 (a)** — 데이터셋 카드 직접 확인, 실존 확인.
- HF의 범용 시계열 데이터셋 허브(`huggingface.co/datasets?modality=timeseries`)에는
  electricity/traffic/weather 등은 있지만 GNSS/GPS를 표방하는 항목은 검색에서 드러나지
  않았습니다(등급 c — 전체 목록을 다 열람하지는 못함).

**정리**: "GNSS 데이터를 다루는 시계열/센서퓨전 모델"이라는 질문에는, **모델 차원에서는
사실상 없고, 데이터셋 차원에서는 자율주행/로보틱스 데이터셋에 곁다리로 포함된 형태로만
존재**한다는 게 정직한 답입니다.

### Part A 종합 판단 — 생태계가 실제로 풍부한가, 얇은가

**정직하게 말하면 얇습니다. 단, 카테고리별로 완전히 다릅니다.**

- **위성영상 파운데이션 모델(A-3)**: 진짜로 두껍습니다. NASA/IBM/Allen AI/MBZUAI 등 실명
  기관이 참여하고, 논문·가중치·데모가 모두 갖춰진 성숙한 서브생태계입니다.
- **지오로케이션(A-2)**: 중간입니다. StreetCLIP·GeoAgent처럼 논문과 실제 사용량을 갖춘
  진짜 모델이 소수 존재하지만, 그 주변에 모델카드도 없는 방치된 저장소(`ArjunMehan` 사례)가
  섞여 있어 옥석 가리기가 필요합니다.
- **GPS 궤적 예측(A-1)과 GNSS 센서퓨전(A-4)**: 사실상 비어 있습니다. HuggingFace 검색이
  직접 0건을 반환했고, 관련 연구는 arXiv 논문 수준에 머물러 있거나 자율주행이라는 인접
  분야에 흡수되어 있습니다.

**이 시리즈의 다른 카테고리(NVIDIA ACE, RealityScan 등)와 비교하면**: 위성영상 쪽은 그
분야들과 마찬가지로 "실제 기관이 뒷받침하는 진짜 기술"이지만, GPS 궤적/GNSS 쪽은 이
아키텍처 스터디 시리즈가 반복해서 경계해 온 "마케팅은 있으나 실체가 얇은" 패턴에 오히려
가깝습니다 — 다만 여기서는 마케팅조차 거의 없고 그냥 "아직 아무도 안 만든" 상태에 가깝다는
점이 다릅니다.

---

## Part B — GIS 통합 크로스레퍼런스

### B-1. 자매 저장소 인용 — QGIS 플러그인 아키텍처

`qgis-architecture-study/docs/04_plugin_system_and_pyqgis.md`가 이미 정리한 핵심 사실을
그대로 인용합니다(재조사하지 않음):

- QGIS 플러그인은 언어(Python/C++)와 무관하게 **`classFactory(iface)`라는 정해진 이름의
  함수를 정의**해야 하고, QGIS 본체가 이 이름을 찾아서 호출하는 "관례 기반 진입점" 방식을
  씁니다(SketchUp의 "확장이 스스로 등록을 호출"하는 방식과 대비).
- 단, 그 문서는 "서드파티 플러그인도 전부 같은 `classFactory` 규칙을 따르는지는 공식 문서
  수준에서만 확인했고, 로컬에는 번들 플러그인 4개만 있어 서드파티 사례를 직접 열어보지
  못했다"는 **열린 질문**을 명시적으로 남겼습니다.

### B-2. QGIS 코어 내장 기능 — Live GPS Tracking

**실시간 GPS 추적은 서드파티 플러그인이 아니라 QGIS 코어에 내장된 기능입니다**(등급 a —
공식 문서 [`docs.qgis.org` Live GPS tracking](https://docs.qgis.org/3.44/en/docs/user_manual/working_with_gps/live_GPS_tracking.html)
직접 확인).

- **GPS 툴바**가 연결·기록을 담당하고, **GPS 정보 패널**이 실시간 위치/위성 신호/정확도를
  표시합니다.
- 연결 방식은 다양합니다: Linux에서는 **GPSD**(미들웨어 서버), Windows에서는 **GPSGate**
  (프리웨어 미들웨어), 또는 시리얼/블루투스 COM 포트 직결. **NMEA** 원시 문장 로깅도
  지원합니다.
- 기록은 GeoPackage/SpatiaLite에 저장 가능 — `qgis-architecture-study/docs/05`가 확인한
  "QGIS 진영이 SQLite 계열을 표준 저장소로 반복 채택한다"는 결론과 정확히 같은 패턴이
  여기서도 재확인됩니다.

### B-3. 실제 오픈소스 GPS/GNSS↔QGIS 연동 프로젝트 3종 (GitHub에서 직접 확인)

| 프로젝트 | 조직/저자 | 실제 근거 | 등급 |
|---|---|---|---|
| [`gis-support/gps-tracker`](https://github.com/gis-support/gps-tracker) | GIS Support Sp. z o.o.(폴란드 GIS 회사) | 실제 Python 소스(`GPSTracker.py`, `gpsConnection.py`, `gpsUtils.py`) 38커밋, GPL-2.0. 별표 1·포크 2로 **소규모/니치** | **(a)** 존재·기능은 코드 직접 확인, **(c)** 활성도(유지보수 정도)는 낮아 보임 |
| [`SGroe/gpx-segment-importer`](https://github.com/SGroe/gpx-segment-importer) | 개인(SGroe), 공식 QGIS 플러그인 저장소([plugins.qgis.org](https://plugins.qgis.org/plugins/GpxSegmentImporter/))에 정식 등록 | GPX 트랙포인트를 선분으로 변환, 거리/속도/고도차 자동 계산, 85커밋, Apache-2.0. 공식 QGIS 플러그인 채널을 통해 배포된다는 점에서 신뢰도가 다른 두 개보다 조금 더 높음 | **(a)** |
| [`MIERUNE/GTFS-GO`](https://github.com/MIERUNE/GTFS-GO) | MIERUNE(일본 GIS 전문 기업, mierune.co.jp) | GTFS(대중교통) 데이터를 파싱해 경로/정류장을 GeoJSON으로 시각화 + 빈도 집계, **별 129·포크 12**, GPL-2.0, CI(유닛테스트+린트)까지 갖춘 **이 3종 중 가장 활발하고 성숙한 프로젝트** | **(a)** |

**GTFS 2 GIS**(`plugins.qgis.org/plugins/qgis_gtfs_plugin/`)라는 별도 플러그인도 "실시간 차량
추적, 대시보드" 기능을 표방하지만, 이번 조사에서 **GitHub 소스 저장소 링크를 찾지 못해
코드 자체는 대조하지 못했습니다** — 공식 플러그인 페이지의 기능 설명 자체는 실재하지만,
그 설명이 코드와 얼마나 일치하는지는 **등급 (c)**로 남깁니다(marketing page만 확인, 소스
미확인 — 이 하우스룰이 정확히 경계하는 패턴).

### B-4. 자매 저장소의 열린 질문을 직접 갱신 — 서드파티도 정말 `classFactory`를 쓰는가

`qgis-architecture-study/docs/04`가 "공식 문서 수준에서만" 확인했다고 남긴 질문에 답하기
위해, 위 표의 `gis-support/gps-tracker` 저장소의 `__init__.py`를 **GitHub 원본에서 직접
열람**했습니다(2026-09-05 확인):

```python
def classFactory(iface):
    from .GPSTracker import GPSTracker
    return GPSTracker(iface)
```

**확인된 사실(등급 a, 로컬/원본 파일 직접 대조)**: 이 서드파티 GPS 플러그인도 정확히
`classFactory(iface)`라는 같은 이름·같은 시그니처를 씁니다. 표본이 1건뿐이라 "모든 서드파티
플러그인이 예외 없이 이 규칙을 따른다"고 일반화하기엔 이르지만(등급은 여전히 b — 한 사례로
전체를 단정하지 않음), 적어도 **"QGIS 공식 문서가 요구하는 관례를 실제 커뮤니티 플러그인이
그대로 지킨다"는 구체적 사례가 하나 추가**되어, 자매 저장소의 열린 질문이 "확인 안 됨"에서
"1건 확인, 일반화는 보류"로 한 단계 진전했습니다. (`SGroe/gpx-segment-importer`의 동일 파일은
브랜치명이 달라 이번 조사에서 접근하지 못했습니다 — 등급 d로 남김.)

---

## 재사용 가능한 설계 패턴 / 향후 활용 아이디어

사용자의 특화 방향(위성사진+GIS+Google 3D로 지형 정밀도 다루기, 공공데이터/빅데이터로
예측·시각화)과 연결해서, **실제로 검증된 것만** 근거로 아이디어를 적습니다(하드웨어 최적화
판단은 배제, [[feedback_no_hardware_bias_in_future_expansion_analysis]] 원칙):

1. **Prithvi/OlmoEarth류 위성 파운데이션 모델은 "제로부터 학습" 없이 지형 특화를 시작할 수
   있는 실제 발판입니다.** 두 모델 모두 Sentinel-2/Landsat 원본 밴드를 그대로 입력받고,
   IBM 쪽은 산불 흔적·홍수 매핑 같은 **파인튜닝 예제와 코드까지 GitHub에 공개**되어 있어
   (`NASA-IMPACT/hls-foundation-os`), "위성사진으로 지형 변화를 탐지"하는 사용자의 방향과
   가장 직접적으로 맞닿습니다. 다만 OlmoEarth는 라이선스 원문을 먼저 확인해야 합니다(A-3의
   열린 질문).
2. **QGIS의 GPS 툴바(B-2)는 "실측 GPS 트랙 → GIS 좌표계로 즉시 편입"이라는, 사용자가 이미
   가진 QGIS 스터디와 이번 조사를 잇는 가장 짧은 다리입니다.** 위성 파운데이션 모델이 만든
   변화 탐지 결과와, 실제 필드에서 QGIS GPS 툴바로 수집한 실측 트랙을 같은 프로젝트 파일
   안에서 겹쳐보는 것이 기술적으로 이미 가능하다는 뜻입니다(둘 다 GeoPackage/SQLite로 만남).
3. **`classFactory` 관례(B-4)는 QGIS 생태계에 새 도구를 얹을 때 안전한 진입점 설계
   패턴으로 그대로 재사용 가능합니다.** 사용자가 구상 중인
   [[project_qgis_assistant_plugin_idea]](QGIS 어시스턴트 플러그인)를 실제 착수할 때, 이번에
   실측으로 확인된 `def classFactory(iface): return MyPlugin(iface)` 형태를 그대로 시작점으로
   쓸 수 있습니다 — 자매 저장소가 문서 수준 근거만 가졌던 것을, 이번 조사가 실제 커뮤니티
   코드로 한 단계 보강했습니다.
4. **GPS 궤적/GNSS 예측 모델의 공백(A-1, A-4)은 위협이 아니라 여백입니다.** HuggingFace에
   다운로드 가능한 형태의 GPS 궤적 예측 모델이 없다는 것은, 공공데이터(예: 대중교통
   GTFS-실시간, 국가 GNSS 관측망)를 직접 다뤄 예측·시각화를 만드는 작업이 아직 "레드오션이
   아니다"라는 뜻이기도 합니다. MIERUNE의 `GTFS-GO`(B-3)처럼 GTFS라는 표준 공공데이터
   포맷을 QGIS로 끌어들이는 실제 성숙한 선례가 있으니, 이를 "GPS/GNSS 원시 데이터"로
   확장하는 것은 참고할 실물 코드가 있는 상태에서 시작하는 셈입니다.
5. **지오로케이션 모델(A-2)의 "옥석가리기" 경험 자체가 재사용 가능한 체크리스트입니다.**
   이번에 실제로 확인한 구분법 — 논문 유무, 다운로드/좋아요 수, 데모 스페이스 존재,
   모델카드 유무 — 은 향후 다른 아키텍처 스터디에서 "새로 뜨는 모델이 진짜인지" 판단할 때
   그대로 재사용할 수 있는 실전 필터입니다.

## 확인 안 됨 / 열린 질문 정리

- OlmoEarth의 "OlmoEarth Artifact License" 정확한 재사용 조건(상업적 이용 가능 여부 등) —
  원문 미열람.
- `tacofoundation/SEN2SR`, `MBZUAI/TerraFM`의 상세 스펙 — 검색 결과 수준에서만 확인, 모델카드
  직접 열람 안 함.
- PIGEON/PIGEOTTO의 "가중치 비공개" 방침을 원 저자 공식 페이지에서 원문 문구로 대조하지
  못함(3자 출처 기반).
- GTFS 2 GIS(`qgis_gtfs_plugin`) 플러그인의 실제 소스코드 — GitHub 링크를 찾지 못해 기능
  설명을 코드로 대조하지 못함.
- `SGroe/gpx-segment-importer`의 `classFactory` 여부 — 브랜치명 문제로 이번 조사에서 파일
  직접 열람 실패, 다음 재검증 시 다시 시도 필요.
- "서드파티 QGIS 플러그인은 전부 `classFactory`를 따른다"는 일반화 — 이번에 1건 추가
  확인했으나 표본이 여전히 작아 등급은 (b)에 머무름.
