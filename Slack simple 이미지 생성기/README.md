# Slack AI 이미지 생성기: Replicate 및 LoRA 기반 이미지 생성 에이전트

## 🌟 프로젝트 개요

이 프로젝트는 **n8n 워크플로우**를 활용하여, 사용자가 Slack에서 입력한 간단한 설명을 AI Agent가 **구체적인 이미지 프롬프트와 LoRA 설정**으로 변환하고, Replicate API를 통해 이미지를 생성한 후 Slack에 업로드하는 **전 과정 자동화 시스템**입니다.  

Slack 메시지를 입력받아 → AI가 프롬프트를 생성 → Replicate 모델을 통해 이미지 생성 → 상태 확인(Polling) → Slack 업로드까지 이어지는 **엔드투엔드 이미지 생성 파이프라인**을 제공합니다.

---

## 🎯 프로젝트 목표

1. **프롬프트 자동화**  
   - 사용자가 입력한 한국어/키워드를 기반으로 AI가 **묘사적인 영어 프롬프트**를 생성  

2. **LoRA 기반 스타일링**  
   - LoRA 가중치(만화/애니, 미드저니, 리얼리즘 등)를 적용하여 다양한 스타일 표현  

3. **사용자 친화성**  
   - Slack 채널에서 요청과 결과 확인이 가능 → 비기술 사용자도 쉽게 이미지 생성 가능  

4. **안정적 비동기 처리**  
   - Replicate API의 장시간 생성 프로세스를 **Polling 로직**으로 안정적으로 처리  

---

## 🛠️ 기술 스택

- **워크플로우 엔진**: n8n(self-hosted)
- **트리거/I/O**: Slack Trigger, Slack Message, Slack File Upload  
- **AI 모델**: OpenAI Chat Model (gpt-4.1-mini)  
- **프롬프트 엔진**: LangChain 기반 AI Agent  
- **이미지 생성**: Replicate API (`black-forest-labs/flux-schnell-lora`)  
- **스타일 커스터마이징 (LoRA)**:  
  - **LoRA (Low-Rank Adaptation)**: 대규모 이미지 생성 모델을 전체 재학습하지 않고, 일부 파라미터만 미세 조정하여 특정 스타일(애니메이션, 리얼리즘, 미드저니 등)을 반영할 수 있는 경량화 학습 기법  
  - **만화/애니/카툰** → `huggingface.co/openfree/flux-chatgpt-ghibli-lora` (trigger: ghibli)  
  - **미드저니(기본값)** → `huggingface.co/strangerzonehf/Flux-Midjourney-Mix2-LoRA` (trigger: MJ v6)  
  - **리얼리즘/실사** → `huggingface.co/strangerzonehf/Flux-Super-Realism-LoRA` (trigger: Super Realism)  

---

## ⚙️ 워크플로우 동작 순서

1. **Slack Trigger**  
   - Slack 메시지 또는 멘션 이벤트를 감지하여 워크플로우 실행  

2. **진행 알림**  
   - `"이미지 생성중.. 잠시만 기다려주세요."` 메시지를 Slack 채널에 전송  

3. **AI Agent 실행**  
   - OpenAI(gpt-4.1-mini) 모델을 이용해 사용자의 설명을 분석  
   - 프롬프트, 이미지 비율(ratio), LoRA 주소, 트리거 단어를 생성  

4. **Replicate API 요청 (Generate Image)**  
   - 생성된 프롬프트와 LoRA 파라미터를 flux-schnell-lora 모델에 전달하여 이미지 생성 요청  

5. **대기 및 상태 확인 (Wait + Get Status)**  
   - 일정 시간 대기 후 Replicate의 상태를 확인  
   - 상태 값에 따라 분기 처리:  
     - `processing` → polling 노드로 이동해 반복 확인  
     - `succeeded` → 이미지 다운로드 단계로 이동  
     - `failed` → 실패 처리 분기로 이동 (현재는 미구현)

6. **Get Image & Upload**  
   - 성공 시 최종 이미지 URL을 다운로드  
   - Slack 채널에 이미지 파일로 업로드  

---

## 🚀 성과 및 기대 효과

- **프롬프트 엔지니어링 자동화**: 짧은 입력도 고품질 시각적 프롬프트로 변환  
- **LoRA 스타일 제어**: 내장된 LoRA 프리셋으로 다양한 아트 스타일 적용 가능  
- **워크플로우 확장성**: 추가 LoRA 스타일이나 다른 이미지 생성 모델로 쉽게 확장 가능  

---

## 💡 배운 점 및 개선 방향

### 배운 점
- **AI 기반 프롬프트 엔지니어링 효과**: 사용자 입력을 그대로 전달하는 것보다 품질 높은 결과 확보에 필수  
- **이미지 생성 AI 활용 이해**: Replicate API 호출과 Slack 연동 과정을 통해 이미지 생성 AI의 기본 구조(프롬프트 구성, 비동기 처리, API 호출 방식) 학습
- **LoRA 개념 이해**: LoRA는 대규모 모델 전체를 다시 학습하지 않고 일부 파라미터만 미세 조정하여 특정 아트 스타일을 반영할 수 있는 경량화 기법으로, 원하는 스타일을 손쉽게 적용하는 데 필수  

### 개선 방향
- **에러 핸들링 강화**: 이미지 생성 실패 시 Slack에 오류 메시지와 로그를 자동 전송하도록 개선  
- **LoRA 활용 다양화**:  
  현재는 일부 LoRA(만화/애니, 미드저니, 리얼리즘)만 지원되지만, 향후에는 사용자의 요청을 다방면으로 분석하여 더 많은 LoRA를 활용할 수 있도록 개선 필요 
  - **자연어 처리 기반 분류**: 전체 문맥을 분석하여 적합한 스타일 선택  
  - **추천 시스템 확장**: 여러 LoRA 후보를 점수화하여 자동 선택하거나 사용자 옵션 제공  
  - **LoRA 라이브러리 확장**: HuggingFace 등 공개 LoRA 모델을 손쉽게 추가·교체 가능한 구조로 개선  
- **이미지 업로드 기반 편집 기능 추가**:  
  단순 생성뿐만 아니라 사용자가 업로드한 이미지를 기반으로 **수정·보정·스타일 변환**을 지원하는 기능 추가
---

