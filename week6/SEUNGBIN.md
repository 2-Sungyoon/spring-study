# 📅 [week 6] 주제: Can Large Language Models Understand Structured Table Data?

## 🎯 학습 목표
- LLM이 테이블의 2차원 구조를 얼마나 이해하는지, 어떻게 이해하는지 파악한다.
- LLM의 성능을 극대화할 수 있는 **최적의 테이블 입력 포맷**을 찾아낸다.
- 테이블 구조 이해를 돕기 위한 **Self Augmented**한 방법을 익힌다.
---

## 📝 주요 개념 정리

## LLM은 정말 테이블을 '이해'하고 있을까?

LLM은 텍스트를 한 줄로 읽는 데 특화되어 있어 위아래로 얽힌 테이블 구조는 어려워한다.

수평(Row) vs 수직(Column): 가로 줄(행)은 비교적 잘 읽지만, 세로 줄(열)을 따라 데이터를 찾는 건 훨씬 어려워한다.

### 1. 직렬화 Serialization

LLM은 기본적으로 '텍스트'를 입력받는 모델입니다. 따라서 2차원 형태의 표를 한 줄의 긴 텍스트로 펼치는 과정이 필요합니다. 이를 **직렬화**라고 합니다.

- **Markdown 방식:** 가장 흔한 방식입니다. `|`와 를 사용해 행과 열을 구분합니다.
- **HTML/JSON 방식:** `<table>`, `<tr>`, `<td>` 태그나 키-값 쌍을 통해 구조를 명시합니다.
- **단순 텍스트:** "이름은 김철수, 나이는 25세..."와 같이 자연어로 풀어서 입력하기도 합니다.

### 2. 위치 및 관계 학습

LLM은 **토큰 간의 거리와 문맥**을 계산합니다.

- **헤더와 셀의 연결:** 모델은 특정 셀의 데이터가 어떤 '헤더(column)' 아래에 있는지, 그리고 같은 row에 있는 다른 데이터와 어떤 관계인지 통계적으로 파악합니다.
- **Attention 메커니즘:** 질문이 들어오면 모델은 표의 수많은 데이터 중 질문과 관련된 특정 행과 열에 Attention을 집중시킵니다.

### 3. 추론과 연산

표를 읽은 후 LLM은 내부에 학습된 지식을 바탕으로 다음 작업을 수행합니다.

- **패턴 매칭:** "판매량이 가장 높은 달은?"이라는 질문을 받으면, 숫자 크기를 비교하는 패턴을 활성화합니다.
- **논리적 결합:** 서로 다른 열에 있는 데이터를 조합해 새로운 정보를 도출합니다
    - 예: 가격과 수량을 곱해 총액 계산.
## 2차원의 구조를 모델이 어떻게 학습할 것인가?

1. 대규모의 table 데이터를 이용하여 **table에 특화된 사전 학습**을 수행하자!

<img width="1190" height="417" alt="Image" src="https://github.com/user-attachments/assets/44549461-cf6a-4a82-9a64-8a5efab7b3ee" />

**Table Serialization의 정의**

이미지 상단에 명시된 것처럼, **Table Serialization**은 2차원 형태의 표(table) 데이터를 모델이 읽을 수 있도록 선형적이고 순차적인 텍스트(linear, sequential text)로 변환하는 과정입니다.

**table → linear, sequential text**

이미지 속의 변환 과정을 보면 아주 직관적입니다.

- **원본 데이터:** 이름, 연구실, 학기, 생일 정보가 담긴 행과 열의 구조입니다.
- **변환 결과:** `이름 연구실 학기 생일 | 마민정 DSBA 1 3월 4일 | 이지윤 DSBA 2 9월 23일 ...`
- **특징:** 각 행을 구분하기 위해 `|` 기호를 사용하고, 별다른 수식어 없이 데이터 값만 순서대로 나열했습니다.

### **TAPAS(Table Parsing)** 모델

<img width="1189" height="415" alt="Image" src="https://github.com/user-attachments/assets/9d06f64e-6f8f-49fa-965d-11e853bdc06b" />

**1. Simple Linearization**
왼쪽의 2x2 표를 한 줄의 텍스트 토큰으로 펼치는 과정입니다.
• **Token Embeddings**: 표의 내용(col1, col2, 0, 1, 2, 3)과 질문(query)을 일렬로 나열합니다. `[CLS]`는 문장의 시작, `[SEP]`는 질문과 표 데이터 사이의 구분자 역할을 합니다.

**2. Multi-layered Embeddings**
텍스트만 나열하면 AI는 어떤 숫자가 몇 행 몇 열에 있는지 알 수 없습니다. 

이를 해결하기 위해 여러 종류의 위치 정보(Metadata)를 합칩니다.

- **Position Embeddings**: 전체 문장에서 토큰의 절대적인 순서 ($POS_0, POS_1, ...$)를 나타냅니다.
- **Segment Embeddings**: 질문 부분($SEG_0$)과 표 데이터 부분($SEG_1$)을 구분합니다.
- **Column Embeddings**: 해당 토큰이 몇 번째 열에 속하는지 나타냅니다 ($COL_1$은 첫 번째 열, $COL_2$는 두 번째 열).
- **Row Embeddings**: 해당 토큰이 몇 번째 행에 속하는지 나타냅니다 ($ROW_0$은 헤더, $ROW_1$은 첫 번째 데이터 행 등).
- **Rank Embeddings**: 열 내에서 값의 순위나 순서를 나타내는 추가적인 수치 정보입니다.
****

### **TAPEX(Table Pre-training via Execution)** 모델

<img width="1192" height="416" alt="Image" src="https://github.com/user-attachments/assets/6a21a6ee-e93e-40d4-ba81-b75c6015ccd0" />

1. Flattening with Special Tokens

왼쪽의 표 데이터를 모델이 읽을 수 있는 한 줄의 문장으로 변환할 때, 데이터 간의 경계를 명확히 하기 위해 특수 토큰을 사용합니다.

- **`[HEAD]`**: 표의 열 제목(Header)이 시작됨을 알립니다. (예: `[HEAD] Contestant | Age | Hometown`)
- **`[ROW]`**: 표의 새로운 행(Row)이 시작됨을 알립니다. 행 번호와 함께 쓰여 구조를 명확히 합니다. (예: `[ROW] 1 Reyna Royo ...`)
- **`|` (파이프 기호)**: 각 셀(Cell) 데이터 사이를 구분하는 구분자 역할을 합니다.

2. Fine-tuning

1. 합성 데이터 생성

모델을 학습시키려면 엄청나게 많은 표와 질문 데이터가 필요합니다. 하지만 사람이 직접 만든 데이터는 한계가 잇죠. 그래서 TAPEX는 다음과 같은 방식을 씁니다.

- **표 수집:** 위키피디아 등에서 수백만 개의 표를 가져옵니다.
- **SQL 생성:** 이 표들에 대해 실행 가능한 **SQL 쿼리를 무작위로 생성**합니다. (예: `SELECT Name WHERE Age > 30`, `SELECT COUNT(*) WHERE ...`)
- **결과 추출:** 생성된 SQL을 실제 SQL 엔진(Executor)으로 실행하여 정답을 도출합니다.

b. pre training

이제 모델(BART 기반)에게 다음과 같은 입력을 줍니다.

- **입력(Input):** [Flattened Table(직렬화된 표)] + [합성된 SQL 쿼리]
- **출력(Target):** [SQL 실행 결과(Answer)]

여기서 중요한 점은 모델이 **SQL 문법 자체를 배우는 것이 아니라, SQL이 수행하는 '논리'를 배운다는 것**입니다.

- 예: `SELECT MAX(Score)`라는 쿼리가 들어오면, 모델은 표 데이터 중 해당 열에서 가장 큰 숫자를 찾아내는 '비교 연산' 능력을 기르게 됩니다.

c. 왜 이렇게 하나요?

기존 모델(TaPas 등)은 표 안의 단어들 사이의 관계만 파악하려고 했습니다. 하지만 TAPEX는 표를 완벽하게 이해했다면, 그 표에 대한 계산 결과도 맞힐 수 있어야 한다고 가정합니다.

- **논리적 추론:** '가장 나이가 많은 사람', '합계', '평균' 같은 연산 능력을 사전 학습 단계에서 미리 갖추게 됩니다.
- **일반화:** 이렇게 SQL 실행으로 단련된 모델은 나중에 사람이 자연어로 묻는 질문(Downstream Task)에도 훨씬 더 정확하게 답할 수 있습니다.

d. Fine tuning

사전 학습이 끝나면, 이제 진짜 사람이 만든 데이터셋으로 미세 조정을 합니다.

- **입력:** [표] + [자연어 질문 (예: "24살인 다른 사람은 누구니?")]
- **학습:** 모델은 사전 학습 때 배운 '비교/추출/연산' 능력을 활용해 자연어 질문에 대한 답을 찾아냅니다.

### 템플릿

<img width="1189" height="416" alt="Image" src="https://github.com/user-attachments/assets/7c0ba9b3-64cf-4090-a5b4-15404eb8d68b" />

1. 작동 원리

- **규칙 설정**: "The [행 이름] of [열 이름] is [값]"과 같은 문장 틀을 미리 만듭니다.
- **변환**: 표의 각 셀 데이터를 이 틀에 집어넣어 문장을 생성합니다.
    - 예: (Aerospace Systems, 2017, 9560) 데이터 → "The funded Aerospace Systems in 2017 was 9560."

2. 이 방식의 특징

- llm이 별도의 특수 학습 없이도 일반 문장을 읽듯이 표 내용을 이해할 수 있습니다.
- 표의 구조를 복잡하게 임베딩하는 것보다(linear serialization)
    
    관계를 명확한 문장으로 설명하기 때문에 모델이 데이터 간의 연결 고리를 더 쉽게 파악하기도 합니다.
    
- 하지만 표의 종류가 바뀔 때마 다 사람이 일일이 템플릿을 새로 짜줘야 한다는 번거로움이 있습니다.

1. LLM이 이미 가지고 있는 지식을 이용하자!

<img width="1188" height="418" alt="Image" src="https://github.com/user-attachments/assets/852df8aa-3d1b-40f3-b810-33ffec8ac2d7" />

**Demonstration**

모델에게 "어떻게 문제를 풀어야 하는지" 알려주는 예제입니다. 'Washington Redskins' 드래프트 기록 표와 그에 대한 질문, 그리고 생각 과정(Thoughts)이 포함되어 있습니다.

**Test Case**

모델이 실제로 풀어야 할 표 데이터입니다. 1919년 브라질 축구 경기 결과와 득점자 정보가 담겨 있습니다.

**Chain of Thoughts : 추론 과정**

모델이 단순히 답만 내놓는 것이 아니고 "Neco가 5월 11일에 2골, 5월 26일에 2골을 넣었으므로 총 4골이다"라는 **단계별 논리**를 거쳐 정답을 도출하는 방식입니다.

이미지 우측의 녹색 박스를 보면 모델이 정답인 `4`를 맞추기 위해 데이터를 어떻게 필터링하고 합산했는지 논리적으로 설명하고 있습니다.

**작동 방식:** 표 형식의 데이터를 텍스트로 입력받아, 특정 조건(선수 이름: Neco, 대회명: American Championship)에 맞는 데이터를 찾아 산술 연산을 수행합니다.


<img width="1190" height="415" alt="Image" src="https://github.com/user-attachments/assets/01019bda-57dc-4eca-b77e-eb6d8deb0538" />
<img width="1189" height="416" alt="Image" src="https://github.com/user-attachments/assets/ff8b54ab-a25f-40e7-96a9-e66dd3066180" />
<img width="1191" height="417" alt="Image" src="https://github.com/user-attachments/assets/377911db-b996-46aa-944b-b4d8f982b12c" />
<img width="1192" height="415" alt="Image" src="https://github.com/user-attachments/assets/bffc0cfe-abc4-4298-92f9-9ecc8ff765c3" />
<img width="1188" height="416" alt="Image" src="https://github.com/user-attachments/assets/ad4ce7de-e1ae-41ce-a68d-3eddc751f1d5" />

| **방식** | **특징** | **예시 문장** | **기술 / 모델** |
| --- | --- | --- | --- |
| **Manual Template** | 미리 정해진 틀에 데이터(값)만 끼워 넣는 방식입니다. | "The age is 42. The education is Master." | `x is x` 형식의 Rule-based |
| **Table-To-Text** | 템플릿보다 조금 더 문장답게 만들지만, 여전히 정형화된 패턴을 따릅니다. | "The person is 42 years old. She has a Master." | Small specialized models (T5, BART 등) |
| **LLM (Serialized)** | 거대 언어 모델이 직접 문맥과 문법을 고려해 가장 자연스럽고 풍부하게 서술합니다. | "The person is 42 years old and has a Master's degree. She gained $594." | GPT-4, Gemini, Llama 등 |

## 💻 코드 실습 및 분석



## 💡 회고 및 질의응답
- 
