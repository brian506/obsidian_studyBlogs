### 서비스별 한 줄 요약 ⭐⭐⭐

|서비스|기능|시험 키워드|
|---|---|---|
|**Rekognition**|이미지/비디오 분석|얼굴 인식, 유명인, 콘텐츠 조정, 라벨링|
|**Transcribe**|음성 → 텍스트|자막 생성, PII 자동 제거, 다국어|
|**Polly**|텍스트 → 음성|TTS, Lexicon, SSML|
|**Translate**|언어 번역|로컬라이제이션, 대량 번역|
|**Lex**|대화형 AI|챗봇, 콜센터 봇, Alexa 기술|
|**Connect**|클라우드 콜센터|가상 고객센터, CRM 통합, 80% 비용 절감|
|**Comprehend**|텍스트 분석 NLP|감정 분석, 핵심 문구 추출, 서버리스|
|**Comprehend Medical**|의료 텍스트 분석|PHI 감지, DetectPHI API|
|**SageMaker**|ML 모델 전체 라이프사이클|개발자/데이터과학자용, 라벨링~배포|
|**Forecast**|시계열 예측|수요 예측, 데이터보다 50% 정확|
|**Kendra**|문서 검색 엔진|자연어 검색, 증분 학습|
|**Personalize**|실시간 맞춤 추천|Amazon.com 동일 기술, 며칠 만에 구현|
|**Textract**|문서에서 데이터 추출|스캔 문서, 손글씨, 양식/테이블|

---

### 주요 서비스 상세 ⭐

**Rekognition — 이미지/비디오 분석**

- 객체, 사람, 텍스트, 장면 탐지
- 얼굴 분석(성별, 나이, 감정), 유명인 인식, 경로 추적
- **Content Moderation**: 부적절 콘텐츠 감지, 신뢰 임계값 설정, 인적 검토는 **Amazon A2I** 연동

**Transcribe — 음성 → 텍스트**

- ASR(자동 음성 인식) 딥러닝 사용
- **PII 자동 제거 (Redaction)** 기능
- 다국어 오디오 자동 언어 식별
- 활용: 고객 통화 전사, 자막 생성, 의료 연동(Comprehend Medical)

**Polly — 텍스트 → 음성**

- **Lexicon**: 발음 어휘 목록 (약어, 스타일링 단어 사용자 정의)
- **SSML**: 강조, 속삭임, 숨소리 등 세밀한 음성 제어

**Lex + Connect 콤보 ⭐**

- Lex: 챗봇/콜봇 구축 (Alexa 동일 기술, ASR + 자연어 이해)
- Connect: 클라우드 가상 고객 센터, 전통 솔루션 대비 **80% 저렴**, 예치금 없음
- 연동 패턴: 전화 수신 → Connect → Lex → Lambda → CRM

**Comprehend vs Comprehend Medical**

- Comprehend: 일반 텍스트 NLP, 서버리스, 감정 분석, 문서 주제 분류
- Comprehend Medical: 의료 텍스트 전용, **PHI 감지 (DetectPHI API)**, 의사 소견서/퇴원 요약서 분석

**Kendra vs Personalize — 혼동 주의**

- **Kendra**: 문서에서 답 찾기 (검색 엔진), 사용자 피드백으로 증분 학습
- **Personalize**: 사용자 행동 기반 **추천** (Amazon.com 추천 기술)

**Textract — 문서 데이터 추출**

- 스캔 이미지/PDF에서 텍스트 + 손글씨 + 테이블 + 양식 데이터 추출
- 단순 OCR이 아닌 구조적 데이터(표, 양식 필드) 인식
- 사용 사례: 세금 서류, 신분증, 의료 청구서, 재무 보고서

---

### 시험 TIP — 문제 유형별 정답

| 상황                     | 정답                     |
| ---------------------- | ---------------------- |
| 이미지에서 얼굴/유명인 인식        | **Rekognition**        |
| 부적절 콘텐츠 자동 필터링 + 인적 검토 | **Rekognition + A2I**  |
| 콜센터 통화 내용을 텍스트로 변환     | **Transcribe**         |
| 텍스트를 음성으로 읽어주기         | **Polly**              |
| 다국어 웹사이트 번역 자동화        | **Translate**          |
| 고객 응대 챗봇/콜봇 구축         | **Lex**                |
| 클라우드 가상 고객 센터          | **Connect**            |
| 이메일/리뷰 감정 분석           | **Comprehend**         |
| 의료 기록에서 환자 정보(PHI) 추출  | **Comprehend Medical** |
| ML 모델 직접 학습/배포 플랫폼     | **SageMaker**          |
| 제품 수요/판매량 예측           | **Forecast**           |
| 사내 문서에서 자연어로 답 검색      | **Kendra**             |
| 쇼핑몰 개인화 추천             | **Personalize**        |
| 스캔 세금 양식/신분증에서 데이터 추출  | **Textract**           |