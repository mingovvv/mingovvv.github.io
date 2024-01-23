---
title: OpenAI frequency_penalty, presence_penalty 튜닝하기 
author: mingo
date: 2024-01-22 00:30:00 +0900
categories: [OpenAI]
tags: [frequency_penalty, presence_penalty]
#image:
#  path: /path/to/image
#  alt: image alternative text
---

-----------------
![44.png](/assets/img/post/202401/44.png){: .border w="800" h="600" }
_OpenAI_

LLM(Large Language Model)의 한 종류인 OpenAI의 GPT 모델을 사용한 프로젝트를 진행하게 되어 파라미터를 튜닝하던 중 `frequency_penalty`와 `presence_penalty`가 굉장히 모호하고 비슷하게 느껴져 정리해보았다.     
우선 공식 다큐먼트를 보고 각각의 파라미터가 어떤 역할을 하는지 알아보자.

## 1. frequency_penalty
![42.png](/assets/img/post/202401/42.png){: .border w="800" h="150" }
_frequency_penalty api reference_

문서에 따르면 `frequency_penalty`는 -2.0 ~ 2.0 사이의 정수값을 가지며, 
양수 값은 지금까지 생성된 텍스트의 빈도횟수를 기반으로 새로운 토큰을 생성하는 것에 불이익을 주어 LLM 모델이 동일한 라인을 반복할 가능성을 줄인다.

## 2. presence_penalty
![43.png](/assets/img/post/202401/43.png){: .border w="800" h="150" }
_presence_penalty api reference_

이어서 `presence_penalty`는 -2.0 ~ 2.0 사이의 정수값을 가지며, 
양수 값은 지금까지 생성된 텍스트의 생성여부에 따라 새로운 토크능ㄹ 생성하는 것에 불이익을 주어 새로운 주제에 대해서 이야기할 가능성을 높인다.

## frequency_penalty vs presence_penalty
처음 공식 다큐먼트만 읽다보면 확 와닿지 않는다. 그러던 중 구글링을 통해 꽤나 설명을 자세하게 해준 외국 블로그를 찾았고 아래 내용은 해당 블로그의 글을 인용해서 작성하였다.

`frequency_penalty`는 동일한 단어를 자주 사용하지 않도록 도와주는 파라미터이며 컴퓨터에게 "동일한 단어를 너무 자주 말하지마!" 라고 말하는 것과 같다.
반면 `presence_penalty`는 다양한 단어의 사용을 권장하는 파라미터로써 "같은 단어만 쓰지말고 다양한 단어를 사용해!" 라고 부탁하는 것이다.

**여기서 중요한 핵심은 타켓팅된 토큰(LLM 텍스트의 단위)에 대하여 
`frequency_penalty`는 페널티를 중복해서 적용할 수 있고 `presence_penalty`는 최초 한 번만 적용된다는 점이다.**

아직 이 말이 무슨 의미인지 와닿지 않을 수 있다.

그럴 땐, "frequency_penalty는 빈도수에 따라 페널티를 적용하겠다", "presence_penalty는 존재여부에 따라 페널티를 적용하겠다." 이런 사전적 의미를 기반으로 접근해도 이해하기 좋다.

좀 더 깊게 설명하자면, 우리는 먼저 `logits`라는 개념을 알아야 한다. 
logits는 LLM 모델이 생성한 텍스트의 각각의 단위인 "토큰"에 대한 점수라고 이해하자.
이 logits 점수가 직접적인 확률은 아니지만 확률을 계산하는 베이스가 되므로 logits가 높을수록 다음 생성될 텍스트에서 해당 토큰이 선택되어 생성될 확률이 높다고 이해하면 된다.

예를 들어 "치즈로 만든" 이라는 텍스트 뒤에 생성될 말은 "피자", "푸딩" 등 다양한 토큰이 존재할 것이고 각각의 토큰은 logits 점수를 가지고 있다는 것이다.
"피자"의 logits 점수가 100, "푸딩"의 logits 점수가 20이라면 "치즈로 만든" 뒤에 "피자"가 생성될 확률 높다.

이제 frequency_penalty 수치를 0.2로 현재 "피자"라는 토큰의 logits 100점이라고 가정하자.
LLM 모델은 이전 대화 히스토리에서 "피자"라는 토큰을 아래와 같이 이미 2번 사용했다면?

```text
피자는 맛있어.
내가 제일 좋아하는 것은 피자야.
치즈로 만든
```
"피자" 토큰은 2회 등장했다.
그 다음 생성될 토큰은 여러가지가 있겠지만, "피자"라는 토큰은 페널티로 인하여 100점에서 0.2 * 2(빈도수 2회) = 0.4점이 감소한 99.6점의 logits 점수를 기반으로 샘플링 될 것이다.
이전 텍스트에서 생성된 토큰 빈도수에 따라 logits 점수가 배로 감소할 것이며 이것은 샘플링을 통해 생성될 토큰의 가능성을 현저하게 낮출 것이다.

반면, presence_penalty는 logits 점수에 페널티를 적용할 때 토큰 존재여부에 따라 최초 한 번만 적용된다.

```text
피자는 맛있어.
내가 제일 좋아하는 것은 피자야.
치즈로 만든
```
동일한 예제를 가지고 이해하자. 역시 "피자" 토큰은 2회 등장했다.
이번에는 presence_penalty 수치를 0.2로 현재 "피자"라는 토큰의 logits 100점이라고 가정하자.

"피자"라는 토큰은 페널티로 인하여 100점에서 0.2점이 감소한 99.6점의 logits 점수를 기반으로 샘플링 될 것이다. 빈도수와 관계없이 등장여부가 중요하다는 점이다.
"피자"라는 단어가 계속 등장해도 역시 logits 점수는 99.6점으로 유지된다.

## penalty 정리 및 튜닝
"피자" 토큰의 최종 logits 점수와 파라미터는
```text
"피자" logits 점수 = 기존 "피자" logits 점수 - (frequency_penalty * "피자" 토큰 빈도수 + presence_penalty)
``` 

| 파라미터             | 범위         | 기본값 | 특징                       |
|:-----------------|:-----------|-----|--------------------------|
| frequency_penalty       | -2.0 ~ 2.0 | 0   | 수치가 높을수록 같은 단어가 등장하지 않음. |
| presence_penalty    | -2.0 ~ 2.0 | 0   | 수치가 높을수록 다양한 언어가 등장함. |

이렇게 정리할 수 있을 것 같다.

> 출처
- Document : <https://platform.openai.com/docs/guides/text-generation/parameter-details>
- Document : <https://platform.openai.com/docs/api-reference/chat/create#chat-create-frequency_penalty>
- Document : <https://platform.openai.com/docs/api-reference/chat/create#chat-create-presence_penalty>
- Frequency vs Presence penalty, what’s the difference? : <https://medium.com/@KTAsim/frequency-vs-presence-penalty-whats-the-difference-openai-api-51b0c4a7229e>
