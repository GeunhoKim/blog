---
author: "Kim, Geunho"
date: 2019-09-25
title: "Vuejs 정리 1 - 인스턴스와 컴포넌트"
---


# 서두
회사에서 Vuejs 교육을 지원해줘서, 좋은 기회로 Vuejs 전문 강사이신 장기효님의 강의를 수강하게 되었다.  
백엔드를 주로 해왔던 개발자 입장에서 수강했던 내용을 다시 정리해보려고 한다.  
Vuejs에 대해 가장 잘 정리된 문서는 [Vuejs 공식 홈페이지의 가이드](https://vuejs.org/v2/guide/)이다. 빠르게 훑어볼 수 있도록 축약하고 보조 자료들을 추가해서 정리한 장기효님의 [Cracking Vue.js](https://joshua1988.github.io/vue-camp/)도 추천한다.  

# 인스턴스
Vuejs를 사용하기 위해서는 가장 먼저 `Vue` 인스턴스를 생성해야 한다. 간단한 테스트를 위해 하나의 html 파일을 생성하고 다음과 같이 내용을 입력한다.  

```html
hello-vuejs.html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Hello Vuejs</title>
</head>
<body>
    <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
</body>
</html>
```

body 태그 내에 Vuejs CDN 스크립트 파일을 불러오고 있다. 스크립트가 브라우저에 로드되면 `Vue` 인스턴스를 생성해서 Vu 애플리케이션을 개발할 수 있다.  
div 태그와 script 태그를 추가로 작성해보자.  

```html
<div id="app"></div>

<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
<script>
    new Vue({
        el: '#app',
        data: {
            str: 'hi Vue!'
        }
    })
</script>
```

`new` 예약어로 `Vue` 인스턴스를 생성했다. `new Vue(instanceOptions)` 형식으로 `instanceOptions`는 미리 정의된 속성을 갖는 생성자 파라미터이다.  
`Vue`는 생성자 함수로, 객체지향 언어에서 클래스를 정의하듯이, 함수를 이용해 클래스를 정의할 수 있다.  
(ES6 부터는 `class` 예약어로 클래스 정의가 가능한데, 이것도 결국 함수이다. 더 자세한 내용은 [Classes - MDN](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Classes)문서를 참고한다.)

Vuejs의 구현 코드를 보면 아래와 같이 생성자 함수가 정의되어 있다. 
```js
function Vue (options) {
  this._init(options)
}
```
Vuejs 2부터는 TypeScript(ts)로 구현되어 있어서 이해를 위해 1.1 브랜치의 코드를 가져왔다. (참고: https://github.com/vuejs/vue/blob/1.1/src/instance/vue.js#L26-L28)  

다시 본론으로 돌아와서, `new Vue(instanceOptions)`의 `instanceOptions`을 살펴보자. 
```js
var instanceOptions = {
        el: '#app', // element의 약자. 인스턴스를 바인딩할 html 태그를 지정한다.
        data: { // 인스턴스의 데이터
            str: 'hi Vue!'
        }
}

new Vue(instanceOptions); 
```
`el` 속성은 바인딩할 태그를 지정하는 것으로 id가 `app`인 html 태그에 바인딩 하겠다는 의미이다. 이 예제에서는 `<div id="app"></div>`에 인스턴스가 바인딩된다.  
즉, 해당 태그 밖의 일에는 방금 생성한 인스턴스가 관여할 수 없고, 오직 바인딩된 태그 내에서만 DOM을 조작하게 된다.  

`data` 속성은 이름 그대로 인스턴스가 가지고 있게될 데이터를 정의한다. `data`에 정의한 속성은 html에 리스팅하기 위한 객체 배열을 가지고 있거나, input 태그의 값을 가지고 있거나, 또는 style 속성에서 사용할 값까지 정의할 수 있다.  

이 외에도 인스턴스의 라이프사이클에 끼워넣을 수 있는 훅을 정의하거나, 다음에 살펴볼 컴포넌트와 템플릿도 등록할 수 있다. 
공식 문서의 [The Vue Instance](https://vuejs.org/v2/guide/instance.html)의 예제와 설명을 살펴보면 쉽게 이해할 수 있다.  

이제 div 태그에 데이터를 불러오고 메서드를 실행하는 코드를 작성해보자.  
```html
<div id="app">
  <p>{{ str }}</p>
</div>
```
태그 내에 콧수염 괄호로 데이터를 감싸면 Vue가 `data` 속성에 정의된 값을 가져온다. 이제 크롬 브라우저에서 파일을 열어보자. 다른 브라우저도 상관 없지만, 크롬이나 파이어폭스의 확장 프로그램 중 Vue 개발자 도구를 설치히면 페이지 동작 중 인스턴스에서 일어나는 모든 일들을 쉽게 살펴볼 수 있다. (각 브라우저의 웹 스토어에서 "Vue.js devtools"를 검색한다. star 수가 많은 것을 선택)

브라우저의 개발자 도구를 열고 Vue 패널로 이동하면 그림 1과 같은 화면을 볼 수 있다.  
![Vue devtools](/vue-instance-and-components-1.png) _그림 1. Chrome, Vue.js devtools_

`<Root>`은 최상위 컴포넌트, 즉 인스턴스를 가리킨다. 마우스를 가져다가 대면 바인딩했던 div 태그가 파란 박스로 `<Root>` 표시가 된다.  
인스턴스의 `data` 속성에 정의했던 `str`도 보인다. 마우스를 가져가면 수정할 수 있는 버튼이 나타나고 값을 변경할 수 있다. 값을 변경하는 즉시 페이지의 내용도 바뀌는 것을 확인할 수 있다.

이제 인스턴스를 변수에 할당해서 직접 살펴보자. `new Vue(instanceOptions)` 코드를 다음과 같이 수정한다.  
```js
var vm = new Vue({
        el: '#app',
        data: {
            str: 'hi Vue!'
        }
    });
```
`var` 예약어는 변수를 선언하는데 쓰이는데, 여기에서는 global scope이 되어 어디서든 접근할 수 있다.  
다시 개발자 도구를 열고 Console 창에서 확인해보자.  

![Vue instance](/vue-instance-and-components-2.png) _그림 2. Vue 인스턴스_

`instanceOptions`의 속성으로 추가했던 `el`, `data`가 `$`기호가 prefix로 추가되어 보이고, 그 외에도 `$`기호로 시작하는 속성들이 보인다.  
`$data`는 특히, 추가한 각 항목들이 인스턴스의 최상위 속성으로 노출된다는 점도 눈이 띈다.  
최상위 속성으로 노출되기 때문에 인스턴스의 scope 내에서는 `str`로 바로 접근이 가능하고 (여기에서는 콧수염괄호로 감쌌던 `str`) `this.str` 처럼 `this` 키워드를 통해 접근할 수 도 있다.  
(`{{ str }}`, `{{ this.str }}`, `{{ this.$data.str }}` 모두 같은 `str`을 가리킨다)

<br/>

# 컴포넌트
컴포넌트는 화면의 구성 요소들을 하나의 묶음 단위로 관리하기 위한 기능이다. 하나의 컴포넌트는 여러개의 하위 컴포넌트를 가질 수 있다.  
도식화 하면 트리 구조가 되는데, 상위 컴포넌트가 가지고 있는 데이터는 리프 노드가 되는 하위 컴포넌트로만 전달할 수 있다. 즉, 데이터는 단방향으로만 흐르는 것이다.  
이러한 제약 덕분에 컴포넌트를 사용하면 다음과 같은 이점들이 있다.  

* 코드 재사용에 유리하다.  
* 화면 구성 요소가 복잡해질 수록 디버깅에 유리하다.  

특히 두 번째 이점은 데이터가 단방향으로만 흐르기 때문에 부각되는 점인데, 하나의 컴포넌트가 데이터 변경의 영향을 받는 것은 오직 부모가 되는 상위 컴포넌트이기 때문이다.  

![Vue components](/vue-instance-and-components-2.png) _그림 3. Vue 컴포넌트_

그림 3에서 나타낸것 처럼, 상위 컴포넌트의 `data` 속성은 하위 컴포넌트의 `props` 읽기전용 속성으로 전달할 수 있다. 상위 컴포넌트의 `data`가 변경되면, 하위 컴포넌트의 `props`도 변경된다. `props`는 읽기전용이므로 변경할 수 없다.  

이제 컴포넌트를 어떻게 등록하는지 살펴보자.  

```js
Vue.component('my-comp', {
    props: ['str'],
    template: '<p>{{ str }}</p>'
})
```

`Vue.component('component name', componentOptions)`을 호출하면 전역 컴포넌트로 등록할 수 있다. 즉, 한번 선언해서 등록해두면 어디서든 사용할 수 있는 것이다.  

컴포넌트 등록 후 html 태그를 다음과 같이 입력하면 Vue가 등록된 컴포넌트의 내용으로 변환해서 DOM에 등록하게 된다.  
```html
<my-comp str="parent component data"></my-comp>
```


