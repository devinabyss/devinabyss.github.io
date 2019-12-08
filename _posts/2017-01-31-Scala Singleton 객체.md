---
published: true
layout: single
toc: true
title: Scala Singleton 객체
category: language
tags: scala
comments: true
---

## object 지시어

- 최근 Scala 를 찔끔 찔끔 공부하고 있는데, 공부한지 얼마 안된 상태에서 가장 생소하면서도 재미있는게 object 지시어다.
- Java 만 접해왔던 개발자라면 무의식적으로 객체를 만드는건  ``new Object()`` 아니면 ``getInstance`` 로 머리 속에 박혀져 있기 쉽상인데, 이 오랜 발상을 뒤집어버리고, class 에 묶여 사고를 하다보면 객체는 class 가 있어야 만들어지는 것이 당연하거늘.
- 지시어부터 왠 object ? 의 느낌적인 느낌이랄까.
- 하지만 객체에 대해 알고 있는 개념 그 자체를 그대로 받아들이면 이해와 사용이 빨라진다.
- class 가 객체를 찍어내기 위한 틀인건 맞는데, object 지시어는 그  틀 없이 바로 객체를 선언한 그대로 하나만 만들겠다는 의미다.

즉 싱글턴 객체.

```scala
object President {
	def  doApology(){
		println("It's a misunderstanding. I am not wrong. They are bastards.")
	}
}

```

- 이 같이 object 를 선언하면 앱이 메모리에 로딩되는 순간 바로 President 라는 객체가 생성되며,
- 컴파일 전 코드의 다른 스코프에서 바로 President.doApology() 를 호출하도록 짤 수 있다.
- 주된 용도는 Java 에서 유틸리티 class 를 static 지시어를 사용해 짜던 것을 대체한다.
- 위 코드를 Java 에서 짠다고 한다면 아래와 같을 것이다.

```scala
class President {
	public static final void doApology() {
		System.out.println("It's a misunderstanding. I am not wrong. They are bastards.");
	}
}
```

- 알다싶이 개념적으로 class 는 object 를 찍어내기 위한 틀이다. 
- 하지만 Java 에선 위와 같은 static 지시어를 통해 class 본래의 개념과는 상충된다고 할 수 있는 용도의 코딩이 자주 사용된다.
- 용도는 같다. 
- 하지만 객체로 코딩하는가 클래스를 빙자한 명령어 집합으로 코딩하는가가 Scala 와 Java 의 큰 차이점을 보여주고, 코딩하는 입장에서도 솔직히 깔끔하고 납득이 간다.
- Scala 는 int char long 과 같은 primitive 형도 없고 + - 같은 연산자도 객체라며 보다 객체 친화적이라는 설명이 많은 입문서 도입부에 명시되어 있는데, 개인적으로 객체지향적이라는 관념(?)이 가장 현실적으로 와닿았던 것은 바로 object 지시어였다.

## 싱글턴 객체는 JVM 의 어디에 로드되는가?

- 이 object 지시어에 조금씩 익숙해져갈 때 즈음.. 해서 떠오른 의문 한가지.
- 싱글턴 객체는 JVM 의 어느 메모리 영역에 로드되는가?
- Java 의 static 지시어로 선언된 메소드와 변수들은 모두 메소드 영역에 할당된다. 
- 당연하지만 static 신공으로 범벅된 유틸리티형 클래스 역시 마찬가지.
- 메소드 영역은 실제로는 힙의 일부이긴 하나, GC 가 일어나지 않고 Class 의 구조나 생성자, 메소드와 같이 어디서나 사용될 수 있는 코드들이 담기는 영역이며, Java 의 유틸리티형 클래스가 그 역할을 할 수 있는 것은 소멸하지 않는 메소드 영역에 그 코드가 로드되기에 가능한 일이다.
- 하지만 Scala 의 object 로 선언된 객체는.. 객체인데? 
- Java 의 법칙대로라면 객체는 일반적으로 힙 영역에 보관된다. 그리고 힙 영역에 속한다는 것은 GC 의 대상이 될 수도 있음을 의미한다. 
- 즉, 객체 선언만 해놓고 참조되지 않는 싱글턴 객체는 GC 의 대상일 될 수도 있는 것인데, 
- 그러면 유틸리티 용도의 코드가 의미가 있는 것일까?

[Stackoverflow - When are Scala objects garbage collected?](http://stackoverflow.com/questions/3956652/when-are-scala-objects-garbage-collected%20%20)

- 좀 더 공식 문서의 내용을 찾고 싶은데, 아직은 찾아내지 못했고..

- 위 stackoverflow 링크의 답변을 참조하면, Scala 의 object 로 선언된 객체의 코드는 컴파일러에 의해 바이트코드로 전환되면서 static 맴버로 초기화된다고 한다. static 맴버로 초기화되니 당연히 메서드 영역에 로드될 것이고 GC 의 대상이 되지 않는다.