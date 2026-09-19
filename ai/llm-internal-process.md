# LLM 내부 동작 6단계

> 작성일: 2026-09-19

- 참조 : [LLM은 어떻게 동작하는가 - 카카오페이 기술블로그](https://tech.kakaopay.com/post/how-llm-works/#3%EB%8B%A8%EA%B3%84-positional-encoding)
- 1단계: Tokenization	입력 문장을 의미 있는 최소 단위로 분리하고, 각 토큰에 고유한 정수 ID를 할당
- 2단계: Embedding 토큰 ID를 의미와 문맥을 담은 고차원 벡터로 변환
- 3단계: Positional Encoding 각 단어의 순서 정보를 벡터에 더해 위치를 인식
- 4단계: Transformer & Attention 문맥을 파악하고, 각 단어의 표현을 정교하게 다듬기
- 5단계: Prediction 다음에 올 토큰의 확률을 계산하고, 최적의 토큰을 선택
- 6단계: Loop & Decoding	예측된 토큰을 반복적으로 입력에 추가해 문장을 완성
