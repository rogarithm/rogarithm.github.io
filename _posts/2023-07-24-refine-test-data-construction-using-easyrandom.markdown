---
layout: post
title: "EasyRandom으로 테스트 데이터 만들기"
date: 2023-07-24 00:54:58 +0900
categories: testing
---
* table of contents
{:toc}

&nbsp;

### 테스트 코드도 유지보수가 필요하다

이미 구현한 기능이더라도 더 다듬을 부분이 있고, 새로 기능을 만들면서 기존 설계를 바꾸기도 한다. 테스트를 만들어두면 이렇게 코드를 바꿀 때 실수할 위험을 줄여준다. 하지만 제품 코드와 같이 테스트도 다듬어두지 않으면 금새 복잡해져서 이런 장점을 누리기 어려워진다. 그래서 나는 테스트 코드를 정리하고, 어떻게 하면 덜 복잡하게 만들 수 있을지를 고민한다.

그간 여러 방식으로 테스트를 다듬었지만, '더 나은 개선 방안은 없을까' 하는 생각이 머릿속을 멤돌았다. 그러던 중 예전 프로젝트에서 받은 코드 리뷰에서 알게 된 EasyRandom이 떠올랐다.

&nbsp;

### EasyRandom?

![EasyRandom](/images/easy-random/easy-random-icon.png)

EasyRandom은 테스트 데이터 생성에 쓸 수 있는 라이브러리로, 데이터로 만들려는 타입이 갖는 필드의 갯수나 종류에 상관없이 필드마다 임의의 값을 넣어 인스턴스를 만들어준다. 테스트 데이터를 만들 때 필드마다 값을 집어넣을 필요가 없어진다는 얘기다.

이제까지 모든 테스트 데이터를 하드코딩해서 만들었는데, 필드마다 어떤 값을 넣어야할지 고민하고, 테스트마다 테스트 데이터를 만들어야 하는 점이 불편하게 느껴졌다. 기존 방식을 EasyRandom으로 바꾸면 이런 점을 개선할 수 있지 않을까 싶었다.

&nbsp;

&nbsp;

### 테스트에서 하드코딩을 쓰지 않아도 괜찮을까?

EasyRandom을 도입하기 전에, 기존 방식을 대신할 수 있는지 확인해보기로 했다. 그래서 기존에 만들어져 있던 테스트에서 테스트 데이터를 어떻게 쓰고 있는지, 생성 방식을 바꿔도 괜찮은지 알아보기로 했다.

&nbsp;

#### 단위 테스트의 경우

기존에 테스트 데이터를 만드는 방식은 하드코딩이었다. 테스트 데이터를 만들 때, 데이터의 각 필드마다 그 필드에 들어갈만한 값을 생각해서 집어넣었다. 만약 테스트 데이터에 (필드의 타입만 만족하는) 임의의 값이 들어가면 문제가 될까? 아니다. 그렇더라도 테스트를 성공하게 할 수 있다.

단위 테스트에선 테스트하고자 하는 클래스가 주변 환경으로부터 격리된다.

![단위 테스트의 객체 구성](/images/easy-random/unit-test-object-relation.png)

실제 의존 객체가 아니라, 어떻게 동작할지 직접 정의한 모의 객체가 테스트에 참여한다. 모의 객체의 동작을 정의할 때, 어떤 데이터를 받았을 때 원하는 동작을 하게 할지 정할 수 있다. 그렇기 때문에 '필드에 들어갈만한 값'이 아니더라도 확인하려는 동작이 발생하도록 할 수 있다.

또한 하드코딩한 값이 실제 환경을 완벽히 반영한다고 보기 힘들다. 예를 들어, 만약 테스트용 DB와 서비스용 DB가 분리되어 있어서 두 DB에 저장된 데이터가 서로 다르다면, 하드코딩한 값이더라도 실제 환경에선 다르게 동작하는 값일 수 있다.

이런 점들을 고려해보니 단위 테스트는 테스트 데이터에 어떤 값이 들어갈지 고려하지 않고, 테스트 대상 객체의 동작 확인에 신경쓰도록 바꿔도 괜찮겠다고 결론내렸다.

&nbsp;

#### 통합 테스트의 경우

통합 테스트는 실제 의존 객체가 잘 협력하는지를 확인한다.

![통합 테스트의 객체 구성](/images/easy-random/integration-test-object-relation.png)

통합 테스트에서는 실제 환경과 연결해서 실행하므로 테스트 데이터의 필드값은 의미가 있는 값이어야 한다. 여기서 의미가 있다는 건, 이를테면 도메인 모델의 각 필드에 정의한 유효성 조건을 만족하는 걸 뜻한다. 만약 테스트 데이터에 (필드의 타입만 만족하는) 임의의 값이 들어가면 문제가 될까? 그렇다. 테스트 데이터를 임의의 값으로 만들 경우, 데이터 값이 실제 환경에 설정한 조건에 맞지 않으면 테스트가 실패할 수 있다. 따라서 테스트 데이터의 값은 테스트로 확인하고자 하는 상황을 일으키는 값이어야 한다.

그렇다면 통합 테스트에서는 EasyRandom을 쓰지 않고 하드코딩으로 테스트 데이터를 만드는 것이 나을까? 나는 그렇지 않다고 생각한다. 왜냐면 하드코딩해야 할 필드 외에 임의값을 가져도 상관없는 필드도 있으며, 이런 필드값을 만드는 것도 EasyRandom이 도와줄 수 있기 때문이다. 하드코딩할 필드도 일정한 범위나 논리를 만족하는 값을 필요로 하고, 이런 조건을 EasyRandom을 써서 매개변수화할 수 있다. 따라서 통합 테스트도 EasyRandom 방식으로 바꿀 수 있다.

&nbsp;

&nbsp;

### 어떻게 적용할까?

각 상황마다 EasyRandom을 어떻게 적용하는지를 예시로 들겠다. 두 예시 모두에서 사용할 DTO는 다음과 같이 정의된다(생성자 등 이외 메서드 정의는 생략했다).

```java
@Getter
@Builder
public class AddTodoRequest {

    @NotBlank(message = "할일 제목이 입력되지 않았습니다")
    private String name;
    private String description;
}
```

#### 특정 값일 필요 없는 테스트 데이터 하드코딩

서비스 레이어의 메서드를 테스트하는 상황을 예로 들겠다.

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

EasyRandom 인스턴스를 하나 만들고, 만들고 싶은 데이터의 타입을 인자로 해서 nextObject() 메서드를 호출하면 해당 타입의 각 필드별로 타입에 맞는 값이 채워진 테스트 데이터를 생성해준다. 하드코딩으로 값을 설정해주던 코드를 이 코드로 바꿔준다.

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

#### 특정 값이어야 하는 테스트 데이터 하드코딩

Todo API를 통합 테스트하는 상황을 예로 들겠다.

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

스프링 애플리케이션 컨텍스트를 띄우는 통합 테스트는 실제 의존 객체를 가져오기 때문에, 앞에서와 같이 테스트 데이터를 만들면 필드값이 확인하고자 하는 조건에 맞는 것을 보장할 수 없기 때문에 테스트가 실패할 수 있다.
EasyRandom을 쓰면서 테스트 데이터의 값이 테스트로 확인하고자 하는 상황을 일으키는 값임을 보장하려면 EasyRandom에서 제공하는 EasyRandomParameter 타입 객체를 이용해야 한다. EasyRandomParameter 객체는 1) 어떤 필드에 2) 어떤 조건을 갖는 값을 반환하게 할 것인지를 입력해 만들 수 있다. 이 중 값을 반환하게 하는 Randomizer<T> 타입은 함수형 인터페이스이기 때문에 람다식으로 조금 더 간소화할 수 있다.

```java
    @Test
    public void addTodoFailWhenRequestNotSatisfyConstraint() {

        Predicate<Field> name = FieldPredicates.named("name")
            .and(FieldPredicates.ofType(String.class))
            .and(FieldPredicates.inClass(AddTodoRequest.class));
        EasyRandomParameters invalidName = new EasyRandomParameters()
            .randomize(name, () -> "");

        EasyRandom generator = new EasyRandom(invalidName);
        AddTodoRequest request = generator.nextObject(AddTodoRequest.class);

        assertThrows(ConstraintViolationException.class,
            () -> todoController.addTodo(addTodoRequest));
    }
```

&nbsp;

### 결론

- 테스트 데이터 이름을 지을 필요가 줄어들었다
- 검증 로직을 단순화할 수 있게 되었다

?
- 테스트 데이터에 어떤 값을 넣어야할지 고민하지 않아도 된다
- 코드가 줄어들어 테스트 읽는 게 수월해졌다

