---
title: '_class에 대하여'
date: 2024-05-19T17:35:26+09:00
draft: true
tags: ["MongoDB", "JPA"]
categories: ["development"]
---

# _class 대체 왜 사용하는 것일까?

spring-data-mongo를 사용하다보면 은연중에 _class가 도큐먼트에 등록되는 상황을 심심치않게 볼 수 있다. 이는 MongDB가 가지는 스키마리스의 성질을 보완하고자 spring-data-mongo에서 기본적으로 사용하는 옵션이다.

_class 필드는 ORM 매핑시 다형성을 활용하여 객체를 도큐먼트와 매핑하는 상황에서 사용된다. 먼저 다형성을 이용해서 도큐먼트를 매핑(설계)한다는 의미부터 집고 넘어가자.

## 도큐먼트와 다형성
다형성(polymorphism)은 프로그램 언어에서 한 객체가 여러 타입의 형태로 사용할 수 있는 경우를 의미하는데, 이를 활용하면 특정 인터페이스를 상속받은 

