---
title: Effective Java - [item31] 한정적 와일드카드를 사용해 API 우연성을 높이라 - PECS 원칙
author: mingo
date: 2023-11-03 00:01:00 +0900
categories: [Effective Java]
tags: [effective java, pecs]
---

----

# 5장 - [아이템 21] - 한정적 와일드카드를 사용해 API 우연성을 높이라

## 한줄 요약
> 

## 한정적 와일드카드를 사용해 API 우연성을 높이라
매개변수화 타입은 불공변(invariant)이다. **즉, `A` 타입이 `B` 타입의 하위 타입일 때 `List<A>` 타입은 `List<B>` 타입의 하위 타입이 아니다.**

