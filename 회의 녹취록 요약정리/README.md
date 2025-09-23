# 📝 회의녹취록 요약·정리 자동화 워크플로우

## 🌟 프로젝트 개요

이 워크플로우는 회의 녹취 파일의 **수집 → STT 변환 → 원문 교정 → 요약 → 기록 및 알림**까지 전 과정을 자동화하는 **n8n 기반 Agent**입니다.  
Google Drive에 업로드된 회의 녹취 파일 생성 이벤트를 감지하여 AssemblyAI로 STT(Speech-to-Text)를 수행한 뒤, **LLM 기반 AI Agent**들이 순차적으로 화자 분석, 원문 교정, 아젠다 및 액션 아이템 분리 요약을 수행합니다.  
최종 결과는 **Notion**에 기록되며, **Slack 및 이메일**을 통해 담당자에게 액션 아이템이 자동 전달됩니다.

---

## 🎯 프로젝트 목표

1. **녹취록-텍스트 자동 변환**  
   - Google Drive 업로드 이벤트 발생 시 AssemblyAI를 사용하여 음성 파일을 자동 전사  

2. **회의록 제작**  
   - AI Agent를 활용해 STT 결과의 오인식 교정 및 구어체/비속어 정제  
   - 발언자별 내용 구분을 명확히 하여 가독성 높은 회의록 생성  

3. **핵심 정보 추출 및 요약**  
   - 교정된 회의록 원문을 기반으로 아젠다별 내용 정리  
   - 담당자·우선순위가 명시된 액션 아이템을 별도로 추출  

4. **자동화된 결과 공유**  
   - 요약된 회의 내용을 Notion에 기록  
   - 액션 아이템은 담당자별 Slack 메시지(Markdown) 및 이메일(HTML)로 자동 전송  

---

## 🛠️ 기술 스택

- **n8n**: 자동화 워크플로우 도구 (self-hosted)
- **Google Drive**: 녹취 파일 업로드 트리거  
- **AssemblyAI**: 음성 파일 전사(STT), 화자 레이블링·언어 감지 지원  
- **AI Agent (LangChain 기반, GPT-4o / GPT-4.1-mini)**:  
  - 화자 매핑 
  - STT 원문 교정 및 정제  
  - 아젠다·액션 아이템 요약  
  - Slack/이메일 메시지 생성  
- **Notion API**: 회의 요약 및 아젠다 기록  
- **Slack / Gmail**: 액션 아이템 담당자 알림  

---

## ⚙️ 워크플로우 동작 순서

1. **파일 감지 및 전사 (STT)**  
   - Google Drive Trigger → 특정 폴더에 파일 업로드 이벤트 감지  
   - AssemblyAI API 호출 → `speaker_labels: true`, `language_detection: true` 옵션 활성화  
   - `If Completed` + `Wait` 노드 활용 → 전사 완료(`status: completed`) 확인 Polling  

2. **화자 분석 및 매핑**  
   - STT 결과의 상징적 화자(A, B 등)를 실제 참여자 정보({name, role, email, slackID, discordID})와 매핑  
   - 대화 맥락 기반으로 가장 적합한 화자를 추론하는 AI Agent 실행  

3. **STT 교정 및 정제**  
   - AI Agent(`한국어 회의록 STT 교정·제작 보조 에이전트`) 실행  
   - 오인식 교정, 구어체/비속어 정제, 발언자 구분 명확화  
   - JSON 스키마(`correction_result`, `correction_reason`) 기반으로 구조화된 결과 출력  

4. **회의 내용 요약 및 분리**  
   - 교정된 회의록을 입력으로 아젠다별 내용 정리  
   - 액션 아이템 추출: `description`, `manager`, `priority(High/Medium/Low)` 포함  

5. **Notion 기록**  
   - 회의 내용을 기록할 Notion 페이지 생성  
   - `Loop Over Items` 노드 사용 → 아젠다 단위로 블록 추가  

6. **결과 알림**  
   - 담당자별 액션 아이템 매핑  
   - AI Agent2 실행 → Slack 메시지(Markdown) / 이메일(Gmail, HTML) 자동 생성  
   - Slack ID가 존재하는 경우 Slack, 없으면 이메일로 발송  

---

## 🚀 성과 및 기대 효과

- **업무 자동화**: 회의록 정리·공유 과정 완전 자동화 → 수작업 시간 절감  
- **정확성 향상**: STT 교정 및 발언자 매핑으로 가독성과 신뢰도 높은 회의록 제공  
- **협업 강화**: 액션 아이템을 담당자별로 자동 전달 → 후속 조치 신속화  
- **지식 자산화**: 표준화된 회의 기록이 Notion에 축적 → 장기적인 레퍼런스 확보  

---

## 💡 배운 점 및 개선 방향

### 한계점
- **참여자 정보 수동 설정 필요**: 워크플로우 실행 전 참여자 정보를 인원수에 맞게 직접 설정해야 함  
- **화자 추론의 한계**: 직책·역할 기반 추론 방식은 특정 상황에서 오판 가능성 존재  
- **Notion 연동의 복잡성**: 블록 단위 구조로 인해 아젠다별 Loop 설계 필요  

### 개선 방향
- **Polling 안정성 확보**: AssemblyAI 전사 완료 Polling에 `최대 시도 횟수(max try cnt)`를 추가하여 무한 대기 방지  
- **참여자 관리 자동화**: Google Contacts/HR 시스템과 연계하여 화자 매핑 자동화  
- **Notion 기록 최적화**: 표준 블록 템플릿 정의 → 기록 구조 단순화  
- **멀티 채널 확장**: Slack/이메일 외에 Teams, Discord 등 협업 툴 알림 확장 검토  

---

