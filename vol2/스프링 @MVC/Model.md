# Model

## Map, Model, ModelMap

다른 어노테이션이 붙어있지 않다면 Map, Model, ModelMap 은 모두 모델 정보를 담는 데 사용할 수 있는 오브젝트가 전달된다.

```java
@RequsetMapping("/list")
public String list(ModelMap model) {
  User user = userService.findUser("admin");
  model.addAttribute(user); // model.addAttribute("user", user) 와 동일
  return "list";
}
```

## @ModelAttribute 와 커맨드 객체(Command Object)

- @ModelAttribute 의 기능
  - 모델에 커맨드 객체나, 단순 데이터 타입등을 담아준다.
  - 여러개의 요청 파라미터를 커맨드 객체에 바인딩 시켜준다.
  - @RequestParam 과 달리 검증(Validation) 작업을 추가적으로 진행한다.
- @RequestParam 의 기능
  - 요청 파라미터로 온 값을 메서드 파라미터에 바인딩 시켜준다. (1:1 매핑)
  - 스프링의 기본 타입 변환 기능을 이용해서 요청 파라미터 값을 메서드 파라미터 타입으로 변환한다.


@ModelAttribute 는 메서드 파라미터에도 부여할 수 있고, 메서드 레벨에도 적용할 수 있다. 두 가지가 비슷하지만 사용 목적이 분명히 다르다.
모델 정보를 생성하고 조작하는 것은 컨트롤러의 몫이다.

컨트롤러가 사용하는 모델 중에는 클라이언트로부터 받는 HTTP 요청정보를 이용해 생성되는 것이 있다. 단순히 검색을 위한 파라미터처럼 컨트롤러 로직에서 사용하고 버리는 요청정보도 있지만,
웹 페이지의 폼 정보 처럼 일단 컨트롤러가 전달받아서 내부 로직에 사용하고 필요에 따라 다시 화면에 출력하기도 하는 요청정보도 있다. 이렇게 클라이언트로부터 컨트롤러가 받는 요청정보 중에서,
하나 이상의 값을 가진 오브젝트 형태로 만들 수 있는 구조적인 정보를 @ModelAttribute 모델이라고 부른다. 

즉, @ModelAttribute 는 컨트롤러가 전달받는 오브젝트 형태의 정보를 가리키는 말이다.

> 요청 파라미터 : 쿼리 스트링이라던지, 폼으로 전달되는 파라미터 값들 

```html
<input type="text" name="name" value="Spring" />
```

위 처럼 되어있을 때 `@RequestParam` 을 이용하여 name 파라미터를 받을 수 있다.

단지, 요청 파라미터를 메서드 파라미터에서 1:1 로 받으려면 `@RequestParam` 이고, 도메인 오브젝트나, DTO 프로퍼티에 요청 파라미터를 바인딩해서 한 번에 받으면 `@ModelAttribute` 이다.

view 단에서 요청 파라미터를 통해서 값들이 넘어오면 우리는 `HttpServletRequest` 객체로 request.getParameter("인풋이나, textArea 등 name 에 설정한 이름") 이러한 방식으로 값을 받아올 수 있다.

```java
@GetMapping("/index")
public String index(HttpServletRequest request) {
  String name = request.getParameter("name");
  String age = request.getParameter("age");
  // 생략
}
```

@RequestParam 으로도 받을 수 있다.

```java
@GetMapping("/index")
public String index(@RequestParam String name, @RequestParam int age) {
  String name = name;
  String age = age;
  // 생략
}
```

이렇게 필요한 값들을 받아서 처리해야하는 파라미터 개수가 많아지면, 코드 가독성도 떨어지고 작성해야할 코드 양도 많아진다. 이러한 문제를 해결하기 위한 것이 `커맨드 객체(Command Object)` 이다.

> 커맨드 객체(Command Object) : HttpServletRequest 로 받아오는 요청 파라미터의 key 값과 동일한 이름의 속성들과 setter 메서드를 가지는 VO 혹은 DTO 를 만들고, 만든 VO 객체를 메서드의 파라미터로
설정하면, 요청 파라미터(HttpServletRequest 에 담긴 파라미터)가 자동으로 커맨드 객체에 바인딩이 된다.

```java
@GetMapping("/index")
public String index(BoardVo boardVo) {
  // 생략
}
```

굳이 모델에 담는 것이 아니면 커맨드 객체 앞에 @ModelAttribute 를 작성하지 않아도, 객체가 바인딩 된다. (마치, Jackson2ObjectMapperBuilder 가 객체를 JSON 으로 변환하기 위해서 
autoDetectGettersSetters() 를 이용하는 것과 같음)

@RequestParam 도 @ModelAttribute 처럼 생략해도 요청 파라미터를 받을 수 있다.

스프링은 어떻게 어노테이션이 없는 파라미터를 보고 어떤 것은 @RequestParam 이 생략된거고 어떤 것은 @ModelAttribute 가 생략된 것인지 구분할 수 있는 것일 까?

스프링이 이 둘을 판별하는 기준은, 몇 가지 단순 타입, 예를 들어 String 이나 int 등은 @RequestParam 으로 보고, 그 외의 복잡한 오브젝트들은 @ModelAttribute 가 생략됬다고 간주한다.
하지만, 단순 타입이 아니라고해서 무조건 @ModelAttribute 가 생략됬다고 보는 것은 위험하다. 스프링은 간단한 숫자나 문자로 전달된 요청 파라미터를 제법 복잡한 오브젝트로 변환할 수도 있다.

따라서 무조건 @ModelAttribute 를 생략하는 것은 위험하다. 그래서 가능한 @ModelAttribute 나 @RequestParam 어노테이션을 사용하는 것을 권장한다. 물론 단순한 커맨드 오브젝트나,
누가 보더라도 쉽게 파악할 수 잇는 단순한 요청 파라미터면 생략하는 것도 나쁘지 않다.

@ModelAttribute 가 해주는 기능은 또 한가지가 있는데 바로 모델에 알아서 담아주는 것이다.

```java
@GetMapping("/index")
public String index(@ModelAttribute("searchVo") BoardVo boardVo) {
  // 생략
}
```

위 처럼 model 에 담을 키값을 지정할 수 도 있고, 지정하지않고 @ModelAttribute 만 사용하면 카멜케이스 기법을 적용하여 키값으로 저장한다.

@ModelAttribute 와 커맨드 객체를 같이 사용하면, 해당 객체를 이용하여 view 에서 `객체변수명.속성명` 이런식으로 원하는 속성 값을 쓸 수 있다.

## Erros, BindingResult

@ModelAttribute 는 단지 오브젝트에 여러 개의 요청 파라미터 값을 넣어서 넘겨주는게 전부가 아니다. @ModelAttribute 가 붙은 파라미터를 처리할 때는 @RequstParam 과 달리
`검증(Validation)` 작업이 추가적으로 진행된다.

@RequestParam 은 스프링의 기본 타입 변환 기능을 이용해서 요청 파라미터 값을 메서드 파라미터 타입으로 변환한다. 만약에 숫자 타입의 파라미터라면 스트링 타입으로 들어온 요청 파라미터의 타입 변환을 시도하고
실패하면 HTTP 400 - Bad Request 응답이 클라이언트로 간다.

만약 친절하게 메시지를 보여주고 싶으면 `org.springframework.beans.TypeMismatchException` 예외를 처리하는 핸들러 예외 리졸버를 추가해주면 된다.

그러나 @ModelAttribute 를 사용했을 때는 다르다. 

UserSearch 의 id 프로퍼티가 int 타입인데 /search?id=abcd 라는 URL 요청이 들어오면 어떻게 될까?

```java
public String search(@ModelAttribute UserSearch, BindingResult result) {
  // 생략
}
```

UserSearch 의 setId() 를 이용해서 id 값을 넣으려고 시도하다가 예외를 만나지만 작업이 중단되고 HTTP 400 - Bad Request 응답 상태 코드가 클라이언트로 전달 되지는 않는다. 타입 변환에 실패해도
작업은 계속된다. 단지 타입 변환 중 발생한 예외가 BindingException 타입의 오브젝트에 담겨서 컨트롤러로 전달될 뿐이다.

@ModelAttribute 입장에서는 파라미터 타입이 일치하지 않는다는 건 검증 작업의 한 가지 결과일 뿐이지, 예상치 못한 예외상황이 아니라는 의미이다. 

`사용자가 직접 입력하는 폼에서 들어오는 정보라면 반드시 검증이 필요하다.` 왜냐하면 사용자가 입력하는 값에는 다양한 오류가 있을 수 있기 때문이다. 

검증 작업에는 `타입 확인, 필수 정보 입력 여부, 길이 제한, 포맷, 값의 허용범위` 등 다양한 검증 기준이 적용될 수 있다. 이렇게 검증 과정을 거친 뒤 오류가 발견되었다 하더라도 
HTTP 400 과 같은 예외 응답 상태를 전달하면서 작업을 종료하면 안된다. 어느 웹사이트에 가서 회원가입을 하는 중에 필수 항목을 하나 빼먹었다고 호출스택 정보와 함께 HTTP 400 에러 메시지가
나타나면 얼마나 황당하겠는가?

`그래서 사용자의 입력 값에 오류가 있을 때는 이에 대한 처리를 컨트롤러에게 맡겨야 한다.` 그러려면 메서드 파라미터에 맞게 요청정보를 추출해서 제공해주는 책임을 가진 어댑터 핸들러는 
실패한 변환 작업에 대한 정보를 컨트롤러에게 제공해줄 필요가 있다. 따라서 이러한 정보를 참고해서 적절한 에러 페이지를 출력하고나, 친절한 에러 메시지를 보여주면서
사용자가 폼을 다시 수정할 기회를 줘야 한다.

바로 이 때문에 @ModelAttribute 를 통해 폼의 정보를 전달 받을 때에는 `org.springframework.validation.Errors` 또는 org.springframework.validation.BindingResult` 타입의 파라미터를
같이 사용해야 한다. Errors 나 BindingResult 파리미터를 함께 사용하지 않으면 스프링은 요청 파라미터의 타입이나 값에 문제가 없도록 애플리케이션이 보장해준다고 생각한다. 따라서 같이 사용하지 않으면
HTTP 400 응답 상태 코드로 변환되지도 않으니, 지저분한 에러 메시지를 만나게 될 것이다.

BindingResult 나 Erros 의 파라미터 위치는 반드시 @ModelAttribute 뒤에 나와야 한다. 자신의 바로 앞에 있는 @ModelAttibute 파라미터의 검증 작업에서 발생한 오류만을 전달해 주기 때문이다.


