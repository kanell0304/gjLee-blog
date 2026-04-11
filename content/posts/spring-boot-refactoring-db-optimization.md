---
title: "[리팩토링] Spring Boot 쇼핑몰 서비스 레이어 DB 접근 최적화"
date: 2026-04-12
tags: ["Java", "Spring Boot", "JPA", "리팩토링", "N+1", "성능최적화"]
categories: ["리팩토링"]
draft: false
---

## 개요

팀 프로젝트로 개발한 Spring Boot 기반 남성 의류 쇼핑몰의 백엔드를 리팩토링하면서
서비스 레이어에서 발견한 **DB 접근 비효율 패턴 5가지**와 개선 방법을 정리했다.

프로젝트를 처음 개발할 당시 기능 구현에 집중하다 보니, 서비스 메서드 곳곳에서
**불필요한 DB 쿼리가 반복 실행되는 패턴**이 여러 곳에 남아 있었다.
이번 리팩토링에서는 코드를 다시 읽으면서 비효율적인 부분을 유형별로 분류하고 개선했다.

---

## 문제 유형 분류

발견된 문제를 크게 5가지 유형으로 분류했다.

| 유형 | 설명 | 해당 서비스 |
|------|------|------------|
| 유형 1 | N+1 쿼리 | Order, ReviewList, Cart |
| 유형 2 | 이미 있는 데이터를 또 조회 | Order, Member |
| 유형 3 | 필요 없는 엔티티 선조회 | Mileage |
| 유형 4 | 전체 조회 후 루프 탐색 (+ 버그) | ReviewList, QnAList |
| 유형 5 | 연관 데이터 분리 조회 | Item |

---

## 유형 1. N+1 쿼리 문제

N+1 문제는 JPA를 사용할 때 가장 흔하게 발생하는 성능 안티패턴이다.
이름의 의미는 다음과 같다.

- **1**: 목록을 가져오는 쿼리 1번
- **N**: 그 결과의 각 항목마다 연관 데이터를 조회하는 쿼리 N번

### 1-1. OrderServiceImpl — 주문 목록 조회

주문 목록을 조회하는 메서드 7개에서 동일한 패턴이 발견됐다.

```java
// 개선 전 — 페이지 20건이면 DB 21번 조회
return orderPage.map(order -> {
    List<OrderItem> items = orderItemRepository.findByOrderId(order.getId()); // 루프마다 실행
    return new OrderDTO(order, items);
});
```

`OrderRepository`에 `@EntityGraph` 주석이 달려 있었지만, `Order` 엔티티에
`orderItems` 컬렉션 필드 자체가 없어서 동작하지 않는 상태로 방치되어 있었다.
그래서 서비스에서 직접 루프를 돌며 하나씩 조회하는 방식이 남은 것이었다.

**해결 방법: 배치 조회(Batch Fetch) 패턴**

`IN` 절로 한 번에 조회한 뒤 Java 메모리에서 `Map`으로 그룹핑하는 방식을 적용했다.

```java
// OrderItemRepository에 추가
@Query("SELECT oi FROM OrderItem oi WHERE oi.order.id IN :orderIds ORDER BY oi.id DESC")
List<OrderItem> findByOrderIds(@Param("orderIds") List<Long> orderIds);
```

```java
// 개선 후 — 항상 DB 2번으로 고정
List<Long> orderIds = orderPage.getContent().stream()
        .map(Order::getId).collect(Collectors.toList());

Map<Long, List<OrderItem>> itemsByOrderId = orderItemRepository.findByOrderIds(orderIds)
        .stream().collect(Collectors.groupingBy(oi -> oi.getOrder().getId()));

return orderPage.map(order ->
        new OrderDTO(order, itemsByOrderId.getOrDefault(order.getId(), List.of())));
```

핵심은 **DB에서 한 번에 다 가져오고, 데이터를 나누는 작업(groupingBy)은 Java 메모리에서 처리**한다는 점이다.

| | 개선 전 | 개선 후 |
|--|---------|---------|
| 20건 페이지 | 21번 | **2번** |
| 50건 페이지 | 51번 | **2번** |

---

### 1-2. ReviewListServiceImpl — 리뷰 이미지 조회

리뷰 목록을 조회하는 4개 메서드에서 같은 문제가 있었다.
그런데 여기서는 이미 `ReviewListRepository`의 모든 쿼리에
`@EntityGraph(attributePaths = "images")`가 적용되어 있었다.

```java
// ReviewListRepository — 이미지가 이미 함께 로딩됨
@EntityGraph(attributePaths = "images")
@Query("SELECT rl FROM ReviewList rl ORDER BY rl.id DESC")
Page<ReviewList> findAllReviewListPage(Pageable pageable);
```

그런데 서비스에서 이미 로딩된 이미지를 또 조회하고 있었다.

```java
// 개선 전 — 이미 로딩됐는데 또 DB 조회
return reviewListPage.map(review -> {
    List<ReviewImage> reviewImageList = reviewImageRepository.findAllByReviewId(review.getId());
    return new ReviewListDTO(reviewImageList, review);
});

// 개선 후 — 메모리에서 바로 꺼냄
return reviewListPage.map(review -> new ReviewListDTO(review.getImages(), review));
```

---

### 1-3. CartServiceImpl — 장바구니 다중 조회

선택된 상품 목록을 처리하는 4개 메서드에서 루프 내 단건 조회가 반복되고 있었다.

```java
// 개선 전
for (Long cartItemId : cartDTO.getSelectId()) {
    Cart cart = cartRepository.findByMemberIdAndOptionId(member.getId(), cartItemId); // N번 반복
    cartList.add(cart);
}

// 개선 후
List<Long> optionIds = Arrays.asList(cartDTO.getSelectId());
List<Cart> cartList = cartRepository.findByMemberIdAndOptionIds(member.getId(), optionIds); // 1번
```

`editCartListByMemberIdANDOptionId`는 기존에 `save()`도 루프에서 반복 호출하던 것을
`saveAll()`로 묶었고, `multipleDeleteItemFromWishList`도 `deleteById()` 반복을
`deleteAll()`로 묶었다.

---

## 유형 2. 이미 있는 데이터를 또 조회

### 2-1. OrderServiceImpl.createOrder() — Member 이중 조회

메서드 안에서 동일한 `memberId`로 Member를 두 번 조회하고 있었다.

```java
// 42번째 줄
Member member = memberRepository.findById(orderDTO.getMemberId()).orElseThrow(...);

// ... 약 100줄의 로직 ...

// 140번째 줄 — 완전히 동일한 쿼리
Member memberMileage = memberRepository.findById(orderDTO.getMemberId()).orElseThrow(...);
if (orderDTO.getUsingMileage() <= memberMileage.getStockMileage()) { ... }
```

42번째 줄의 `member` 객체가 메서드 끝까지 살아있는데 재사용하지 않고 있었다.

```java
// 개선 후 — 기존 변수 재사용
if (orderDTO.getUsingMileage() <= member.getStockMileage()) { ... }
```

---

### 2-2. MemberServiceImpl.updateMember() — 존재 확인 후 즉시 재조회

존재 여부를 `COUNT` 쿼리로 확인하고, 바로 다음 줄에서 같은 조건으로 `SELECT` 쿼리를 또 날리고 있었다.

```java
// 개선 전 — 2번 조회
if (!existsByEmail(memberDTO.getEmail())) {              // SELECT COUNT(*) ...
    throw new RuntimeException("회원을 찾을 수 없습니다.");
}
Member searchMember = memberRepository.findByEmail(...); // SELECT * ... (동일 조건)

// 개선 후 — 1번 조회
Member searchMember = memberRepository.findByEmail(...);
if (searchMember == null) {
    throw new RuntimeException("회원을 찾을 수 없습니다.");
}
```

---

## 유형 3. 필요 없는 엔티티 선조회

### 3-1. MileageServiceImpl — Member/Order 선조회 (6개 메서드)

`memberId`를 파라미터로 받아서 마일리지를 조회하는 메서드인데,
`member.getId()`를 쓰기 위해 Member 엔티티 전체를 먼저 불러오고 있었다.

```java
// 개선 전
Member member = memberRepository.findById(memberId).orElseThrow(...); // 전체 필드 로딩
List<Mileage> mileageList = mileageRepository.findAllByMemberId(member.getId()); // 결국 memberId와 동일

// 개선 후
List<Mileage> mileageList = mileageRepository.findAllByMemberId(memberId); // 바로 사용
```

`findAllByMemberEmail`의 경우 email → ID 변환이 필요했으므로,
`MileageRepository`에 email로 직접 조회하는 쿼리를 추가하여 Member 조회를 완전히 없앴다.

```java
// MileageRepository에 추가
@Query("SELECT m FROM Mileage m WHERE m.member.email = :email ORDER BY m.id DESC")
List<Mileage> findAllByMemberEmail(@Param("email") String email);
```

---

## 유형 4. 전체 조회 후 루프 탐색 (+ 버그 수정)

이 유형은 단순히 비효율적인 것을 넘어 **로직 버그**까지 함께 존재했다.

### 4-1. ReviewListServiceImpl.checkPurchaseStatus() — 3가지 문제

리뷰 작성 전 구매 이력을 확인하는 메서드에서 3가지 문제가 동시에 발견됐다.

```java
// 문제 1: N+1 체인 유발
List<OrderDTO> orderDTO = orderService.findAllByMemberId(memberId);

// 문제 2: 전체 주문을 가져와서 루프 탐색
int listIndex = 0;
for (OrderDTO targetOrder : orderDTO) {

    // 문제 3: listIndex가 항상 0 고정 → 각 주문의 첫 번째 상품만 확인
    // 두 번째 상품 이후는 영원히 구매 확인 불가 → 구매했어도 리뷰 못 씀
    if (targetOrder.getOrderItemList().get(listIndex).getItemId() == itemId) {
        checkStatus = true;
    }
}
```

`listIndex`가 루프 안에서 증가하지 않으니 항상 0이다.
예를 들어 첫 번째 주문에 상품 A, B가 있고 B를 구매했는데
index 0인 A와만 비교하므로 구매 확인이 실패하는 버그다.

```java
// 개선 후 — 단일 EXISTS 쿼리 1번
return orderService.existsPurchase(memberId, itemId);
```

```java
// OrderItemRepository에 추가
@Query("SELECT COUNT(oi) > 0 FROM OrderItem oi " +
       "WHERE oi.order.member.id = :memberId AND oi.item.id = :itemId AND oi.order.delFlag = false")
boolean existsByMemberIdAndItemId(@Param("memberId") Long memberId, @Param("itemId") Long itemId);
```

---

### 4-2. QnAListServiceImpl.checkWritingStatus() — Long 비교 버그

작성자 권한 확인 메서드에서 회원의 전체 QnA 목록을 가져와 루프를 돌고 있었고,
`Long` 타입을 `==` 연산자로 비교하는 버그가 있었다.

```java
// 개선 전 — Long == 비교 버그
for (QnAList targetQnAList : qnAList) {
    if (targetQnAList.getId() == qnaListId) { // ID가 128 이상이면 항상 false
        checkStatus = true;
    }
}
```

Java에서 `Long` 객체는 **-128 ~ 127 범위만 캐싱**된다.
127을 넘는 ID값에서는 `==`로 비교하면 같은 값이어도 다른 객체 참조를 비교하게 되어 항상 `false`가 된다.
즉, QnA 글 ID가 128 이상인 경우 **작성자가 자신의 글을 수정/삭제할 수 없는 버그**가 발생한다.

```java
// 개선 후 — DB에서 직접 존재 여부 확인 (버그 자체가 없어짐)
return qnAListRepository.existsByMemberIdAndQnaListId(memberId, qnaListId);
```

```java
// QnAListRepository에 추가
@Query("SELECT COUNT(ql) > 0 FROM QnAList ql WHERE ql.member.id = :memberId AND ql.id = :qnaListId")
boolean existsByMemberIdAndQnaListId(@Param("memberId") Long memberId, @Param("qnaListId") Long qnaListId);
```

---

## 유형 5. 연관 데이터 분리 조회

### 5-1. ItemServiceImpl.getOne() — 4번 분리 조회

상품 상세 1건 조회 시 기본 정보, 이미지, 옵션, 인포를 각각 별도 쿼리로 조회하고 있었다.

```java
// 개선 전 — 4번 쿼리
Item item = itemRepository.findById(id)...;                        // 쿼리 1
List<ItemImage> images = itemImageRepository.findByItemId(id);     // 쿼리 2
List<ItemOption> options = itemOptionRepository.findByItemId(id);  // 쿼리 3
return new ItemDTO(item, images, options, item.getInfo());          // 쿼리 4 (지연 로딩)
```

같은 파일의 `getAllItemsWithImageAndOptionsAndInfo()`는 이미 배치 조회로 올바르게 구현되어 있는데
`getOne()`만 이 방식으로 남아 있었다.

`@EntityGraph`를 적용해 연관 데이터를 함께 로딩하도록 개선했다.
단, `@OneToMany` List 컬렉션을 여러 개 동시에 JOIN하면 카테시안 곱 문제가 생길 수 있으므로
Hibernate가 보조 SELECT로 처리하게 한다.

```java
// ItemRepository에 추가
@EntityGraph(attributePaths = {"images", "options", "info"})
@Query("SELECT i FROM Item i WHERE i.id = :id")
Optional<Item> findWithDetailsById(@Param("id") Long id);
```

```java
// 개선 후
Item item = itemRepository.findWithDetailsById(id)
        .orElseThrow(() -> new IllegalArgumentException("해당 상품이 존재하지 않습니다. ID: " + id));
return new ItemDTO(item, item.getImages(), item.getOptions(), item.getInfo());
```

---

## 정리

이번 리팩토링을 통해 느낀 점을 정리하면 다음과 같다.

**1. N+1은 코드만 봐서는 잘 안 보인다**

루프 안에서 Repository 메서드를 호출하는 코드는 얼핏 보면 자연스러워 보인다.
실제로 실행되는 쿼리 수를 추적해봐야 문제가 드러난다.
JPA를 사용할 때는 `spring.jpa.show-sql=true`나 쿼리 로그를 습관적으로 확인하는 것이 중요하다.

**2. `@EntityGraph`가 있어도 서비스에서 또 조회하면 의미 없다**

Repository에 `@EntityGraph`를 붙여뒀는데 서비스에서 별도 Repository를 또 호출하면
최적화 효과가 전혀 없다. Repository와 Service 코드를 함께 보는 습관이 필요하다.

**3. 비효율 옆에 버그가 같이 숨어있는 경우가 많다**

`checkPurchaseStatus()`의 `listIndex` 고정 버그,
`checkWritingStatus()`의 `Long ==` 비교 버그처럼,
성능 문제를 찾다 보면 로직 버그도 함께 발견되는 경우가 많았다.
코드를 다시 읽는 것만으로도 버그를 잡을 수 있었다.

**4. 해결 방법은 결국 "DB 왕복 횟수를 줄이는 것"**

모든 유형의 개선이 결국 같은 방향을 가리키고 있었다.
DB 쿼리는 네트워크 왕복 비용이 발생하기 때문에 횟수를 줄이는 것이 핵심이다.
한 번에 가져와서 Java 메모리에서 처리하거나, EXISTS 쿼리로 최소한의 데이터만 조회하는 방식이 효과적이었다.
