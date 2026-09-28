# PNG vs JPEG 품질 95 프레임 저장 성능 및 Halpe-26 결과 차이 검증

<div align="center">

![Python](https://img.shields.io/badge/Python-3.10%20%7C%203.12-blue?logo=python)
![OpenCV](https://img.shields.io/badge/OpenCV-4.x-green?logo=opencv)
![ONNXRuntime](https://img.shields.io/badge/ONNX%20Runtime-CPU-orange?logo=onnx)
![RTMPose](https://img.shields.io/badge/Pose-RTMPose--M%20(Halpe26)-purple)
![RTMDet](https://img.shields.io/badge/Detector-RTMDet--nano-red)
**러닝 영상 포즈 추정 파이프라인에서 PNG와 JPEG 품질 95의 저장시간·용량·관절 좌표 차이를 예비 표본과 전체 프레임으로 비교한 실험입니다.**

</div>

---

## 1. 요약

| 항목 | 관측 결과 |
|---|---:|
| 프레임 저장시간 | PNG 17.69초 → JPEG 품질 95 3.07초, 82.6% 단축 |
| 저장용량 | PNG 902MB → JPEG 품질 95 186MB, 79.4% 감소 |
| 공통 유효 프레임 | 111개 |
| 26개 관절 평균 좌표 차이 | 1.9855px |
| 3px 이내 좌표 비율 | 94.42% |
| 확인된 예외 | 화면 진입·퇴장 구간에서 최대 275.49px 차이 관측 |

**판단:** 이 한 개 시험 영상에서는 JPEG 품질 95가 저장 비용을 크게 줄였고, 주 러닝 구간의 좌표 차이는 대체로 작았습니다. 다만 이 결과는 정답 좌표와 비교한 정확도 평가가 아니며, 다른 영상 조건에서도 같은 품질을 보장하지 않습니다. MVP 적용 후보로 사용할 수 있지만 다양한 해상도·조명·모션 블러 영상의 추가 검증이 필요합니다.

## 2. 실험 배경과 질문

> Halpe-26 포즈 추정 모델을 구동할 때 PNG를 JPEG 품질 95로 바꾸면 저장시간과 용량이 얼마나 줄고, 동일 모델의 관절 좌표 결과가 얼마나 달라지는지 확인하기 위해 시작했습니다.

- **배경**: 러닝 자세 분석 시스템에서 비디오의 각 프레임을 디스크에 추출·저장하는 과정(OpenCV 기반)은 파이프라인 전체 지연 시간(Latency)의 상당 부분을 차지합니다.
- **핵심 질문 2가지**:
  1. ⚡ **속도 및 용량**: PNG 대신 JPEG 품질 95로 압축 저장하면 프레임 추출 속도와 용량이 얼마나 개선되는가?
  2. **결과 차이**: 손실 압축으로 인해 **Halpe-26 26개 관절 좌표**와 계산 피처가 어느 정도 달라지는가?

본 프로젝트는 **[Phase 1] 29장 균등 샘플링 예비 검증**으로 필터 및 기본 오차를 먼저 검토한 후, 그 확신을 바탕으로 **[Phase 2] 255장 전체 전수 검증**으로 확장하여 수행되었습니다.

---

## 3. 핵심 측정 결과

<div align="center">

| 평가 항목 | PNG (기존) | JPEG 품질 95 (개선) | 개선 및 영향 효과 |
|---|:---:|:---:|:---:|
| **프레임 추출/저장 시간** | **17.69 초** | **3.07 초** | **⚡ 82.6% 단축 (5.76배 가속)** |
| **디스크 저장 용량** | **902 MB** | **186 MB** | **💾 79.4% 디스크 절감** |
| **26개 관절 평균 좌표 차이** | 기준점 (0 px) | **1.986 px** | 단일 시험 영상의 공통 유효 프레임 기준 |
| **3.0 px 이내 좌표 비율** | 100 % | **94.4 %** | 대부분의 좌표 차이는 3px 이내 |
| **계산 피처 차이** | 기준값 | **시험에서 계산한 각도 차이 0.1° 미만** | 제한된 표본에서 관측한 값 |
| **MVP 적용 판단** | - | **조건부 적용 후보** | 경계 구간 필터 유지와 추가 영상 검증 필요 |

</div>

> **핵심 결론:**
> JPEG 품질 95는 이 시험에서 프레임 저장시간과 용량을 크게 줄였습니다. 좌표 차이는 주 러닝 구간에서 대체로 작았지만 화면 진입·퇴장 구간에는 큰 이상치가 있었습니다. 따라서 경계 구간을 제외하는 조건과 함께 MVP 적용 후보로 판단했으며, 일반화를 위해서는 더 다양한 입력 검증이 필요합니다.

---

## 4. [Phase 1] 29장 균등 추출 예비 검증

> 상세 자료 및 원시 데이터: [📂 `preliminary_29frames/`](preliminary_29frames/)

255장 전체를 돌리기 전, 알고리즘과 필터의 동작 여부를 먼저 빠르게 확인하기 위해 **전체 255장 중 9프레임 간격으로 29장(`1, 10, 19, ..., 253`번)을 균등 표본 추출**하여 예비 검증을 진행했습니다.

### ① 29장 프레임 필터링 추적 컨택트 시트
PNG와 JPEG에서 각각 29장을 동일하게 디텍터/필터에 통과시켰을 때의 프레임 보존 여부를 전수 시각화한 결과입니다.

![29장 프레임 필터 컨택트 시트](preliminary_29frames/results_det_score_0.3/frame_filter_contact_sheet.png)

### ② 18장 제외 원인 규명
- 초기에 29장 중 11장만 JSON에 저장되어 '매칭 누락'이 의심되었으나, 코드 추적 결과 **두 형식 모두 완벽히 동일한 18장을 제외하고 동일한 11장을 저장**했음을 확인:
  - **사람 미검출 (11장)**: 러너 진입 전 및 화면 퇴장 후 구간 (`1, 10, 19, 37, 46, 55, 64, 73, 91, 100, 253번`)
  - **가장자리 경계 제외 (7장)**: 러너가 화면 경계선에 걸린 프레임 (`28, 82, 109, 118, 127, 136, 145번`)
  - **유효 저장 (11장)**: 주 러닝 구간 (`154, 163, 172, 181, 190, 199, 208, 217, 226, 235, 244번`)

### ③ 235번 프레임 이상치(Outlier) 및 유리창 반사상 정밀 분석
235번 프레임에서 왼쪽 귀(LEar) 키포인트가 165.1px 차이가 난 원인을 과학적으로 역추적했습니다.

![235번 이상치 정밀 분석](preliminary_29frames/results_det_score_0.3/outlier_overlay.png)

![트래킹 컨텍스트 분석](preliminary_29frames/results_det_score_0.3/tracking_context.png)

- **원인**: 러너가 화면 오른쪽 유리창 문을 지나갈 때 **유리창 반사상**이 발생하여 모델의 키포인트 외삽이 튄 현상.
- **처리 결과**: 이 예비 시험에서는 `outside_range` 필터(화면 10px 이내 경계 조건)가 해당 퇴장 이상치를 제외했습니다. 다른 촬영 조건에서도 같은 결과가 나오는지는 추가 검증이 필요합니다.

> **Phase 1 결론:**
> 29장 예비 검증에서 **JPEG 품질 95 전환 시 82.6% 시간 단축(17.7초 → 3.07초)**과 평균 좌표 차이 1.84px를 관측했습니다. 표본이 작기 때문에 결론을 확정하지 않고 Phase 2의 255장 전체 프레임 비교로 확장했습니다.

---

## 5. [Phase 2] 255장 전체 프레임 검증

예비 검증의 확신을 바탕으로 `test7` 영상의 **1번부터 255번까지 전 구간을 단 한 프레임도 빠짐없이 전수 추론·비교**했습니다.

### ① 255장 전체 26개 관절 전수 산점도 (All 26 Keypoints Scatter Plot)
전체 유효 구간(111개 프레임 × 26개 관절 = **총 2,886개 키포인트**)의 픽셀 좌표 거리 오차를 전수 타정한 산점도입니다.

![전체 26개 관절 산점도](assets/images/keypoint_error_plot.png)

> **시각화 해석:**
> 러너가 온전히 보이는 **주 러닝 구간(148~230번)**에서는 좌표 차이가 작은 점들이 밀집합니다. 반면 화면 진입(138~144번)과 퇴장(235~242번)에서는 큰 이상치가 관측되므로 평균값만으로 품질을 판단하면 안 됩니다.

---

### ② 전체 프레임 타임라인 오차 추이 (Timeline Trend)
프레임 번호에 따른 평균 관절 오차(파란 실선)와 최대 관절 오차(주황 점선)의 변화 추이입니다.

![전체 타임라인 오차 추이](assets/images/full255_timeline_error_trend.png)

---

### ③ 픽셀 오차 최대 발생 Top 5 프레임 가로 비교 스트립 (Top 5 Outlier Strip)
255장 중 PNG vs JPEG 간 좌표 차이가 가장 컸던 상위 5개 프레임의 러너 크롭 및 포즈 비교입니다.

![Top 5 가로 비교 스트립](assets/images/top5_worst_frames_comparison_strip.png)

---

## 6. 세부 분석 리포트

세부적인 통계 및 부위별 분석 내용입니다.

<details>
<summary><b>🦴 [세부 1] Halpe26 26개 관절별 오차 분포 분석</b></summary>

<br>

![26개 관절별 오차 분포](assets/images/full255_keypoint_error_distribution.png)

### 신체 부위별 상세 분석:
1. **몸통 및 골반 (Kpt 11, 12, 18, 19)**:
   - **평균 오차 1.0px 미만**. 러닝 자세 분석의 기준축이 되는 척추와 골반은 JPEG 압축 노이즈에 매우 강건(Robust)합니다.
2. **상체 및 팔 (Kpt 05, 06, 07, 08, 09, 10)**:
   - **평균 오차 0.8px ~ 1.2px**. 팔 흔들림 궤적 추적에 오차가 거의 없습니다.
3. **무릎 및 다리 (Kpt 13, 14, 15, 16)**:
   - 무릎 굴곡 각도(Flexion)의 핵심인 무릎과 발목 관절 역시 **평균 1.1px 수준**으로 서브픽셀 정밀도를 유지합니다.
4. **발끝 및 뒤꿈치 (Kpt 20 ~ 25)**:
   - 빠른 스텝으로 인한 모션 블러로 평균 1.5~2.2px 수준의 오차가 관측되었으나, 실제 신발 크기(약 100px 이상) 대비 2% 미만으로 매우 미미합니다.

</details>

<details>
<summary><b>🛡️ [세부 2] Top 5 이상치 대시보드 및 시스템 안전성</b></summary>

<br>

![Top 5 정밀 분석 대시보드](assets/images/top5_worst_discrepancy_frames.png)

| 순위 | 프레임 번호 | 최대 오차 (px) | 평균 오차 (px) | 최대 오차 관절 | 발생 원인 |
|:---:|:---:|---:|---:|:---:|---|
| **1위** | **`00000144`** | **275.49 px** | 15.49 px | **왼엄지발(LBigToe)** | 화면 진입 초기 발끝이 프레임 가장자리에 걸림 |
| **2위** | **`00000140`** | **252.81 px** | 12.07 px | **왼발목(LAnkle)** | 러너 진입 시 신체 부분 폐색(Occlusion) |
| **3위** | **`00000138`** | **232.71 px** | 27.40 px | **왼새끼발(LSmallToe)** | 진입 시 발끝 키포인트 외삽(Extrapolation) 차이 |
| **4위** | **`00000237`** | **211.47 px** | 18.80 px | **오른새끼발(RSmallToe)** | 화면 우측 유리창 문 통과 시 반사광 혼선 |
| **5위** | **`00000242`** | **158.06 px** | 7.79 px | **왼새끼발(LSmallToe)** | 퇴장 직전 유리문 반사상 및 신체 이탈 |

> **경계 구간 처리:**
> 위 Top 5 오차는 화면 진입(138~144번) 및 퇴장(237~242번) 구간에서 발생했습니다. 해당 실험의 `outside_range` 필터는 이 구간을 제외했지만, 다른 촬영 조건에서도 동일하게 동작하는지는 추가 검증이 필요합니다.

</details>

<details>
<summary><b>🏃 [세부 3] 생체역학 러닝 피처별 영향 평가</b></summary>

<br>

| 분석 피처 | 관련 관절 | 관측 오차 | 피처에 미치는 영향 | 결론 |
|---|---|:---:|---|:---:|
| **무릎 굴곡 각도<br>(Knee Flexion)** | 골반 - 무릎 - 발목 | 1.0 ~ 1.1 px | 이 시험에서 계산한 각도 차이가 0.1° 미만 | **관측 차이 작음** |
| **몸통 전경 각도<br>(Trunk Lean)** | 엉덩이 중심 - 목 | 0.8 ~ 1.0 px | 이 시험에서 척추 기준축의 각도 차이가 0.05° 미만 | **관측 차이 작음** |
| **케이던스 (Cadence)** | 발목/발끝 수직 궤적 | 1.5 ~ 2.2 px | 피크 프레임 시점이 유지되는지 확인 | **시험 영상에서 변화 미관측** |
| **지면 접촉 시간<br>(Ground Contact Time)** | 발뒤꿈치/엄지발가락 | 1.5 ~ 2.2 px | 착지·이륙 추정 프레임이 유지되는지 확인 | **시험 영상에서 변화 미관측** |

</details>

---

## 7. 결론과 적용 조건

1. **MVP 적용 후보**
   - JPEG 품질 95는 이 시험에서 PNG보다 저장시간과 용량을 줄이면서 공통 유효 프레임의 평균 좌표 차이 1.9855px를 보였습니다.
2. **적용 조건**
   - 이 저장소에서 검증한 값은 품질 95이므로 다른 품질 값에 그대로 일반화하지 않습니다.
   - 화면 진입·퇴장 구간의 큰 이상치를 제외하는 경계 필터를 유지하고, 제외 여부를 로그로 확인해야 합니다.
3. **추가 검증**
   - 서로 다른 해상도, 조명, 의상, 배경, 모션 블러와 촬영 거리의 영상을 반복 측정해야 합니다.
   - 정답 좌표가 없으므로 이 실험만으로 모델 정확도가 유지됐다고 표현하지 않습니다.

---

## 8. 실행 및 재현 방법

```bash
# 1. 저장소 클론
git clone https://github.com/RunnersFeed-Project-Oracle-boot-camp/PNG-vs-JPEG-Rendering-Performance-and-FPS-Comparison.git
cd PNG-vs-JPEG-Rendering-Performance-and-FPS-Comparison

# 2. 필수 라이브러리 설치
pip install opencv-python numpy matplotlib rtmlib onnxruntime tqdm

# 3. 전체 255장 전수 검증 및 종합 차트/보고서 일괄 생성
python scripts/run_test7_png_jpeg_full_255_validation.py

# 4. Top 5 최대 오차 프레임 플롯만 단독 생성
python scripts/plot_test7_top5_worst_error_frames.py

# 5. 관절 전수 산점도(keypoint_error_plot) 단독 생성
python scripts/plot_test7_keypoint_error_scatter.py

# 6. [Phase 1] 29장 예비 검증 감사 재현 실행
python preliminary_29frames/audit_preliminary_29frames.py --source /path/to/poc --out /path/to/out
```

---

## 9. 저장소 디렉토리 구조

```
├── README.md                                      # 본 종합 검증 보고서
├── .gitignore                                     # Git 무시 규칙
├── preliminary_29frames/                          # [Phase 1] 29장 균등 추출 예비 검증 실험
│   ├── README.md                                  # 예비 검증 실험 상세 기록 문서
│   ├── audit_preliminary_29frames.py              # 예비 감사 실행 스크립트
│   ├── results_det_score_0.3/                     # 임계값 0.3 표본 결과, 컨택트 시트, 이상치 분석
│   └── results_det_score_0.4/                     # 임계값 0.4 표본 결과
├── assets/
│   └── images/                                    # 고해상도 시각화 차트 및 플롯
│       ├── keypoint_error_plot.png                # [산점도] 2,886개 관절 전수 분포
│       ├── full255_timeline_error_trend.png       # [타임라인] 255 프레임 오차 추이
│       ├── full255_keypoint_error_distribution.png# [분포] 26개 관절별 오차 막대그래프
│       ├── top5_worst_discrepancy_frames.png      # [대시보드] Top 5 상세 분석 5x2 플롯
│       ├── top5_worst_frames_comparison_strip.png # [스트립] Top 5 가로 비교 카드 뷰
│       ├── frame_00000154_diff.png                # [표본] 정상 러닝 프레임 비교
│       └── frame_00000235_diff.png                # [표본] 반사상 이상치 프레임 비교
├── data/
│   ├── full255_frame_metrics.csv                  # 111개 유효 프레임별 상세 수치 표
│   └── full255_validation_summary.json            # 전체 통계 요약 데이터 JSON
└── scripts/
    ├── run_test7_png_jpeg_full_255_validation.py  # 255장 전수 검증 및 차트 생성 메인 스크립트
    ├── plot_test7_top5_worst_error_frames.py      # Top 5 플롯 단독 생성기
    ├── plot_test7_keypoint_error_scatter.py       # 관절 전수 산점도 생성기
    └── render_test7_frame_pose_diff.py            # 개별 프레임 오버레이 렌더러
```
