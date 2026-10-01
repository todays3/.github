<h1 align="center">하루만장</h1>

<p align="center">
  <b>예약 시각에 맞춰 뉴스를 모으고, 온프레미스 LLM으로 골라 카카오톡으로 보내는 개인화 큐레이션 서비스</b>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white" />
  <img src="https://img.shields.io/badge/FastAPI-009688?style=for-the-badge&logo=fastapi&logoColor=white" />
  <img src="https://img.shields.io/badge/APScheduler-2E7D32.png?style=for-the-badge" />
  <img src="https://img.shields.io/badge/SQLAlchemy-D71F00?style=for-the-badge&logo=sqlalchemy&logoColor=white" />
  <br/>
  <img src="https://img.shields.io/badge/Ollama-000000?style=for-the-badge&logo=ollama&logoColor=white" />
  <img src="https://img.shields.io/badge/Qwen2.5-615EEB.png?style=for-the-badge&logo=qwen&logoColor=white" />
  <img src="https://img.shields.io/badge/Groq-F55036?style=for-the-badge&logo=groq&logoColor=white" />
  <img src="https://img.shields.io/badge/Kakao-FEE500.png?style=for-the-badge&logo=kakaotalk&logoColor=000" />
</p>

---

사용자는 주제·데스크·발송 시각만 정해 둡니다. 서버는 그 시각 전에 RSS와 유튜브를 모아 세 꼭지를 고르고, 시각이 되면 카카오톡으로 전달합니다.

```
발송 시각 · 주제 · 데스크
→ 선행 시간만큼 미리 수집 · 초안 작성
→ 가까운 시각끼리는 수집을 한 번만
→ 로컬 LLM 우선, 실패 시 Groq, 둘 다 실패 시 규칙 기반 선정
→ 예약 시각에 카카오 전송, 놓치면 당일 미발송만 재시도
```

## 핵심 문제 해결

### 발송 정시성 — 작성과 전송을 다른 틱으로 분리

로컬 큐레이션은 수십 초에서 수분까지 흔들립니다. 예약 시각에 수집과 생성을 시작하면 카카오 전송이 슬롯을 넘깁니다. 최근 예약 실행 30건의 준비 시간 P90에 5초를 더해 선행 시간을 잡고, 15초 틱은 초안만 만들고 5초 틱은 만들어진 초안만 보냅니다. 슬롯을 놓친 초안은 당일 안에만 다시 보내며, 실패는 3회에서 멈춥니다.

### 크롤 효율 — 30분 안의 시각은 수집을 공유

발송 시각이 겹치면 사용자마다 같은 RSS·유튜브를 다시 받습니다. 같은 날·같은 시간대에서 30분 창으로 슬롯을 묶고, 주제와 소스를 합쳐 한 번만 수집합니다. 동시에 들어온 작성 요청은 리더만 수집하고 나머지는 그 결과를 기다립니다. 각 사용자는 자기 소스만 잘라 개인화하고, 작성·전송 병렬도는 기본 3, 상한 8입니다.

### LLM 서빙 — 로컬을 먼저 쓰고, 한도와 장애는 호출 전에 막음

온프레미스 모델이 죽거나 Groq 무료 한도(30 RPM, 8,000 TPM)에 걸리면 큐레이션이 그대로 실패합니다. 한글 프롬프트는 토큰을 글자 수의 절반으로 잡으면 한도를 넘겨 429가 납니다. 호출은 로컬을 먼저 시도하고, 연결 실패나 모델 없음이면 일정 시간 로컬을 건너뛴 뒤 Groq로 넘깁니다. 분당 토큰·요청은 한도의 85%만 슬라이딩 윈도로 쓰고, 한글은 글자 수를 그대로 토큰으로 선점합니다. 둘 다 실패하면 규칙 기반으로 골라 발송은 유지합니다.

## 아키텍처

세 문제는 한 파이프라인 위의 서로 다른 층입니다. 스케줄러가 시각을 나누고, 수집 층이 크롤을 합치고, LLM 층이 프로바이더를 가립니다.

```
                    ┌─ prepare (15s) ─ 초안만 생성
APScheduler ────────┼─ send (5s) ──── 시각 이후 전송
                    └─ catchup ────── 당일 미발송만 재시도
                              │
                    선행 시간 = P90(준비 시간) + 5초
                              │
              30분 창으로 슬롯 묶음
                              │
                    주제·소스 합집합, 리더 1회 수집
                    (동시 요청은 대기, 풀 유지 50분)
                              │
                    사용자별 소스만 절단
                              │
              ┌─ 로컬 LLM (정상일 때만)
큐레이션 ─────┼─ 실패·비가용 → Groq (TPM/RPM 85%)
              └─ 둘 다 실패 → 규칙 기반 선정
                              │
                         카카오톡 전송
```

| 층 | 하는 일 | 지키려는 것 |
|---|---|---|
| 스케줄 | 선행 작성 / 정시 전송 / 당일 재시도를 위상으로 분리 | 생성 지연이 발송 시각을 밀지 않음 |
| 수집 | 가까운 슬롯은 풀 하나, 개인화만 사용자별 | 같은 피드를 반복해서 받지 않음 |
| LLM | 로컬 → 원격 → 규칙 선정, 호출 전 한도 선점 | 모델 장애와 429가 발송 중단으로 이어지지 않음 |
