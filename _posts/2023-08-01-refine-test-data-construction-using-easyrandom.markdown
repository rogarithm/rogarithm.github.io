---
layout: post
title: "EasyRandom으로 테스트 데이터 만들기"
date: 2023-08-01 15:59:58 +0900
categories: testing
---
* table of contents
{:toc}

&nbsp;

### 테스트 코드도 유지보수가 필요하다

프로그램을 만들다보면 이미 구현한 기능이더라도 더 다듬을만한 곳이 보인다. 쓰임에 더 맞는 변수 이름, 예외 상황을 더 잘 나타내는 메시지, 연산을 요청하기에 더 어울리는 객체 등. 코드를 다듬으면 코드를 더 잘 이해하게 되고, 요구사항이 바뀌어서 코드를 변경해야 하더라도 어떻게 바꿔야할지 생각해내기 수월해진다.

| ![NeedRefactorOrNot](/images/easy-random/do-we-need-to-refactor.png) |
|:----------------------------------------------------------------------------:|
|               다듬지 않은 채로 코드를 방치한 최악의 결과[^xkcd]               |

&nbsp;

코드를 다듬다보면 실수하기도 하지만, 꼼꼼하게 만들어둔 테스트가 있다면 실수할 위험을 줄여준다. 나는 제품 코드와 마찬가지로 테스트 코드를 다듬는다. 프로그램이 커질수록 테스트 코드도 늘어나고, 테스트 코드도 제품 코드와 같이 다듬어두지 않으면 금새 복잡해지기 때문이다. 그간 여러 방식으로 테스트를 다듬었지만, '더 나은 개선 방안은 없을까' 하는 생각이 머릿속을 멤돌았다. 그러던 중 예전 코드 리뷰에서 알게 된 EasyRandom이 떠올랐다.

&nbsp;

&nbsp;

### EasyRandom이란?

![EasyRandom](/images/easy-random/easy-random-icon.png)

EasyRandom은 테스트 데이터 생성에 쓸 수 있는 라이브러리로, 데이터로 만들려는 타입이 갖는 필드의 갯수나 종류에 상관없이 필드마다 임의의 값을 넣어 인스턴스를 만들어준다. **테스트 데이터를 만들 때 필드마다 값을 집어넣을 필요가 없어진다**는 얘기다.

이제까지 모든 테스트 데이터를 하드코딩해서 만들었는데, 필드마다 어떤 값을 넣어야할지 고민하고, 테스트마다 테스트 데이터를 만들어야 하는 점이 불편하게 느껴졌다. 기존 방식을 EasyRandom으로 바꾸면 이런 점을 개선할 수 있지 않을까 싶었다.

&nbsp;

&nbsp;

### 테스트에서 하드코딩을 쓰지 않아도 괜찮을까?

EasyRandom을 도입하면 테스트 데이터 생성 방식이 바뀌므로 기존 코드에 큰 변화가 예상되었다. 무작정 바꾸기보다는 테스트마다 적용 가능한지, 가능하다면 어떤 식으로 적용해야 할지 고려해보기로 했다.

&nbsp;

#### 단위 테스트의 경우

기존의 하드코딩으로 테스트 데이터를 생성하는 방식에서는 데이터의 각 필드마다 들어갈만한 값을 집어넣었다. 만약 테스트 데이터에 (필드의 타입만 만족하는) 임의의 값이 들어가면 문제가 될까? 아니다. 그렇더라도 테스트를 성공하게 할 수 있다.

단위 테스트에선 테스트하고자 하는 클래스가 주변 환경으로부터 격리된다. 모의 객체는 실제 의존 객체를 대신해 테스트에 참여한다. 

![단위 테스트의 객체 구성](/images/easy-random/unit-test-object-relation.png)

&nbsp;

이때 프로그래머는 모의 객체가 어떻게 동작할지 (어떤 데이터를 받았을 때 원하는 동작을 하게 할지) 정의한다. 이렇게 모의 객체가 동작하도록 할 데이터를 직접 정할 수 있기 때문에, '필드에 들어갈만한 값'이 아니더라도 확인하려는 동작이 발생하도록 할 수 있다. 예를 들어, 타입만 맞는 값이면 예상 동작을 하도록 모의 객체를 정의할 경우, **타입이 맞는 어떤 입력이 들어와도 테스트를 통과**시킬 수 있다.

또한 하드코딩한 값이 실제 환경을 완벽히 반영한다고 보기 힘들다. 예를 들어, 만약 테스트용 DB와 서비스용 DB가 분리되어 있고 두 DB에 저장된 데이터가 서로 다르다면, 하드코딩한 값이더라도 어떤 환경과는 다른 결과를 낼 수 있다.

이런 점들을 고려해보니 단위 테스트의 테스트 데이터를 하드코딩 대신 EasyRandom로 만들어도 되겠다고 결론내렸다.

&nbsp;

#### 통합 테스트의 경우

통합 테스트는 실제 의존 객체가 잘 협력하는지를 확인한다.

![통합 테스트의 객체 구성](/images/easy-random/integration-test-object-relation.png)

&nbsp;

통합 테스트에서는 실제 환경과 연결해서 실행하므로 테스트 데이터의 필드값은 의미가 있는 값이어야 한다. 예를 들어 도메인 모델의 어떤 필드에 빈 값이 들어오면 안되는 유효성 조건이 있다면, 이 조건을 만족하지 않는 테스트 데이터일 경우 테스트가 실패할 것이다 (유효성이 실패하는 상황을 테스트하는 게 아닌 이상).

만약 통합 테스트에서 테스트 데이터에 임의의 값이 들어가면 문제가 될까? 그렇다. 예시로 든 것과 같은 상황에서 데이터 값이 실제 환경에 설정한 조건에 맞지 않으면 테스트가 실패할 수 있기 때문이다. 따라서 통합 테스트에서 테스트 데이터의 값은 **테스트로 확인하고자 하는 상황을 일으키는 값이어야** 한다.

그렇다면 통합 테스트에서는 EasyRandom을 쓰지 않고 하드코딩으로 테스트 데이터를 만드는 것이 나을까? 나는 그렇지 않다고 생각한다. 왜냐면 하드코딩해야 할 필드 외에 임의값을 가져도 상관없는 필드도 있으며, 이럴 때 EasyRandom을 써먹을 수 있기 때문이다. 또한 하드코딩해야 하는 필드값도 EasyRandom으로 특정 조건을 만족하는 값만 만들도록 바꿀 수 있다.

이런 점들을 고려해보니 통합 테스트에서도 테스트 데이터를 하드코딩 대신 EasyRandom로 만들어도 되겠다고 결론내렸다.

&nbsp;

&nbsp;

### 어떻게 적용할까?

이제 EasyRandom을 어떻게 적용할지 알아보자. 간단한 예시로, 할 일 관리 백엔드 API에서 새로 구현 중인 할 일 추가 API에 대한 테스트에 EasyRandom을 적용해보겠다.

테스트 데이터로 만들 타입 정의는 다음과 같다 (생성자 등 메서드 정의는 생략했다). 이 중 name 필드는 빈 문자열이면 안되는 유효성 조건이 있다.
```java
@Getter
@Builder
public class AddTodoRequest {

    @NotBlank(message = "할일 제목이 입력되지 않았습니다")
    private String name;
    private String description;
}
```

&nbsp;

#### 테스트 데이터가 특정 값일 필요 없는 경우

다음 단위 테스트는 서비스 레이어의 동작을 테스트하고, 하드코딩으로 테스트 데이터를 만들고 있다.

```java
@Test
public void saveTodoSuccessWhenRequestIsValid() {

    //given
    AddTodoRequest addTodoRequest = AddTodoRequest.builder()
        .name("물 사기")
        .description("집 앞 슈퍼에서 물 사오기")
        .build();

    //when
    when(todoMapper.insertTodo(any(Todo.class))).thenReturn(1);

    //then
    todoService.saveTodo(addTodoRequest);
    verify(todoMapper).insertTodo(any(Todo.class));
}
```

&nbsp;

EasyRandom을 어떻게 적용할까?
1. EasyRandom 인스턴스를 하나 만들고
	```java
	EasyRandom generator = new EasyRandom();
	```
2. 만들려는 데이터의 타입을 인자로 nextObject() 메서드를 호출하면 해당 타입의 각 필드별로 타입에 맞는 값이 채워진 테스트 데이터를 생성해준다.
	```java
	Type testData = generator.nextObject(Type.class);
	```

기존 테스트에 이렇게 적용할 수 있다. 기존 테스트와 달리 addTodoRequest를 만들 때 값을 하드코딩하지 않는 것을 확인할 수 있다.

```java
@Test
public void saveTodoSuccessWhenRequestIsValid() {

    //given
    EasyRandom generator = new EasyRandom();
    AddTodoRequest addTodoRequest = generator.nextObject(AddTodoRequest.class);

    //when
    when(todoMapper.insertTodo(any(Todo.class))).thenReturn(1);

    //then
    todoService.saveTodo(addTodoRequest);
    verify(todoMapper).insertTodo(any(Todo.class));
}
```

&nbsp;

#### 테스트 데이터가 특정 값이어야 하는 경우

다음 통합 테스트는 name 필드값이 빈 문자열일 때 유효성 체크가 실제로 일어나는지를 확인한다. 이전 예와 마찬가지로 하드코딩으로 테스트 데이터를 만들고 있다.

```java
@Test
public void addTodoFailWhenRequestNotSatisfyConstraint() {

    AddTodoRequest addTodoRequest = AddTodoRequest.builder()
        .name("")
        .description("집 앞 슈퍼에서 물 사오기")
        .build();

    assertThrows(ConstraintViolationException.class,
        () -> todoController.addTodo(addTodoRequest));
}
```

&nbsp;

통합 테스트는 실제 의존 객체를 가져온다. 이전과 같이 EasyRandom으로 테스트 데이터를 만들면 필드값이 원하는 조건을 만족하지 않을 수 있다. 그러면 테스트가 실패할 수 있다.

이런 경우 EasyRandomParameter 타입 객체를 이용할 수 있다. 이 객체로 생성할 필드값의 조건을 설정할 수 있다.

- 먼저 EasyRandomParameter 객체를 만든다. 이 객체는 두 가지 정보를 입력해 만든다. 하나는 
필드의 이름, 타입, 필드가 속한 클래스 등의 조건을 적용할 필드에 대한 정보이고, 다른 하나는 생성할 데이터의 조건이다. 둘을 인자로 EasyRandomParameters 객체를 만든다.
```java
EasyRandomParameters parameters = new EasyRandomParameters()
    .randomize(
        FieldPredicates.named("fieldName")
            .and(FieldPredicates.ofType(FieldType.class))
            .and(FieldPredicates.inClass(Type.class)),
        new Randomizer<String>() {
            @Override
            public String getRandomValue() {
                return "";
            }
        }
    );
```
- 만들어진 EasyRandomParameter 객체를 EasyRandom을 만들 때 인자로 넣는다.
	```java
  EasyRandom generator = new EasyRandom(invalidName);
	```
- 만들어진 EasyRandom 객체는 설정된 조건에 맞는 값만 생성한다.
	```java
  Type testData = generator.nextObject(Type.class);
	```

기존 테스트에 이렇게 적용할 수 있다. 값을 하드코딩하지 않는 것을 확인할 수 있다. 이제 addTodoRequest의 description 필드는 여전히 임의의 값으로 생성되지만, name 필드의 값은 항상 빈 문자열만 갖게 된다.

```java
@Test
public void addTodoFailWhenRequestNotSatisfyConstraint() {

    EasyRandomParameters invalidName = new EasyRandomParameters()
        .randomize(
            FieldPredicates.named("name")
                .and(FieldPredicates.ofType(String.class))
                .and(FieldPredicates.inClass(AddTodoRequest.class)),
            new Randomizer<String>() {
                @Override
                public String getRandomValue() {
                    return "";
                }
            }
        );

    EasyRandom generator = new EasyRandom(invalidName);
    AddTodoRequest request = generator.nextObject(AddTodoRequest.class);

    assertThrows(ConstraintViolationException.class,
        () -> todoController.addTodo(addTodoRequest));
}
```

&nbsp;

&nbsp;

### 결론

EasyRandom을 테스트에 적용 후

- 테스트 데이터에 어떤 값을 넣어야할지 고민을 덜 하게 되었다.
- 테스트 데이터를 만드는 코드가 삭제되어서 테스트 읽는 게 수월해졌다.
- 통합 테스트에서는 조건에 맞는 값을 반환하도록 EasyRandomParameter를 구현해야 해서 오히려 코드가 늘어났다. 다만 여러 군데서 쓰게 될 경우 더 효율적일 것 같다.
- 어떤 부분은 코드가 늘어나기도 했고, 어떤 부분은 줄어들기도 했다. 적용 후 추가로 구현하는 기능에서도 지속적으로 적용해봤을 때 도입한 것이 괜찮은 선택이었는지 더 잘 알 수 있을 것 같다.

&nbsp;

---
[^xkcd]: 원본은 [여기](https://xkcd.com/292/)서 확인할 수 있다.
