---
author: "Kim, Geunho"
date: 2013-06-15
title: "Java - 깊은 복사(Deep copy)와 얕은 복사(Shallow copy)"
---


Java에서 `int`, `long`, `float`, `double`과 같은 primitive type은 선언되는 변수 자체에 값을 가지는데, 값을 복사하거나 할당할 때 흔히 다음과 같이 코드를 작성하게 된다.
 
```java
int a = 3;
int b = 1;
a = b;
```

![primitive type copy](/java-deep-copy-shallow-copy-1.png) _그림 1. primitive type의 복사_

너무나 간단한 코드이지만 이를 `Object` 클래스를 상속하는 모든 객체(reference type)에서 그대로 적용하면, reference 값만을 복사하게 되어 같은 참조를 가진 두 개의 변수가 생기게 된다.

![reference type, shallow copy](/java-deep-copy-shallow-copy-2.png) _그림 2. 객체의 얕은 복사_

그림2와 같은 복사를 얕은 복사(Shallow copy)라고 부른다. 변수 `a`와 `b`는 `x100`이라는 같은 reference 값을 가지게 되므로 한 쪽에서 데이터를 변경하면 다른 변수에서도 참조하는 값이 변경되는 것이다.  

참조 값이 아닌 객체의 모든 값을 복사하는 깊은 복사(Deep copy)를 하기 위해서는 `Cloneable` 인터페이스의 `clone()` 메서드를 호출하면 된다.  
사용자가 정의한 클래스라면 `Cloneable` 인터페이스를 implement하여 `Object clone()` 메서드를 override한다.

```java 
public class Route implements Cloneable {
    ArrayList<Integer> path;
    double cost;

    public Route clone() throws CloneNotSupportedException {
        Route route = (Route) super.clone();
        route.path = (ArrayList<Integer>) path.clone();
        route.cost = this.cost;
        return route;
    }
    …
}
```
 
메서드 내에서는 반환할 객체를 새로 생성하여 각 멤버 변수를 다시 `clone()` 메서드로 복제하여 설정해서 반환하고 있다.  
예제 코드의 `path` 멤버 변수는 `ArrayList<Integer>` 형식의 reference type인데, `HashMap`이나 `ArrayList` 같은 컬렉션들은 이미 `clone()` 메서드가 구현되어 있다. 따라서 새로 생성한 `Route` 객체의 `path` 멤버 변수에는 기존 멤버에서 `clone()` 메서드를 호출하여 깊은 복사를 하고, primitive type인 `cost` 멤버 변수는 직접 할당하여 반환하게 된다.  

만약 `path` 멤버 변수가 사용자 정의 클래스라면, `Route` 클래스와 마찬가지로 `Cloneable`로 구현하여 `clone()` 메서드 호출로 복사를 하거나, 클래스 내부의 멤버 변수들을 직접 탐색해서 복사하는 방법도 있을 것이다.

 
```java
public class Route implements Cloneable {
    Path path;  // 사용자 정의 클래스

    public Route clone() throws CloneNotSupportedException {
        route route = (Route) super.clone();
        route.path = (Path) path.clone(); 
        // 단, Path 클래스도 마찬가지로 clone() method가 구현되야 함

        return route;
    }
    …
}
```