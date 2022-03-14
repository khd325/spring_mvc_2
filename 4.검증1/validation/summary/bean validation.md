# Bean Validation

---

## Bean Validation - 소개

검증 기능을 매번 코드로 작성하는 것은 매우 번거롭다.

특정 필드에 대한 검증 로직은 대부분 빈 값인지, 특점 범위를 넘는지 같은 매우 일반적인 로직이다.

Bean Validation은 특정한 구현체가 아니라 Bean Validation 2.0이라는 기술 표준이다.

쉽게 말하면 검증 애노테이션과 여러 인터페이스의 모음이다.

일반적으로 사용하는 구현체는 하이버네이트 Validator이다.

---

## Bean validation - 시작

---

### 의존관계 추가

`build.gradle`에 `spring-boot-starter-validation`의존관계를 추가하면 된다.

`implementation 'org.springframework.boot:spring-boot-starter-validation'`


### Bean Validation 애노테이션 적용

```java
@Data
public class Item {

    private Long id;
    @NotBlank
    private String itemName;

    @NotNull
    @Range(min = 1000, max = 1000000)
    private Integer price;

    @NotNull
    @Max(9999)
    private Integer quantity;

    public Item() {
    }

    public Item(String itemName, Integer price, Integer quantity) {
        this.itemName = itemName;
        this.price = price;
        this.quantity = quantity;
    }
}
```

BeanValidationTest

```java
public class BeanValidationTest {

    @Test
    void beanValidation(){
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        Validator validator = factory.getValidator();

        Item item = new Item();
        item.setItemName(" ");
        item.setPrice(0);
        item.setQuantity(10000);

        Set<ConstraintViolation<Item>> violations = validator.validate(item);

        for (ConstraintViolation<Item> violation : violations) {
            System.out.println("violation = " + violation);
            System.out.println("violation.getMessage() = " + violation.getMessage());
        }
    }
}

```

```text
ValidatorFactory factory = Validation.buildDefaultValidatorFactory();`
Validator validator = factory.getValidator();
```

검증기 생성 코드

스프링과 통합하면 직접 작성하지 않아도 된다.

`Set<ConstraintViolation<Item>> violations = validator.validate(item);`

검증기에 `item`을 넣고 그 결과를 반환 받는다. 

비어있으면 오류가 없다는 뜻이다.

---

## Bean Validation - 스프링 적용

스프링 부트는 `spring-boot-starter-validation` 라이브러리를 넣으면 자동으로 Bean Validator를 스프링에 통합한다.

그리고 `LocalValidatorFactoryBean`을 글로벌 Validator로 등록한다. 

이 Validator는 `@NotNull`같은 애노테이션을 보고 검정을 수행한다. 글로벌 Validator가 적용되어 있기 때문에 `@Valid`,`@Validated`만 적용하면 된다.

그러면 검증오류가 발생했을 때 `FieldError`,`ObjectError`를 생성해서 `BindingResult`에 담아준다.


> 참고로 직접 글로벌 Validator를 직접 등록하면 스프링 부트는 Bean Validator를 글로벌 Validator로 등록하지 않는다.

### 검증 순서

1. `@ModelAttribute` 각각의 필드에 타입 변환 시도
   1. 성공하면 다음으로
   2. 실패하면 `typeMismatch`로 `FieldError`추가
2. Validator 적용

ex) 

+ `itemName`에 문자"A" 입력 -> 타입 변환 성공 -> `itemName` 필드에 BeanValidation 적용
+ `price`에 문자"A" 입력 -> 바인딩 실패 -> typeMismatch FieldError 추가 -> `price` 필드는 BeanValidation 적용X


---

## Bean Validation - 에러 코드

`itemName` 필드를 공백으로 요청하면 로그에 찍히는 codes는 다음과 같다.

`NotBlank.item.itemName,NotBlank.itemName,NotBlank.java.lang.String,NotBlank`

`MessageCodesResolver`가 다양한 메시지 코드를 순서대로 생성한다.

이를 이용하여 메시지를 등록할 수 있다.

errors.properties 추가

```text
NotBlank={0} 공백X
Range={0}, {2} ~ {1} 허용
Max={0}, 최대 {1}
```

더 자세하게 사용하고 싶다면 `NotBlank.item.itemName= ~~~`처럼 사용할 수 있다.

`{0}`은 필드 명이고 `{1}`,`{2}`는 각 애노테이션마다 다르다.

`@NotBlank(message = "XX")`은 기본 메시지를 지정할 수 있는데

BeanValidation은

1. 생성된 메시지 코드 순서대로 `messageSource`에서 메시지를 찾는다.
2. 애노테이션의 `message` 속성 사용
3. 라이브러리가 제공하는 기본 값을 사용 ex) 공백일 수 없습니다.

---

## Bean Validation - 오브젝트 오류

오브젝트 오류의 경우는 `@ScriptAssert()`를 사용하면 된다.

---

```java

@Data
@ScriptAssert(lang = "javascript", script = "_this.price * _this.quantity >= 10000", message = "총합이 10000원 넘게 입력해주세요")
public class Item {

    private Long id;
    @NotBlank
    private String itemName;

    @NotNull
    @Range(min = 1000, max = 1000000)
    private Integer price;

    @NotNull
    @Max(9999)
    private Integer quantity;

    public Item() {
    }

    public Item(String itemName, Integer price, Integer quantity) {
        this.itemName = itemName;
        this.price = price;
        this.quantity = quantity;
    }
}
```

`@ScriptAssert(lang = "javascript", script = "_this.price * _this.quantity >= 10000", message = "총합이 10000원 넘게 입력해주세요")`

메시지 코드도 `ScriptAssert.item`, `ScriptAssert`가 생성이 된다.

그러나 실무에서는 오브젝트 오류의 경우 직접 자바 코드로 작성하는 것이 권장된다.

```java
@PostMapping("/add")
    public String addItem(@Validated @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {

        if (item.getPrice() != null && item.getQuantity() != null) {
            int resultPrice = item.getPrice() * item.getQuantity();

            if (resultPrice < 10000) {
                bindingResult.reject("totalPriceMin",new Object[]{10000,resultPrice},null);
            }
        }

        if (bindingResult.hasErrors()) {
            log.info("errors = {}", bindingResult);
            return "validation/v3/addForm";
        }

        Item savedItem = itemRepository.save(item);
        redirectAttributes.addAttribute("itemId", savedItem.getId());
        redirectAttributes.addAttribute("status", true);
        return "redirect:/validation/v3/items/{itemId}";
    }
```

---

## Bean Validation - 수정에 적용


```java
    @PostMapping("/{itemId}/edit")
    public String edit(@PathVariable Long itemId,@Validated @ModelAttribute Item item, BindingResult bindingResult) {
        if (item.getPrice() != null && item.getQuantity() != null) {
            int resultPrice = item.getPrice() * item.getQuantity();

            if (resultPrice < 10000) {
                bindingResult.reject("totalPriceMin",new Object[]{10000,resultPrice},null);
            }
        }

        if(bindingResult.hasErrors()){
            log.info("errors = {}",bindingResult);
            return "validation/v3/editForm";
        }

        itemRepository.update(itemId, item);
        return "redirect:/validation/v3/items/{itemId}";
    }
```

`public String edit(@PathVariable Long itemId,@Validated @ModelAttribute Item item, BindingResult bindingResult)`

`Item` 객체에 `@Validated`추가

검증오류가 발생하면 editForm으로 이동

```html
<!DOCTYPE HTML>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="utf-8">
    <link th:href="@{/css/bootstrap.min.css}"
          href="../css/bootstrap.min.css" rel="stylesheet">
    <style>
        .container {
            max-width: 560px;
        }
        .field-error {
            border-color: #dc3545;
            color: #dc3545;
        }
    </style>
</head>
<body>

<div class="container">

    <div class="py-5 text-center">
        <h2 th:text="#{page.updateItem}">상품 수정</h2>
    </div>

    <form action="item.html" th:action th:object="${item}" method="post">

        <div th:if="${#fields.hasGlobalErrors()}">
            <p class="field-error" th:each="err : ${#fields.globalErrors()}" th:text="${err}">글로벌 오류 메시지</p>
        </div>

        <div>
            <label for="id" th:text="#{label.item.id}">상품 ID</label>
            <input type="text" id="id" th:field="*{id}" class="form-control" readonly>
        </div>
        <div>
            <label for="itemName" th:text="#{label.item.itemName}">상품명</label>
            <input type="text" id="itemName" th:field="*{itemName}" th:errorclass="field-error" class="form-control">
            <div class="field-error" th:errors="*{itemName}">
                상품명 오류
            </div>
        </div>
        <div>
            <label for="price" th:text="#{label.item.price}">가격</label>
            <input type="text" id="price" th:field="*{price}" class="form-control" th:errorclass="field-error">
            <div class="field-error" th:errors="*{price}">
                가격 오류
            </div>
        </div>
        <div>
            <label for="quantity" th:text="#{label.item.quantity}">수량</label>
            <input type="text" id="quantity" th:field="*{quantity}" class="form-control" th:errorclass="field-error">
            <div class="field-error" th:errors="*{quantity}">
                수량 오류
            </div>
        </div>

        <hr class="my-4">

        <div class="row">
            <div class="col">
                <button class="w-100 btn btn-primary btn-lg" type="submit" th:text="#{button.save}">저장</button>
            </div>
            <div class="col">
                <button class="w-100 btn btn-secondary btn-lg"
                        onclick="location.href='item.html'"
                        th:onclick="|location.href='@{/validation/v3/items/{itemId}(itemId=${item.id})}'|"
                        type="button" th:text="#{button.cancel}">취소</button>
            </div>
        </div>

    </form>

</div> <!-- /container -->
</body>
</html>
```

addForm과 동일하게 변경

---

## Bean Validation의 한계

---

등록과 수정시 요구사항이 다를 수 있다.

### 수정시 요구사항

+ 등록시에는 `quantity` 수량을 최대 9999이지만 수정시에는 무제한으로 가능
+ 등록시에는 id에 값이 없어도 되지만 수정시에는 id값이 필수


```java
@Data
public class Item {

    @NotNull
    private Long id;
    
    @NotBlank
    private String itemName;

    @NotNull
    @Range(min = 1000, max = 1000000)
    private Integer price;

    @NotNull
    //@Max(9999)
    private Integer quantity;

    public Item() {
    }

    public Item(String itemName, Integer price, Integer quantity) {
        this.itemName = itemName;
        this.price = price;
        this.quantity = quantity;
    }
}
```

수정 요구사항에 맞게 `@NotNull`을 추가하고 `@Max` 애노테이션을 제거

수정은 정상적으로 동작하지만 등록할땐 저장이 되지 않는다.

`ield error in object 'item' on field 'id': rejected value [null]`

등록폼에서는 id의 값이 넘어가지 않는다. 그렇기 때문에 `@NotNull`에서 검증이 실패한다.

결과적으로 `item`은 등록과 수정 에서 검증 조건 충돌이 발생한다.

---

## Bean Validation - groups

동일한 모델 객체를 등록할 때와 수정할 때 다르게 검증하는 방법

1. BeanValidation의 groups 기능 사용
2. item을 직접 사용하지 않고, ItemSaveForm, ItemUpdateForm 같은 폼 전송을 위한 별도의 모델 객체를 만들어서 사용

---

### groups 사용

검증에 사용할 기능을 그룹으로 나누어 적용할 수 있다.

```java
public interface UpdateCheck { }
```

```java
public interface SaveCheck { }
```

인터페이스만 만들어두고 Item 객체를 수정한다.

```java
@Data
public class Item {

    @NotNull(groups = {UpdateCheck.class})
    private Long id;

    @NotBlank(groups = {SaveCheck.class, UpdateCheck.class})
    private String itemName;

    @NotNull(groups = {SaveCheck.class, UpdateCheck.class})
    @Range(min = 1000, max = 1000000, groups = {SaveCheck.class, UpdateCheck.class})
    private Integer price;

    @NotNull(groups = {SaveCheck.class, UpdateCheck.class})
    @Max(value = 9999,groups = {SaveCheck.class})
    private Integer quantity;

    public Item() {
    }

    public Item(String itemName, Integer price, Integer quantity) {
        this.itemName = itemName;
        this.price = price;
        this.quantity = quantity;
    }
}
```

`groups()`기능을 사용해서 검증에 적용할 인터페이스를 지정한다.

```java
    @PostMapping("/add")
    public String addItemV2(@Validated(SaveCheck.class) @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {

        if (item.getPrice() != null && item.getQuantity() != null) {
            int resultPrice = item.getPrice() * item.getQuantity();

            if (resultPrice < 10000) {
                bindingResult.reject("totalPriceMin",new Object[]{10000,resultPrice},null);
            }
        }

        if (bindingResult.hasErrors()) {
            log.info("errors = {}", bindingResult);
            return "validation/v3/addForm";
        }

        Item savedItem = itemRepository.save(item);
        redirectAttributes.addAttribute("itemId", savedItem.getId());
        redirectAttributes.addAttribute("status", true);
        return "redirect:/validation/v3/items/{itemId}";
    }
```

`@Validated`의 파라미터로 인터페이스 클래스를 넣어주면 검증 시시에 다른 조건으로 검증이 가능하다.

하지만 실제로는 groups기능은 전반적인 복잡도가 올라가서 잘 사용하지 않그ㅗ 등록용 폼 객체와 수정용 폼 객체를 분리해서 사용한다.

---

## Form 전송 객체 분리

`Item`객체는 검증에 사용되지 않으므로 검증 애노테이션들 제거

```java
@Data
public class Item {

    //@NotNull(groups = {UpdateCheck.class})
    private Long id;

   // @NotBlank(groups = {SaveCheck.class, UpdateCheck.class})
    private String itemName;

    //@NotNull(groups = {SaveCheck.class, UpdateCheck.class})
    //@Range(min = 1000, max = 1000000, groups = {SaveCheck.class, UpdateCheck.class})
    private Integer price;

    //@NotNull(groups = {SaveCheck.class, UpdateCheck.class})
    //@Max(value = 9999,groups = {SaveCheck.class})
    private Integer quantity;

    public Item() {
    }

    public Item(String itemName, Integer price, Integer quantity) {
        this.itemName = itemName;
        this.price = price;
        this.quantity = quantity;
    }
}
```

ItemSaveForm

```java
@Data
public class ItemSaveForm {

    @NotBlank
    private String itemName;

    @NotNull
    @Range(min = 1000, max = 1000000)
    private Integer price;

    @NotNull
    @Max(9999)
    private Integer quantity;
}
```


ItemUpdateForm

```java
@Data
public class ItemUpdateForm {

    @NotNull
    private Long id;

    @NotBlank
    private String itemName;

    @NotNull
    @Range(min = 1000, max = 1000000)
    private Integer price;
    
    private Integer quantity;

}
```

```java
    @PostMapping("/add")
    public String addItem(@Validated @ModelAttribute("item") ItemSaveForm form, BindingResult bindingResult, RedirectAttributes redirectAttributes) {

        if (form.getPrice() != null && form.getQuantity() != null) {
            int resultPrice = form.getPrice() * form.getQuantity();

            if (resultPrice < 10000) {
                bindingResult.reject("totalPriceMin",new Object[]{10000,resultPrice},null);
            }
        }

        if (bindingResult.hasErrors()) {
            log.info("errors = {}", bindingResult);
            return "validation/v4/addForm";
        }

        Item item = new Item();
        item.setItemName(form.getItemName());
        item.setPrice(form.getPrice());
        item.setQuantity(form.getQuantity());

        Item savedItem = itemRepository.save(item);
        redirectAttributes.addAttribute("itemId", savedItem.getId());
        redirectAttributes.addAttribute("status", true);
        return "redirect:/validation/v4/items/{itemId}";
    }
```

`@ModelAttribute`의 객체를 ItemSaveForm으로 변경한다.

이때 name을 지정하지 않으면 `itemSaveForm`으로 저장돼서 기존 템플릿 뷰의 `Object`이름을 바꿔주어야 하는데 `@ModelAttribute("item")`으로 설정해서 템플릿 뷰의 코드는 바꾸지 않는다.

```java
        Item item = new Item();
        item.setItemName(form.getItemName());
        item.setPrice(form.getPrice());
        item.setQuantity(form.getQuantity());
```

새로운 Item객체에 ItemSaveForm에 들어온 값들을 저장해서 로직 실행


상품 수정 

```java
@PostMapping("/{itemId}/edit")
    public String edit(@PathVariable Long itemId, @Validated @ModelAttribute("item") ItemUpdateForm form, BindingResult bindingResult) {
        if (form.getPrice() != null && form.getQuantity() != null) {
            int resultPrice = form.getPrice() * form.getQuantity();

            if (resultPrice < 10000) {
                bindingResult.reject("totalPriceMin",new Object[]{10000,resultPrice},null);
            }
        }

        if(bindingResult.hasErrors()){
            log.info("errors = {}",bindingResult);
            return "validation/v4/editForm";
        }

        Item itemParam = new Item();
        itemParam.setItemName(form.getItemName());
        itemParam.setPrice(form.getPrice());
        itemParam.setQuantity(form.getQuantity());

        itemRepository.update(itemId, itemParam);
        return "redirect:/validation/v4/items/{itemId}";
    }
```

---

## Bean Validation - HTTP 메시지 컨버터

`Bean Validation`은 `HttpMessageConverter`(`@RequestBody`) 에도 적용할 수 있다.

```java
@Slf4j
@RestController
@RequestMapping("/validation/api/items")
public class ValidationItemApiController {

    @PostMapping("/add")
    public Object addItem(@RequestBody @Validated ItemSaveForm form, BindingResult bindingResult){

        log.info("API 컨트롤러 호출");

        if(bindingResult.hasErrors()){
            log.info("검증 오류 발생 errors={}",bindingResult);

            return bindingResult.getAllErrors();
        }


        log.info("성공 로직 실행");

        return form;
    }
}
```

### API의 경우 3가지 경우를 나누어서 생각한다.

+ 성공 요청: 성공
+ 실패 요청: JSON을 객체로 생성하는 것 자체가 실패
+ 검증 오류 요청: 객체로 생성은 하지만 검증에서 실패

`HttpMessageConverter`는 JSON을 `Item`객체로 생성하는데 실패한다.

`price` 필드에 문자를 넣는 요청같은 경우에 `Item`객체 자체를 만들지 못해서 컨트롤러가 호출되지 못하고 예외갑 ㅏㄹ생한다.


### @ModelAttribute Vs @RequestBody

`@ModelAttribute`는 각각의 필드 단위로 세밀하게 적용되기 때문에 특정 필드에 타입이 맞지 않는 오류가 발생해도 나머지 필드는 정상 처리할 수 있다.

`HttpMessageConverter`는 전체 객체 단위로 적용되기 때문에 객체를 만드는데에 성공해야 `@Validated`가 적용된다.

`HttpMessageConverter` 단계에서 실패해서 예외가 발생할때 원하는 모양으로 처리하는 방법은 뒤에 예외 처리 부분에서 다룬다.

