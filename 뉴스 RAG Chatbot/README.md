# 📰 뉴스 자동화 시스템: RAG 구축 및 챗봇 워크플로우

## 🌟 프로젝트 개요

이 프로젝트는 **뉴스 콘텐츠 수집 → 임베딩 및 벡터 DB 저장 → 일일 요약 알림 → 대화형 챗봇 질의응답**까지 전 과정을 자동화한 **n8n 기반 AI Agent 시스템**입니다.  
뉴스를 다양한 소스로부터 자동 수집하고 벡터화하여 RAG(Retrieval-Augmented Generation) 구조를 구축하며, 요약된 뉴스를 매일 전달하고, 사용자 질의에 대해 벡터 DB 및 웹 검색을 통해 지능적으로 응답하는 **엔드투엔드 자동화 시스템**입니다.

본 시스템은 아래 3개의 주요 워크플로우로 구성됩니다:

1. **news_rag**: 다양한 RSS 피드에서 뉴스를 수집하고 Redis를 활용하여 중복을 방지하며, OpenAI 임베딩을 통해 Supabase Vector Store에 데이터를 적재하는 **RAG 구축 파이프라인**  
2. **news-feeds**: 수집된 최신 뉴스를 AI Agent(Google Gemini Chat Model)가 요약하여 **Telegram 메시지로 일일 뉴스 요약 알림** 전송  
3. **news_rag_chatbot**: Telegram Trigger를 통해 사용자 질문을 받아, **Vector DB 검색 → 필요 시 웹 검색 보조**를 수행하는 **대화형 챗봇 워크플로우**
4. **search_tool**: Naver Web, Naver Blog, Google Search **웹 검색 기능을 지원하는 Sub-workflow**

---

## 🎯 프로젝트 목표

1. **뉴스 데이터베이스 구축 및 관리**  
   - 정치·경제·사회·국제·IT 등 다양한 카테고리의 RSS 피드에서 뉴스를 정기적으로 수집  
   - Redis 키를 활용하여 중복 저장을 방지하고 안정적인 Supabase 기반 벡터 DB 적재 파이프라인 구현  

2. **효율적인 뉴스 요약 및 전달**  
   - 수집된 뉴스 콘텐츠를 Gemini 모델 기반 AI Agent가 요약  
   - 매일 주요 뉴스를 Telegram을 통해 간편하게 받아볼 수 있도록 자동화  

3. **지능형 질의응답 서비스 제공** 
   - 사용자의 질문에 대해 Vector DB에서 검색된 뉴스 기반으로 답변  
   - 정보가 부족하거나 최신 데이터가 필요한 경우 `web_search_tool`을 활용하여 실시간 검색 보강  

---

## 🛠️ 기술 스택

- **n8n**: 자동화 워크플로우 플랫폼(self-hosted)
- **RSS Feeds**: 뉴스 원문 데이터 소스  
- **Redis**: 중복 뉴스 필터링 및 키 관리  
- **OpenAI Embeddings**: 뉴스 콘텐츠 임베딩 처리  
- **Supabase Vector Store**: 임베딩 데이터 저장소 (RAG용 Vector DB)  
- **Google Gemini Chat Model**: 뉴스 요약 생성  
- **OpenAI GPT-4.1-mini**: 챗봇 질의응답 생성  
- **Telegram**: 사용자 알림 및 질의응답 인터페이스  
- **web_search_tool**: 벡터 DB의 정보 부족 시 보조 검색 기능  
   - Google search API
   - Naver web search API
   - Naver blog search API

---

## 📊 워크플로우

### 1. news_rag
![img](https://github.com/gyunih0/n8n_workflows/blob/main/%EB%89%B4%EC%8A%A4%20RAG%20Chatbot/image/news_rag_img.png?raw=true)

### 2. news_feeds
![img](https://github.com/gyunih0/n8n_workflows/blob/main/%EB%89%B4%EC%8A%A4%20RAG%20Chatbot/image/news_feeds_img.png?raw=true)

### 3. news_rag_chatbot
![img](https://github.com/gyunih0/n8n_workflows/blob/main/%EB%89%B4%EC%8A%A4%20RAG%20Chatbot/image/news_rag_chatbot_img.png?raw=true)

### 4. search_tool
![img](https://github.com/gyunih0/n8n_workflows/blob/main/%EB%89%B4%EC%8A%A4%20RAG%20Chatbot/image/search_tool_img.png?raw=true)

---

## ⚙️ 워크플로우 동작 순서

### 1. 뉴스 수집 및 RAG 구축 (`news_rag`)
- 다중 RSS 피드(최신, 헤드라인, 정치, 경제, 사회 등)를 설정하여 뉴스 수집  
- `Split Out` + `Loop Over Items` 노드를 통해 뉴스 항목을 처리  
- Redis에서 뉴스의 `link`를 키로 조회 → 중복 여부 확인  
- 새로운 뉴스만 `Recursive Character Text Splitter`로 분할 → OpenAI 임베딩 수행 (chunk size: 3000, chunk overlab: 500)
- 임베딩 결과를 Supabase Vector Store의 `documents` 테이블에 저장  
- 임베딩 완료 후엔 `news-feeds` 워크플로우 실행

### 2. 뉴스 요약 알림 (`news-feeds`)
- RSS 피드에서 수집한 뉴스 데이터를 통합(`Aggregate`)  
- Gemini Chat Model 기반 AI Agent가 뉴스 내용을 간결하게 요약  
- 시스템 메시지를 통해 **요약만 출력**하도록 제어  
- 결과를 Telegram 메시지로 사용자에게 전송  

### 3. 대화형 챗봇 질의응답 (`news_rag_chatbot`)
- Telegram Trigger를 통해 사용자 질문 수신  
- `Postgres Chat Memory`를 사용하여 대화 맥락 유지  
- OpenAI GPT-4.1-mini 기반 AI Agent가 질의응답 수행  
- **Vector DB 검색 → 실패 시 web_search_tool 보조 검색**  
- 웹 검색 결과 사용 시 반드시 `"🔎 웹 검색 결과를 기반으로 정리한 내용입니다."` 문구 포함  

---

## 🚀 성과 및 기대 효과

- **완전 자동화된 뉴스 RAG 시스템**: 수집부터 임베딩, 검색, 응답까지 전 과정 자동화  
- **최신 이슈 요약 및 실시간 인사이트 확보**: 매일 핵심 뉴스 요약을 빠르게 받아볼 수 있음  
- **지능형 Q&A 서비스 제공**: 축적된 뉴스 DB 기반 질의응답과 실시간 검색 보강으로 높은 정확도 확보  
- **높은 유지보수성과 확장성**: 기능별 워크플로우 분리로 변경 및 기능 추가에 유연하게 대응 가능  

---

## 💡 배운 점 및 개선 방향

### 배운 점 (한계 및 해결)
- **중첩 루프 및 DB 노드 제약**: Supabase의 `Get a row` 노드는 입력 아이템이 여러 개여도 하나만 반환 → Redis 기반 중복 필터링 구조로 대체  
- **루프 설계 지침**: n8n에서 loop를 중첩해 사용하는 것은 권장 및 지원하지 않음.
- **모델 효율성 고려**: 뉴스 요약은 고성능 모델 대신 효율적인 LLM으로도 충분함  

### 개선 방향
- **중복 체크 방식 개선**: Redis를 활용하여 뉴스 중복 여부를 확인하고 있으나, 메모리 기반 DB 특성으로 인해 데이터가 휘발될 수 있는 한계가 존재하므로 영속적인 저장소나 데이터베이스 테이블을 활용하여 보다 안정적이고 신뢰성 있는 중복 관리 방안을 마련하는 방향으로 개선 필요.
- **챗봇 시스템 메시지 고도화**: 예외 상황(벡터 DB 검색 실패, 웹 검색 요청 등)에 대한 세부 응답 규칙 추가 → 사용자 경험 개선  
- **추가 기능 확장**: 뉴스 요약 알림에 카테고리별 필터, 중요도 기반 하이라이트 기능 추가 검토  

---

