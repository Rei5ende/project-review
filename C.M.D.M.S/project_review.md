<div align="center">

# 🌊 C.M.D.M.S.

### Coastal Marine Debris Monitoring System

**AI(YOLOv5 + SAHI) 기반 고해상도 UAV 항공 이미지 해양 쓰레기 탐지 및 정량화 웹 서비스**

<br>

![Python](https://img.shields.io/badge/Python-3776AB?style=flat-square&logo=python&logoColor=white)
![PyTorch](https://img.shields.io/badge/PyTorch-EE4C2C?style=flat-square&logo=pytorch&logoColor=white)
![YOLOv5](https://img.shields.io/badge/YOLOv5-00FFFF?style=flat-square&logo=yolo&logoColor=black)
![SAHI](https://img.shields.io/badge/SAHI-Slicing_Inference-6E56CF?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)

</div>

---

## 📌 At a Glance

> - 🛰️ **기능** — UAV로 촬영한 고해상도 해안 항공 이미지를 업로드하면, 해양 쓰레기의 **종류·위치·수량**을 자동으로 탐지·정량화하는 웹 서비스
> - 🧠 **기술** — 실시간 탐지에 강한 **YOLOv5**에 **SAHI(Slicing Aided Hyper Inference)**를 결합하여, 고해상도 이미지 속 **미세 객체 탐지 성능**을 극대화
> - 📊 **데이터** — 실제 다양한 환경에서 수집된 **TACO(Trash Annotations in Context)** 데이터셋(COCO 포맷, 60개 계층 카테고리)으로 학습
> - 🌍 **효과** — 수거 인력·자원의 **효율적 배분**을 지원하고, 정량 데이터 기반의 의사결정으로 **해양 생태계 보호**와 **UN SDGs** 달성에 기여

---

## 1. 배경 및 목적 (Problem Statement & Goal)

### 🚨 문제 상황

제주도 연안은 매년 심각해지는 해양 쓰레기 문제에 직면해 있습니다.

> 제주도의 연간 해양 쓰레기 수거량은 **2만 톤을 돌파**했으며,
> 그중 **플라스틱이 약 70.1%**를 차지합니다.

이러한 쓰레기는 해양 생태계를 파괴하고 수산업·관광업에 직접적인 피해를 입히지만, 정작 이를 관측하고 대응하는 방식은 여전히 뒤처져 있습니다.

### ⚠️ 기존 방식의 한계

| 관측 방식 | 주요 한계점 |
| :--- | :--- |
| **수작업 현장 관측** | 넓은 해안선을 사람이 직접 순찰 — 막대한 **인력·시간 낭비**, 관측 주기가 길고 데이터의 객관성 부족 |
| **일반 드론 관측** | 육안 확인 또는 단순 촬영에 의존 — 미세 쓰레기에 대한 **탐지 정밀도 부족**, 정량화 불가 |

### 🎯 프로젝트 목적

C.M.D.M.S.는 **UAV 항공 촬영**과 **AI 객체 탐지 기술**을 결합하여 위 문제를 해결합니다.

- 사람의 눈을 대신해 **미세 쓰레기까지 자동 탐지**하고,
- 쓰레기의 종류와 양을 **정량 데이터로 산출**하며,
- 이를 **누구나 접근 가능한 웹 대시보드**로 제공하여

현장 관측의 한계를 넘어 **데이터 기반의 효율적인 해양 정화 의사결정**을 지원하는 것을 목표로 합니다.

---

## 2. AI 방법론 및 데이터셋 (AI Methodology & Dataset)

### 🧩 Base Model — 왜 YOLOv5인가

해양 정화 현장은 방대한 양의 이미지를 신속하게 처리해야 하는 환경입니다. YOLOv5는 이러한 요구에 부합하는 **속도와 정확도의 탁월한 균형**을 제공합니다.

- **One-stage Detector** 구조로 실시간에 가까운 추론 속도 확보
- 경량화된 아키텍처로 **드론·엣지 환경으로의 확장 가능성** 보유
- 활발한 커뮤니티와 검증된 학습 파이프라인으로 **빠른 프로토타이핑** 지원

### 🔬 Enhancement — SAHI를 결합한 이유

고해상도 UAV 이미지는 넓은 영역을 담는 대신, 개별 쓰레기 객체가 전체 프레임에서 차지하는 픽셀 비중이 극히 작습니다. 표준 추론에서는 이러한 **소형 객체(small object)**가 다운샘플링 과정에서 소실되어 탐지 성능이 급격히 저하됩니다.

> **SAHI(Slicing Aided Hyper Inference)**는 원본 고해상도 이미지를 **격자 형태의 작은 슬라이스로 분할**하여 각 조각에 대해 개별 추론을 수행한 뒤, 결과를 **원본 좌표계로 병합(merge)**하는 기법입니다.

이를 통해 얻는 이점은 다음과 같습니다.

- **미세 객체 탐지율 향상** — 각 슬라이스 내에서 객체의 상대적 크기가 커져 탐지 확률이 상승
- **정밀한 정량화** — 놓치던 소형 쓰레기까지 집계에 반영되어 수량 산출 신뢰도 개선
- **모델 재학습 불필요** — 추론 단계(inference-time)에 적용되는 기법으로, 기존 YOLOv5 가중치를 그대로 활용

### 📦 Dataset — TACO (Trash Annotations in Context)

학습에는 실제 쓰레기 이미지 데이터셋인 **TACO**를 사용했습니다.

| 특징 | 설명 |
| :--- | :--- |
| **다양한 환경** | 해변, 도로, 자연 배경 등 실세계의 다양한 조명·배경 조건에서 수집 |
| **표준 포맷** | 산업 표준인 **COCO 포맷**의 어노테이션으로 파이프라인 호환성 확보 |
| **계층적 분류** | **60개 카테고리**의 계층(hierarchy) 구조로, 세분화된 쓰레기 종류 분류 가능 |
| **Unlabeled Litter 처리** | 라벨링되지 않은 쓰레기 영역에 대한 처리 기능을 지원하여 학습 노이즈 관리 |

---

## 3. 서비스 워크플로우 (Web Service Action Flow)

사용자는 별도의 전문 지식 없이 **4단계**만으로 분석 결과를 얻을 수 있습니다.

```text
 ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
 │   [1] UPLOAD │     │ [2] INFERENCE│     │ [3] DETECTION│     │ [4] DASHBOARD│
 │              │     │              │     │              │     │              │
 │  이미지 업로드 │ ──▶ │   AI 추론 수행 │ ──▶ │  객체 탐지·집계 │ ──▶ │   결과 시각화  │
 │              │     │              │     │              │     │              │
 │  (Preview)   │     │  (Loading)   │     │ (YOLOv5+SAHI)│     │(Visualization)│
 └──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
```

| 단계 | 사용자 액션 | 시스템 동작 |
| :---: | :--- | :--- |
| **1. Upload** | UAV 촬영 이미지를 업로드하고 미리보기 확인 | 이미지 유효성 검사 및 **Preview** 렌더링 |
| **2. Inference** | 분석 시작 버튼 클릭 | **YOLOv5 + SAHI** 파이프라인 실행, **Loading** 상태 표시 |
| **3. Detection** | 대기 | 슬라이스별 추론 → 병합 → 쓰레기 **종류·수량 집계** |
| **4. Dashboard** | 분석 결과 열람 | 탐지 결과 이미지와 통계를 **대시보드로 시각화** |

---

## 4. 기대 효과 (Impact Assessment)

C.M.D.M.S.가 제공하는 정량 데이터는 단순한 탐지를 넘어 실질적인 가치를 창출합니다.

- **💰 효율적 자원 배분 (Resource Optimization)**
  쓰레기 밀집 구역을 데이터로 식별하여, 한정된 수거 인력과 장비를 **우선순위가 높은 지역에 집중 투입**할 수 있습니다.

- **⏱️ 수거 효율성 향상 (Operational Efficiency)**
  수작업 관측 대비 관측 주기를 단축하고, **정량 데이터 기반의 수거 계획 수립**으로 운영 비용을 절감합니다.

- **🐢 해양 생태계 보호 (Ecosystem Protection)**
  신속한 오염 현황 파악과 대응으로 쓰레기가 생태계에 미치는 피해를 **선제적으로 최소화**합니다.

- **♻️ 지속 가능한 발전 목표 기여 (Contribution to SDGs)**
  UN 지속가능발전목표 중 **SDG 14(해양 생태계 보전)** 및 **SDG 11(지속가능한 도시와 공동체)** 달성에 기여합니다.

---

## 5. 시작 가이드 (Getting Started)

> ⚙️ 아래는 표준 실행 절차의 뼈대입니다. 실제 스크립트 경로·인자에 맞게 수정하여 사용하세요.

### 📋 Prerequisites

- Python 3.8+
- CUDA 지원 GPU (권장)

### 🔧 Installation

```bash
# 1. 저장소 클론
git clone https://github.com/[your-username]/C.M.D.M.S.git
cd C.M.D.M.S.

# 2. 가상환경 생성 및 활성화 (선택)
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

# 3. 의존성 패키지 설치
pip install -r requirements.txt
```

### 🚀 Run Inference

```bash
# 이미지에 대한 쓰레기 탐지 추론 실행
python detect.py \
    --weights [모델 가중치 경로 ex: weights/best.pt] \
    --source  [입력 이미지 경로 ex: data/samples/] \
    --slice                          # SAHI 슬라이싱 추론 활성화
```

### 🌐 Run Web Service

```bash
# 웹 서버 실행
python app.py
# 브라우저에서 http://localhost:5000 접속
```

---

## 🛠️ Tech Stack

| 구분 | 기술 |
| :--- | :--- |
| **AI / ML** | `Python` · `PyTorch` · `YOLOv5` · `SAHI` |
| **Dataset** | `TACO (COCO Format)` |
| **Frontend** | `[프론트엔드 스택 입력 ex: React, HTML/CSS/JS]` |
| **Backend** | `[백엔드 스택 입력 ex: Flask, FastAPI]` |
| **Database** | `[DB 입력 ex: MySQL, SQLite]` |

> 🔺 **TODO:** 프론트엔드·백엔드·DB 항목을 실제 사용한 스택으로 교체하세요.

---

## 👥 Team

| 이름 | 역할 |
| :--- | :--- |
| **김동우** | Leader |
| 김동현 | Member |
| 김태욱 | Member |
| 김서현 | Member |
| 김명진 | Member |

---

<div align="center">

**🌊 깨끗한 바다를 위한 AI, C.M.D.M.S.**

</div>
