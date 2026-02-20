# KCI 논문 저자 식별 알고리즘 통합 전략 보고서 (Final)

## 저자 역할 분류 + 동명이인 구분 + 동일저자 추적 + LLM 알고리즘 비교 + 온톨로지 데이터 구조

---

## Executive Summary

성균관대학교 연구성과관리시스템(RIMS)의 KCI 논문 24,532건에 대해 저자 역할(교신저자, 1저자, 공동1저자, 순위저자)을 정확하게 식별하고, 동명이인 구분 및 동일저자 추적까지 수행하는 자동화 시스템을 구축한다. 이후 결과를 Biblo 온톨로지 전략을 적용한 데이터 구조로 축적하여 연구자 지식 기반을 구축한다.

**핵심 전략:**
- KCI Open API + PDF 1페이지 파싱 이중 경로로 저자 역할 식별
- **수학 알고리즘과 LLM 자연어 프롬프트 알고리즘을 병행**하여 최적의 식별 정확도 달성
- RIMS 축적 데이터를 활용한 5-Signal 저자 동일성 판별
- 5-Phase Pipeline: 수집 → 역할 판별 → 저자 동일성 검증 → RIMS 업데이트 → 온톨로지 구축
- Biblo 온톨로지 전략 적용: ID 기반 엔티티 체계 + 월별 증분 파이프라인

---

## Part I: 현황 분석

### 1. RIMS 데이터 현황

| 항목 | 수치 |
|------|------|
| 전체 논문 | 136,952건 |
| KCI 논문 (대상) | 24,532건 |
| 저자 참여 레코드 | 5,134,679건 |
| 고유 이름 수 | 312,456명 |
| 고유 연구자 ID (PRTCPNT_ID) | 19,666명 |
| PRTCPNT_ID 미보유 레코드 | 4,760,125건 (92.7%) |

### 2. TPI_DVS_CD (저자 역할 코드) 분포

| 코드 | 의미 | 건수 | 정확도 |
|------|------|------|--------|
| 2 | 1저자 | 9,851 | 비교적 정확 |
| 3 | 교신저자 | 3,797 | 일부만 식별 |
| 4 | 공저자 | 486,351 | **미분류 혼재** |

**핵심 문제**: TPI=4에 교신저자, 2저자, 순위저자가 미분류 상태로 혼재 (486,351건)

### 3. 동명이인 현황

| 항목 | 수치 |
|------|------|
| 동명이인 (같은 이름, 다른 ID) | 13,577명 |
| 최다 동명이인 | Kim, J. (214명), Lee, J. (210명) |
| 소속 변경 이력자 (같은 ID, 2개+ 소속) | 6,167명 |

### 4. 저자 동일성 판별에 활용 가능한 RIMS 데이터

| 데이터 | 보유율 | 활용 방안 |
|--------|--------|-----------|
| 논문 제목 (ORG_LANG_PPR_NM) | 100% | 연구 주제 유사도 계산 |
| 초록 (ABST_CNTN) | 87.0% (KCI 97.4%) | TF-IDF / 임베딩 기반 유사도 |
| 학술지명 (SCJNL_NM) | 100% | 연구 분야 추정 |
| 발행연도 (PBLC_YM) | 100% | 시계열 경력 추적 |
| ORCID | 69명 | 확정적 식별자 (보유 시) |
| SCOPUS_ID | 1,240명 | 확정적 식별자 (보유 시) |
| 이메일 (EMAL_ADDR) | 10,437명 | 준확정적 식별자 |
| 공저자 네트워크 | 78,509 공저논문 | 동명이인 구분의 강력한 신호 |
| 소속기관 (BLNG_AGC_NM) | 대부분 보유 | 시계열 소속 변경 추적 |

---

## Part II: 알고리즘 설계

### 5. 알고리즘 아키텍처: 5-Phase Pipeline

```
Phase 1: 데이터 수집          Phase 2: 저자 역할 판별
  RIMS → KCI ID/DOI 추출        API + PDF 교차 검증
  Track A: KCI API               의사결정 트리 / LLM 프롬프트
  Track C: PDF 1페이지 파싱       역할 분류 (5개 유형)
        │                              │
        ▼                              ▼
Phase 3: 저자 동일성 검증 ◀──── Phase 2 결과
  동명이인 구분 (Disambiguation)
  동일저자 추적 (Entity Resolution)
  수학 점수 모델 / LLM 판단 / 하이브리드
        │
        ▼
Phase 4: 매칭 및 업데이트
  퍼지 매칭 + 동일성 검증 결합
  RIMS 데이터 보정
  최종 산출물 생성
        │
        ▼
Phase 5: 온톨로지 구축 (신규)
  엔티티 ID 부여 (기관/분야/키워드)
  계층 구조 + aliases 매핑
  월별 증분 파이프라인 자동화
```

### 6. Phase 1: 데이터 수집

#### 6.1 대상 논문 필터링
- RIMS에서 ID_KCI IS NOT NULL → 24,532건
- DOI + KCI ID 모두 보유: 15,872건
- KCI ID만 보유: 8,660건

#### 6.2 Track A: KCI Open API
- 엔드포인트: `open.kci.go.kr/po/openapi/openApiSearch.kci`
- apiCode: articleDetail
- 출력: author-division (1=주저자, 2=공저자), author-part (역할 텍스트), ORCID, 소속
- 제약: API 키 발급 필요, 일 5,000건 제한

#### 6.3 Track C: PDF 1페이지 파싱 (핵심 경로)
1. KCI 웹페이지 접속 → "원문 보러가기" 링크 추출
2. PDF 1페이지 다운로드
3. 텍스트 추출 (PyMuPDF / Tesseract OCR)
4. 저자명 + 역할 기호(* † ‡) 파싱
5. 하단 저자 소개 블록 파싱 (소속, 이메일, ISNI)

**기호 규칙**:
| 기호 | 의미 |
|------|------|
| * | 교신저자 (corresponding author) |
| ** | 공동교신저자 |
| † | 공동제1저자 (equal contribution) |
| 순서 1번째 | 제1저자 |
| 마지막 순서 | 시니어/교신 후보 |

### 7. Phase 2: 저자 역할 판별

#### 7.1 수학 알고리즘: 의사결정 트리

```
PDF에서 * 기호?  ──YES──→  교신저자 (순서 1번이면 1저자 겸 교신)
      │ NO
PDF에서 † 기호?  ──YES──→  공동 제1저자
      │ NO
저자 순서 1번?   ──YES──→  1저자
      │ NO
마지막 저자(≥3명)?──YES──→  마지막저자/시니어 (API로 교신 확인)
      │ NO
API '교신' 포함?  ──YES──→  교신저자 (API 기반)
      │ NO
      └──────────→  공저자 + 순서 번호
```

#### 7.2 신뢰도 산출

| 판별 조건 | 신뢰도 |
|-----------|--------|
| API + PDF 일치 | 1.0 |
| PDF 기호만 확인 | 0.9 |
| API만 확인 | 0.8 |
| 순서 기반 추정 | 0.6 |
| API ≠ PDF 불일치 | 0.4 |
| 판별 불가 | 0.0 → 수동 |

---

## Part III: LLM 자연어 프롬프트 알고리즘 (신규)

### 8. LLM 기반 저자 식별 전략

#### 8.1 기본 개념

기존 수학 알고리즘(의사결정 트리, 가중 점수 모델)은 사전 정의된 규칙을 순차적으로 적용하는 방식이다. 반면 **LLM 자연어 프롬프트 알고리즘**은 PDF/API에서 추출한 텍스트 데이터를 LLM에게 자연어로 맥락을 설명하고, 판단을 요청하는 방식이다.

**핵심 사상:** "수학 공식으로 코딩하지 않아도, 자연어로 판단 기준을 명확히 기술하면 LLM이 복합적 맥락을 이해하여 더 정확한 판단을 내릴 수 있다."

#### 8.2 적용 영역별 LLM 프롬프트 설계

**영역 1: 저자 역할 판별 (Phase 2 대체/보완)**

```
[System Prompt]
당신은 KCI 학술 논문의 저자 역할을 판별하는 전문가입니다.
다음 규칙을 적용하여 각 저자의 역할을 판별하세요:
- * 기호가 붙은 저자 → 교신저자 (corresponding author)
- † 기호가 붙은 저자 → 공동제1저자 (equal contribution)
- 저자 목록의 첫 번째 → 1저자 (first author)
- 저자 목록의 마지막(3명 이상일 때) → 시니어/교신 후보
- 하단 저자정보 블록에 "교신저자" 표기 → 교신저자 확정

[User Prompt]
다음은 KCI 논문 PDF 1페이지에서 추출한 텍스트입니다:

<PDF 텍스트>
{extracted_text}
</PDF 텍스트>

<KCI API 데이터>
{api_response}
</KCI API 데이터>

각 저자에 대해 다음 JSON 형식으로 역할을 판별해주세요:
{
  "authors": [
    {
      "name_kr": "한글명",
      "name_en": "영문명",
      "role": "first_author | corresponding | co_first | last_author | co_author",
      "rank": 순서번호,
      "confidence": 0.0~1.0,
      "evidence": "판별 근거 설명"
    }
  ]
}
```

**영역 2: 동명이인 구분 (Phase 3 대체/보완)**

```
[System Prompt]
당신은 학술 연구자의 동일성을 판별하는 전문가입니다.
동명이인 구분 시 다음 신호를 종합적으로 고려하세요:
1. 확정 식별자 (ORCID, 이메일) → 일치 시 동일인 확정
2. 공저자 네트워크 → 같은 공저자와 함께 논문 = 동일인 가능성 높음
3. 연구 주제 연속성 → 같은 분야 연구 = 동일인 가능성
4. 소속 변경 타당성 → 시간순으로 자연스러운 경력 진행인지
5. 메타데이터 → 기관명, 이니셜, 부가 식별자 등

[User Prompt]
RIMS에서 "김경수"를 검색하여 다음 3명의 후보를 발견했습니다:

<후보 1: ID=12345>
- 소속: 성균관대 언어AI 전공
- 최근 논문 주제: 자연어처리, 영어교육, 인공지능
- 공저자: 박철수, 이영희, 최민준
- 활동 기간: 2018~2022
</후보>

<후보 2: ID=67890>
- 소속: 서울대 물리학과
- 최근 논문 주제: 양자역학, 광학
- 공저자: 장동건, 정우성
- 활동 기간: 2015~2024
</후보>

<후보 3: ID=11111>
- 소속: 고려대 경영학과
- 최근 논문 주제: 마케팅, 소비자 행동
- 공저자: 한가인, 송혜교
- 활동 기간: 2020~2024
</후보>

새 논문 정보:
- 제목: "AI 기반 영어 교육 시스템의 효과 분석"
- 소속: 성균관대 영어영문학과 조교수
- 공저자: 박철수, 김민수
- 발행: 2024년

이 논문의 "김경수"는 위 3명의 후보 중 누구인지 판별하고,
판별 근거와 신뢰도를 JSON으로 제공하세요.
```

**영역 3: 비정형 텍스트 처리 (OCR 결과, 비표준 형식)**

```
[System Prompt]
다음은 OCR로 추출된 학술 논문 PDF의 저자 정보입니다.
OCR 노이즈가 있을 수 있으니, 맥락을 고려하여 저자명과 역할을 판별하세요.

[User Prompt]
OCR 추출 텍스트:
"김 철 수* · 이 영 희** · 박 민 수
*교 신저 자: 성 균관대 학교 소 프트웨 어학 과"

→ OCR 노이즈를 보정하고 저자 역할을 판별해주세요.
```

#### 8.3 LLM 알고리즘의 구현 아키텍처

```python
class LLMAuthorClassifier:
    """LLM 기반 저자 역할 판별기"""

    def __init__(self, model="claude-sonnet-4-5-20250929"):
        self.client = Anthropic()
        self.model = model
        self.system_prompt = ROLE_CLASSIFICATION_PROMPT

    def classify_authors(self, pdf_text, api_data=None):
        """PDF 텍스트 + API 데이터로 저자 역할 판별"""
        user_prompt = self._build_prompt(pdf_text, api_data)

        response = self.client.messages.create(
            model=self.model,
            system=self.system_prompt,
            messages=[{"role": "user", "content": user_prompt}],
            max_tokens=2000
        )

        return self._parse_response(response)


class LLMDisambiguator:
    """LLM 기반 동명이인 구분기"""

    def __init__(self, model="claude-sonnet-4-5-20250929"):
        self.client = Anthropic()
        self.model = model
        self.system_prompt = DISAMBIGUATION_PROMPT

    def disambiguate(self, candidate_name, candidates, new_paper):
        """후보 목록에서 새 논문의 저자를 식별"""
        user_prompt = self._build_context(candidates, new_paper)

        response = self.client.messages.create(
            model=self.model,
            system=self.system_prompt,
            messages=[{"role": "user", "content": user_prompt}],
            max_tokens=1500
        )

        return self._parse_result(response)
```

### 9. 수학 알고리즘 vs LLM 알고리즘 비교 분석

#### 9.1 비교 총괄표

| 평가 항목 | 수학 알고리즘 (의사결정 트리 + 가중 점수) | LLM 자연어 프롬프트 | 하이브리드 (권장) |
|----------|----------------------------------------|-------------------|----------------|
| **구현 복잡도** | 높음 (모든 케이스 규칙 정의 필요) | 낮음 (프롬프트만 작성) | 중간 |
| **정형 데이터 처리** | ★★★ 우수 (규칙 기반 정확) | ★★☆ 양호 | ★★★ |
| **비정형 데이터 처리** | ★☆☆ 취약 (OCR 노이즈, 비표준 형식) | ★★★ 우수 (맥락 이해) | ★★★ |
| **동명이인 구분** | ★★☆ (가중 점수 기계적 합산) | ★★★ (맥락적 종합 판단) | ★★★ |
| **처리 속도** | ★★★ 매우 빠름 (ms 단위) | ★☆☆ 느림 (API 호출, 초 단위) | ★★☆ |
| **비용** | ★★★ 무료 (로컬 연산) | ★☆☆ API 호출 비용 발생 | ★★☆ |
| **재현성** | ★★★ 완벽 (동일 입력 → 동일 출력) | ★★☆ 높으나 100% 아님 | ★★★ (수학으로 검증) |
| **엣지 케이스 대응** | ★☆☆ 사전 정의된 규칙 외 대응 불가 | ★★★ 새로운 패턴도 맥락으로 대응 | ★★★ |
| **설명 가능성** | ★★★ 점수 산식 투명 | ★★★ 자연어로 판단 근거 제공 | ★★★ |
| **유지보수** | ★☆☆ 새 규칙마다 코드 수정 필요 | ★★★ 프롬프트 수정만으로 조정 | ★★☆ |
| **대규모 배치** | ★★★ 24,532건 1시간 이내 | ★☆☆ 24,532건 수일 소요 | ★★☆ |
| **감사 추적성** | ★★★ 점수+근거 완벽 기록 | ★★☆ JSON 응답 기록 가능 | ★★★ |

#### 9.2 비용 분석

| 항목 | 수학 알고리즘 | LLM (Claude Sonnet 4.5) |
|------|-------------|------------------------|
| 24,532건 처리 | 서버 비용 ~$0 | API 호출 ~$50~100 |
| 동명이인 13,577건 | 서버 비용 ~$0 | API 호출 ~$30~60 |
| 월별 증분 (~2,000건) | ~$0 | ~$5~10/월 |
| 연간 운영비 | 서버비만 | ~$60~120 추가 |

#### 9.3 적합 영역 분석

**수학 알고리즘이 더 적합한 영역:**
1. **정형화된 규칙 적용**: PDF * 기호 → 교신저자 (규칙이 명확)
2. **대규모 배치 처리**: 24,532건 전체를 빠르게 처리
3. **확정 식별자 매칭**: ORCID, 이메일 등 정확한 1:1 매칭
4. **퍼지 이름 매칭**: Levenshtein 거리 등 정량적 유사도 계산
5. **공저자 Jaccard 유사도**: 집합 연산 기반 수치 계산

**LLM 알고리즘이 더 적합한 영역:**
1. **OCR 노이즈 처리**: 깨진 텍스트에서도 맥락으로 저자명/역할 추출
2. **비표준 저자 표기**: 학회마다 다른 저자 표기 관행 이해
3. **복합적 동명이인 판별**: 5개 신호를 사람처럼 종합적으로 판단
4. **소속 변경 타당성 평가**: "연구교수→조교수"의 자연스러움을 맥락으로 판단
5. **애매한 케이스 처리**: 수학 모델로 0.4~0.6점 나오는 경계 사례
6. **새로운 패턴 대응**: 코드 수정 없이 프롬프트 조정으로 즉시 대응

### 10. 권장 전략: 하이브리드 접근법

#### 10.1 하이브리드 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                      입력 데이터                              │
│    PDF 추출 텍스트 + KCI API + RIMS 기존 데이터                │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─ 1차: 수학 알고리즘 (빠르고 저비용) ───────────────────────┐
│  의사결정 트리 → 역할 판별                                   │
│  확정 ID 매칭 → 동일인 확정                                  │
│  가중 점수 모델 → 동일성 점수 산출                            │
│                                                             │
│  결과: 신뢰도 ≥ 0.7 → 자동 확정 (전체의 ~70%)               │
│        신뢰도 < 0.7 → 2차 판별로 이관 (~30%)                │
└──────────────────────────┬──────────────────────────────────┘
                           │ 신뢰도 < 0.7 케이스
                           ▼
┌─ 2차: LLM 프롬프트 (정확하고 맥락적) ─────────────────────┐
│  자연어 프롬프트로 맥락 전달                                  │
│  LLM이 종합적으로 판단 + 근거 제공                           │
│  OCR 노이즈 보정, 비표준 형식 처리                           │
│                                                             │
│  결과: 신뢰도 ≥ 0.7 → 자동 확정                             │
│        신뢰도 < 0.5 → 수동 검토 이관                        │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─ 교차 검증 ────────────────────────────────────────────────┐
│  수학 점수와 LLM 판단이 일치 → 최종 신뢰도 상향              │
│  불일치 → 수동 검토 대상으로 플래그                           │
└─────────────────────────────────────────────────────────────┘
```

#### 10.2 하이브리드 전략의 기대 효과

| 항목 | 수학 단독 | LLM 단독 | 하이브리드 |
|------|----------|---------|-----------|
| 자동 처리율 | ~70% | ~85% | **~88%** |
| 수동 검토 비율 | ~30% | ~15% | **~12%** |
| 처리 시간 (24K건) | ~1시간 | ~50시간 | **~5시간** |
| API 비용 | $0 | ~$100 | **~$30** (30%만 LLM) |
| 정확도 (정형) | 95% | 93% | **95%** |
| 정확도 (비정형) | 60% | 90% | **90%** |
| 정확도 (동명이인) | 80% | 88% | **90%** |

#### 10.3 최종 권장

**"수학 알고리즘 우선, LLM 보완" 하이브리드 전략을 권장한다.**

이유:
1. **비용 효율**: 전체의 ~70%는 수학 알고리즘으로 무료 처리, LLM은 ~30%에만 투입
2. **속도**: 대규모 배치에서 수학 알고리즘이 압도적으로 빠름
3. **재현성**: 수학 알고리즘의 결정론적 결과 + LLM의 감사 추적 결합
4. **정확도 극대화**: 수학 모델의 경계 사례를 LLM이 보완하여 전체 정확도 향상
5. **유지보수 용이**: 새로운 패턴은 프롬프트 수정으로 즉시 대응 가능

---

## Part IV: 저자 동일성 검증

### 11. 문제 정의

두 가지 핵심 과제:

**과제 A: 동명이인 구분 (Name Disambiguation)**
- "Kim, J."라는 이름이 214명의 서로 다른 연구자를 지칭
- PRTCPNT_ID가 없는 레코드가 92.7%
- 같은 이름이라도 다른 사람임을 구분해야 함

**과제 B: 동일저자 추적 (Entity Resolution)**
- 같은 연구자가 소속/직위 변경으로 다르게 기술됨
  - 예: "김경수, 성균관대 언어AI 전공 연구교수" → "김경수, 성균관대 영어영문학과 조교수"
- 6,167명이 2개 이상 소속 기관 보유

### 12. 다중 신호 기반 저자 동일성 판별 (Multi-Signal)

5개 신호를 종합하여 동일 저자 여부를 판별:

```
Identity Score = w₁·ID신호 + w₂·네트워크신호 + w₃·주제신호 + w₄·시계열신호 + w₅·메타신호
```

#### Signal 1: 확정 식별자 — 최우선

| 식별자 | 보유율 | 효과 |
|--------|--------|------|
| PRTCPNT_ID | 7.3% | 같은 ID → 동일인 확정 |
| ORCID | 0.4% | 같은 ORCID → 동일인 확정 |
| SCOPUS_ID | 6.3% | 같은 SCOPUS → 동일인 확정 |
| 이메일 | 53.1% (ID보유자) | 같은 이메일 → 동일인 확정 |

#### Signal 2: 공저자 네트워크 — w₂ = 0.30

- 같은 이름의 두 레코드가 같은 공저자와 논문 → 동일인 가능성 극히 높음
- Jaccard 유사도: `J(A, B) = |CoAuthors_A ∩ CoAuthors_B| / |CoAuthors_A ∪ CoAuthors_B|`
- RIMS에 78,509건의 공저 논문 존재

#### Signal 3: 연구 주제 유사도 — w₃ = 0.25

- TF-IDF + 코사인 유사도 / 학술지 분야 매칭
- 초록 보유율 97.4% (KCI)

#### Signal 4: 시계열 경력 추적 — w₄ = 0.20

- 발행연도로 연구 활동 타임라인 구성
- 소속 변경이 시간순으로 자연스러운지 검증

#### Signal 5: 메타데이터 매칭 — w₅ = 0.10

- 이니셜, 소속기관 부분 일치, KRI_ID 등 부가 식별자

### 13. 동일성 판별 — 수학 모델 vs LLM 모델

**수학 모델 (기본):**
```python
total = 0.30*s_network + 0.25*s_topic + 0.20*s_temporal + 0.10*s_meta
if total >= 0.7 and (best - second) >= 0.2:
    return "자동 확정"
```

**LLM 모델 (보완):**
수학 모델로 0.4~0.7 점수 구간의 경계 사례에 대해 LLM에게 후보 정보를 자연어로 제공하고, 종합 판단을 요청. LLM은 수치로 표현하기 어려운 맥락적 판단(소속 변경의 자연스러움, 연구 분야 전환의 합리성 등)에서 수학 모델 대비 우위.

### 14. 동일저자 추적 시나리오

**시나리오: "김경수" 소속 변경**

| 연도 | 소속 (PDF 추출) | RIMS 기존 데이터 |
|------|----------------|-----------------|
| 2022 | 성균관대 언어AI 전공 연구교수 | PRTCPNT_ID=12345 |
| 2024 | 성균관대 영어영문학과 조교수 | ??? (새 논문) |

**판별 과정:**
1. "김경수"로 RIMS 검색 → 후보 3명 (ID=12345, 67890, 11111)
2. Signal 1: 확정 ID 없음
3. Signal 2: 공저자 네트워크 강한 일치 (+0.3)
4. Signal 3: 주제 유사도 높음 (+0.2)
5. Signal 4: 시계열 타당 (+0.15)
6. Signal 5: 동일 대학 (+0.10)
7. **종합 0.75 → ID=12345로 확정**

---

## Part V: 매칭 및 업데이트

### 15. 5-Level 퍼지 매칭

| Level | 방법 | 신뢰도 |
|-------|------|--------|
| L1 | 완전 일치 (정규화 후) | 1.0 |
| L2 | 성 + 이니셜 매칭 | 0.9 |
| L3 | 한글 ↔ 영문 교차 (로마자 변환) | 0.8 |
| L4 | 소속 + 부분이름 + ORCID | 0.7 |
| L5 | 편집거리 (Levenshtein, rapidfuzz) | 0.5~ |

### 16. 최종 매칭 = 이름 매칭 × 동일성 검증

```
최종 신뢰도 = Name_Match_Score × Identity_Score
```

### 17. 업데이트 스키마

| 필드명 | 타입 | 구분 | 설명 |
|--------|------|------|------|
| TPI_DVS_CD | int | 보정 | 2=1저자, 3=교신, 4=공저자, 5=공동1저자 |
| AUTHOR_ROLE | varchar | 신규 | first / corresponding / co_first / last / co_author |
| AUTHOR_ORDER | int | 신규 | 논문 내 저자 순서 |
| ROLE_CONFIDENCE | float | 신규 | 역할 판별 신뢰도 |
| ROLE_SOURCE | varchar | 신규 | api / pdf / llm / hybrid / manual |
| MATCH_CONFIDENCE | float | 신규 | 이름 매칭 신뢰도 |
| IDENTITY_CONFIDENCE | float | 신규 | 동일성 검증 신뢰도 |
| IDENTITY_EVIDENCE | varchar | 신규 | 동일성 판별 근거 |
| IDENTITY_METHOD | varchar | 신규 | deterministic / math / llm / hybrid / manual |

### 18. 산출물

1. **rims_article_parti_updated.csv** — 보정된 RIMS 저자 데이터
2. **manual_review_list.xlsx** — 수동 검토 대상 (신뢰도 < 0.7)
3. **disambiguation_report.xlsx** — 동명이인 판별 결과 상세
4. **processing_report.html** — 처리 통계 리포트

---

## Part VI: 온톨로지 데이터 구조 (Biblo 전략 적용)

### 19. Biblo 온톨로지 전략 포인트 추출

Biblo 프로젝트(전략 문서 v3.0, Supabase 스키마, 백서)에서 추출한 핵심 전략 포인트 8가지:

| # | 전략 포인트 | 출처 | KCI 적용성 |
|---|-----------|------|-----------|
| SP-1 | FRBR 2계층 모델 (Work → Manifestation) | 백서, 전략v3 | ★★★ |
| SP-2 | 온톨로지 성숙도 레벨 모델 (L0→L3) | 전략v3 | ★★★ |
| SP-3 | ID 기반 의미 검색 ("ID를 부여하면 벡터 없이 의미 검색") | 전략v3 | ★★★ |
| SP-4 | 엔티티 계층구조 (parent_id 자기참조) | 스키마, 전략v3 | ★★★ |
| SP-5 | aliases 컬럼을 통한 텍스트→ID 정규화 | 전략v3 | ★★★ |
| SP-6 | N:M 관계 테이블 (Junction Table) 패턴 | 스키마 | ★★★ |
| SP-7 | entity_relations (엔티티 간 교차 관계) | 스키마 | ★★☆ |
| SP-8 | 5단계 데이터 파이프라인 + SHACL 검증 | 백서, 전략v3 | ★★☆ |

#### SP-1: FRBR 2계층 모델
- **Biblo**: Work(작품, 314K) → Book/Manifestation(도서, 360K) 1:N 관계
- **KCI 적용**: Article(논문) → Author Record(저자 출현 레코드) 1:N + Author Identity(식별된 저자) 엔티티

#### SP-2: 온톨로지 성숙도 레벨 모델

| 레벨 | Biblo 상태 | KCI 현재 상태 | KCI 목표 |
|------|-----------|-------------|---------|
| L0 | 테이블만 존재 | — | — |
| L1 | 메타데이터 존재 | 저자명·소속 텍스트 존재 | 현재 수준 |
| L1.5 | 데이터 있으나 ID 미부여 | 알고리즘 역할 분류 완료 | Phase 1~3 후 |
| L2 | ID + 계층 + 관계 | 저자 ID, 기관/분야 계층 | **1차 목표** |
| L3 | OWL/SPARQL/LOD | 연구자 그래프, LOD 연계 | 장기 목표 |

#### SP-3~SP-8: 요약
- **SP-3**: "ID를 부여하면 벡터 없이 의미 검색이 된다" → 동명이인 해소의 근본 해법
- **SP-4**: parent_id 자기참조 → 기관·분야 계층 구조
- **SP-5**: aliases[] 컬럼 → 저자명·기관명 변형 매핑
- **SP-6**: N:M Junction 테이블 → 저자-논문-기관-분야 관계
- **SP-7**: entity_relations → 엔티티 간 교차 관계
- **SP-8**: 5단계 파이프라인 + SHACL 검증 → 월별 증분 처리

### 20. 데이터 구조 설계 원칙

| 원칙 | 설명 | 출처 |
|------|------|------|
| 원본 보존 | RIMS 덤프 원본은 변경하지 않고 별도 보존 | Biblo "Raw Preservation" |
| 2계층 분리 | 원본(Raw) ↔ 정제(Refined) 레이어 분리 | SP-1 FRBR |
| ID 중심 설계 | 모든 엔티티에 고유 ID 부여 | SP-3 |
| 계층 구조 | parent_id로 분류 체계 표현 | SP-4 |
| 별칭 지원 | aliases/name_variants로 텍스트→ID 매핑 | SP-5 |
| Junction 관계 | N:M은 반드시 관계 테이블로 표현 | SP-6 |
| 스냅샷 관리 | 월별 덤프를 스냅샷으로 관리, 증분 처리 | 월 1회 요건 |

### 21. 전체 ERD

```
┌─ RAW LAYER (원본 보존) ──────────────────────────────────────┐
│  rims_dump ──1:N──→ rims_article_raw ──1:N──→ rims_author_raw│
│  (덤프 메타)          (논문 원본)              (저자 원본)      │
└──────────────────────────┬────────────────────┬───────────────┘
                           │ ETL               │ ETL
┌─ REFINED LAYER (정제/온톨로지) ──────────────────────────────┐
│  article ◄──── author_article ────► author_identity          │
│  (논문)   1:N   (저자-논문 관계)  N:1   (식별 저자)           │
│    │              - role, rank, confidence     │              │
│    ├─N:M─→ keyword          ├─N:M─→ institution              │
│    └─N:1─→ journal          ├─N:M─→ research_field           │
│                             └─N:M─→ coauthor_network         │
│  entity_relations (엔티티 간 교차 관계)                       │
│  identification_log (식별 이력/감사)                          │
└──────────────────────────────────────────────────────────────┘
```

### 22. 핵심 테이블 명세

#### RAW LAYER

**rims_dump**: id, dump_date, dump_type, file_article, file_author, article_count, author_count, status, created_at, processed_at

**rims_article_raw**: id, dump_id(FK), rims_paper_id, org_lang_ppr_nm, eng_ppr_nm, scjnl_nm, pblc_ym, doi, abst_cntn, kci_yn, raw_json, created_at

**rims_author_raw**: id, dump_id(FK), rims_paper_id, prtcpnt_id, author_name_kr, author_name_en, tpi_dvs_cd, org_nm, dept_nm, raw_json, created_at

#### REFINED LAYER

**article**: id, rims_paper_id(UNIQUE), title_kr, title_en, journal_id(FK), publish_date, doi, abstract, is_kci, author_count, first_dump_id, last_dump_id

**author_identity** (핵심): id, canonical_name_kr, canonical_name_en, name_variants[], prtcpnt_ids[], primary_institution_id(FK), primary_field_id(FK), orcid, article_count, confidence_score, status

**journal**: id, name, aliases[], issn, eissn, publisher, field_id(FK), is_kci

#### 온톨로지 엔티티 (계층구조)

**institution**: id, name, aliases[], parent_id(self FK), institution_type, country
```
성균관대학교 (id=1, parent_id=NULL)
  ├── 공과대학 (id=10, parent_id=1)
  │   └── 소프트웨어학과 (id=101, parent_id=10)
  └── 자연과학대학 (id=20, parent_id=1)
```

**research_field**: id, name, aliases[], parent_id(self FK), field_code, level

**keyword**: id, label, aliases[], parent_id(self FK)

#### 관계 테이블 (N:M Junction)

**author_article**: author_id, article_id, role, author_rank, confidence, source, raw_author_id
**author_institution**: author_id, institution_id, period_start, period_end, is_primary
**author_field**: author_id, field_id, weight, article_count
**article_keyword**: article_id, keyword_id, relevance
**coauthor_network**: author_id_1, author_id_2, collaboration_count, first/last_collaboration
**entity_relations**: source_type, source_id, relation_type, target_type, target_id, metadata

#### 이력 테이블

**identification_log**: dump_id, author_identity_id, raw_author_id, action, algorithm_phase, confidence, signals_used(jsonb), previous_state(jsonb)

### 23. Biblo ↔ KCI 패턴 매핑 총괄

| Biblo 패턴 | Biblo 적용 | KCI 적용 | 비고 |
|-----------|-----------|---------|------|
| FRBR Work | 작품 (314K) | article (논문) | 추상적 지적 단위 |
| FRBR Manifestation | book (360K) | rims_article_raw (원본) | 물리적 발현 |
| — | — | author_identity (저자) | 고유 엔티티 |
| genre | 장르 (13K) | research_field (연구분야) | 분류 체계 |
| concept | 개념 (374K) | keyword (키워드) | 주제어 |
| place | 장소 (80K) | institution (기관) | 소속 기관 |
| work_genres | 작품-장르 (630K) | author_field (저자-분야) | N:M 관계 |
| work_concepts | 작품-개념 (1.5M) | article_keyword (논문-키워드) | N:M 관계 |
| work_places | 작품-장소 (457K) | author_institution (저자-기관) | N:M + 시간 |
| entity_relations | 교차 (326) | entity_relations | 동일 패턴 |

---

## Part VII: 월별 RIMS 덤프 처리 파이프라인

### 24. 전체 흐름

```
매월 1일: RIMS 덤프 수신
     ▼
STEP 1: 덤프 적재
  rims_dump 레코드 생성 → rims_article_raw/rims_author_raw 벌크 INSERT
     ▼
STEP 2: 증분 감지 (Delta Detection)
  신규/변경/삭제 논문 식별 → article INSERT/UPDATE
     ▼
STEP 3: 저자식별 알고리즘 (Phase 1~4)
  수집 → 역할 판별 → 동일성 검증 → 매칭/업데이트
  수학 알고리즘 1차 → LLM 보완 2차 (하이브리드)
     ▼
STEP 4: 온톨로지 갱신
  institution/research_field/keyword 매핑 + coauthor_network 갱신
     ▼
STEP 5: 품질 검증 (SHACL)
  필수 필드·관계 무결성·중복 검사 → 통계 리포트
```

### 25. 증분 처리 전략

| 구분 | 감지 방법 | 처리 |
|------|----------|------|
| 신규 논문 | rims_paper_id NOT IN article | INSERT + 전체 알고리즘 |
| 변경 논문 | raw_json diff 비교 | UPDATE + 변경 필드만 재처리 |
| 삭제 논문 | 이전 덤프에 있고 현재 없음 | soft delete |
| 신규 저자 | 새 논문의 저자 레코드 | 하이브리드 식별 |
| 기존 저자 새 논문 | author_identity 존재 + 신규 article | author_article INSERT |

### 26. 월별 예상 처리량

| 항목 | 월 예상 | 누적 (1년) |
|------|---------|-----------|
| 신규 논문 | ~2,000건 | ~24,000건 |
| 신규 저자 레코드 | ~7,400건 | ~88,800건 |
| 수학 알고리즘 처리 | ~5,200건 (70%) | — |
| LLM 보완 처리 | ~2,200건 (30%) | — |
| 알고리즘 처리 시간 | ~2~4시간 | — |

---

## Part VIII: 기술 스택 및 실행 계획

### 27. 핵심 라이브러리

| 라이브러리 | 용도 | 카테고리 |
|-----------|------|---------|
| pandas | RIMS CSV 데이터 처리 | 데이터 |
| requests / lxml | KCI API/웹 접근 | 수집 |
| PyMuPDF (fitz) | PDF 텍스트 추출 | 추출 |
| pytesseract + Pillow | OCR (이미지 PDF) | 추출 |
| rapidfuzz | 퍼지 문자열 매칭 | 매칭 |
| korean-romanizer | 한글 로마자 변환 | 매칭 |
| scikit-learn | TF-IDF + 코사인 유사도 | 동일성 |
| networkx | 공저자 네트워크 분석 | 동일성 |
| anthropic | Claude API (LLM 프롬프트) | LLM 판별 |
| tqdm | 진행률 표시 | 유틸 |

### 28. 모듈 구조

```
author_classify/
├── config.py                # 설정
├── main.py                  # 파이프라인 오케스트레이터
├── data/
│   ├── rims_loader.py       # RIMS CSV 로드
│   └── output_writer.py     # 결과 출력
├── collectors/
│   ├── kci_api.py           # KCI Open API (Track A)
│   ├── kci_scraper.py       # KCI 웹페이지 스크래핑
│   └── pdf_downloader.py    # PDF 다운로드
├── extractors/
│   ├── api_parser.py        # API XML 파싱
│   ├── pdf_extractor.py     # PDF 텍스트 추출
│   └── author_parser.py     # 저자명/역할 파싱
├── matchers/
│   ├── name_matcher.py      # 5-Level 퍼지 매칭
│   ├── role_classifier.py   # 의사결정 트리
│   └── cross_validator.py   # 교차 검증
├── identity/
│   ├── disambiguator.py     # 동명이인 구분
│   ├── entity_resolver.py   # 동일저자 추적
│   ├── coauthor_network.py  # 공저자 네트워크 분석
│   ├── topic_similarity.py  # 연구 주제 유사도
│   └── temporal_tracker.py  # 시계열 경력 추적
├── llm/                        # 신규
│   ├── prompt_templates.py    # 프롬프트 템플릿
│   ├── llm_classifier.py     # LLM 역할 판별
│   ├── llm_disambiguator.py  # LLM 동명이인 구분
│   └── hybrid_engine.py      # 하이브리드 오케스트레이터
├── ontology/                   # 신규
│   ├── entity_manager.py      # 엔티티 CRUD
│   ├── hierarchy_builder.py   # 계층 구조 빌더
│   ├── alias_mapper.py        # aliases 매핑
│   └── incremental_sync.py    # 월별 증분 동기화
└── utils/
    ├── text_normalizer.py   # 텍스트 정규화
    ├── korean_utils.py      # 한글 처리
    └── rate_limiter.py      # API 속도 제한
```

### 29. 실행 계획 (5-Sprint)

**Sprint 1: 기반 구축 + PDF 경로** (API 키 없이 진행)
1. RIMS 데이터 로더 구현
2. PDF URL 추출 + 텍스트 추출기
3. 저자명/역할 기호 파서
4. 10건 샘플 테스트

**Sprint 2: 매칭 + 역할 판별**
5. 퍼지 매칭 (5-Level)
6. 역할 의사결정 트리
7. 100건 샘플 테스트

**Sprint 3: 저자 동일성 검증**
8. 공저자 네트워크 + TF-IDF 엔진
9. 시계열 경력 추적
10. 5-Signal 종합 모델
11. 동명이인 100건 테스트

**Sprint 4: LLM 통합 + 하이브리드** (신규)
12. LLM 프롬프트 설계 및 테스트
13. 하이브리드 오케스트레이터 구현
14. 수학 vs LLM vs 하이브리드 정확도 비교 실험
15. 최적 임계값 튜닝 (0.7 기준점 조정)

**Sprint 5: KCI API 통합 + 대규모 실행 + 온톨로지**
16. KCI API 키 연동 + 교차 검증
17. 전체 24,532건 배치 실행
18. 온톨로지 엔티티 초기 구축
19. 월별 증분 파이프라인 자동화

---

## Part IX: 리스크, KPI, 결론

### 30. 리스크 및 대응

| 이슈 | 빈도 | 대응 |
|------|------|------|
| API 키 미발급 | — | PDF 경로만으로 우선 진행 |
| PDF 접근 불가 | ~20% | DOI 리다이렉트 → 순서 기반 추정 |
| 이미지 PDF | ~10% | Tesseract OCR + LLM OCR 보정 |
| 저자 기호 비표준 | ~5% | LLM 맥락 기반 파싱 (하이브리드) |
| 한영 저자명 불일치 | ~15% | 5-Level 퍼지 매칭 |
| 동명이인 | ~4.3% | 5-Signal + LLM 하이브리드 |
| 소속 변경자 | ~31% | 시계열 추적 + LLM 판단 |
| PRTCPNT_ID 미보유 | 92.7% | 하이브리드 추정 |
| LLM API 장애 | 드묾 | 수학 알고리즘 단독 Fallback |

### 31. 성공 지표 (KPI)

| 지표 | 목표 |
|------|------|
| 교신저자 식별률 | > 85% |
| 1저자 검증률 | > 95% |
| 저자명 매칭 정확도 | > 90% |
| 동명이인 정확 구분률 | > 85% (하이브리드) |
| 동일저자 추적 정확도 | > 88% (하이브리드) |
| 수동 검토 비율 | < 12% (하이브리드) |
| 전체 처리 시간 | < 72시간 (배치) |

### 32. 기대 효과

| 항목 | AS-IS | TO-BE |
|------|-------|-------|
| 교신저자 데이터 | 3,797건 | 20,000건+ |
| 저자 역할 분류 | 3단계 | 5단계 |
| 동명이인 처리 | 수동/미처리 | 자동 판별 (하이브리드) |
| 소속 변경 추적 | 미지원 | 시계열 자동 추적 |
| 데이터 정확도 | 미검증 | 신뢰도 기반 품질 관리 |
| 저자 온톨로지 | 없음 | ID 기반 엔티티 체계 (L2) |
| 월별 운영 | 수동 | 자동 증분 파이프라인 |

### 33. 온톨로지 성숙도 로드맵

**Phase A (L0→L1)**: 기반 구축 — 1~2개월
- RAW LAYER 테이블 생성, 첫 RIMS 덤프 적재, 증분 감지 로직

**Phase B (L1→L1.5)**: 저자식별 알고리즘 적용 — 2~3개월
- 하이브리드 알고리즘 실행, author_article 관계 구축, identification_log 기록

**Phase C (L1.5→L2)**: 온톨로지 ID 체계 구축 — 2~3개월
- institution/research_field/keyword 엔티티 구축 (계층 + aliases)
- N:M 관계 테이블 + coauthor_network + entity_relations 완성
- **"ID 기반 의미 검색" 달성**

**Phase D (L2→L3)**: 고도화 — 장기
- 벡터 임베딩, OWL 온톨로지, SPARQL, 외부 LOD 연계

### 34. 결론

본 프로젝트는 KCI 논문 저자 식별이라는 핵심 과제를 수학 알고리즘과 LLM 자연어 프롬프트의 **하이브리드 접근법**으로 해결하고, 그 결과를 Biblo 온톨로지 전략을 적용한 체계적인 데이터 구조에 축적한다.

**핵심 결론:**
1. **하이브리드 전략**: 수학 알고리즘(빠르고 저비용)으로 70%를 처리하고, LLM(정확하고 맥락적)으로 30%를 보완 → 비용 대비 최고 정확도
2. **온톨로지 L2 목표**: ID 기반 엔티티 체계를 구축하여 "SQL만으로 연구자 프로필 검색" 달성
3. **월별 자동화**: RIMS 덤프 수신부터 온톨로지 갱신까지 전 과정 자동화
4. **감사 추적**: identification_log로 모든 식별 결정의 근거와 방법 기록

---

## 부록 A: RIMS 데이터 필드 명세

### rims_article_data.csv
| 필드 | 활용 |
|------|------|
| ARTICLE_ID | 내부 조인 키 |
| ORG_LANG_PPR_NM | 논문 제목 → 주제 유사도 |
| ABST_CNTN | 초록 → TF-IDF / LLM 유사도 |
| SCJNL_NM | 학술지명 → 분야 추정 |
| PBLC_YM | 발행연도 → 시계열 추적 |
| DOI / ID_KCI | 외부 데이터 접근 키 |

### rims_article_parti_data.csv
| 필드 | 활용 |
|------|------|
| PRTCPNT_ID | 확정 식별자 |
| PRTCPNT_NM / PRTCPNT_FULL_NM | 이름 매칭 대상 |
| TPI_DVS_CD | 보정 대상 |
| BLNG_AGC_NM | 소속 매칭 + 시계열 추적 |
| ORCID_ID / SCOPUS_ID | 확정 식별자 |
| EMAL_ADDR | 준확정 식별자 |
| ARTICLE_ID | 공저자 네트워크 구축 |

---

*본 보고서는 RIMS 데이터 품질 고도화 프로젝트의 통합 기술 전략 문서입니다.*
*저자 식별 알고리즘 + LLM 비교 분석 + Biblo 온톨로지 전략 적용을 포괄합니다.*
*성균관대학교 | 2026.02*
