---

layout: post

title: "[Spring] spring data jpa에서 save 내부 원리"

tags: [Development, spring data jpa, spring]

date: 2022-08-24 17:32:00 +0900

categories: [Spring Boot Development]

---



## 문제

```java
  @Test
  void retrieveInboxAllLettersTest() {
        Letter letter = new Letter("content", user);
        letter.send(user1);
        letterRepository.save(letter);
        letterRepository.save(letter);
        letterRepository.save(letter);
        List<InboxLetterResponse> inboxLetterResponses = letterService.retrieveInboxAllLetters(user1.getId());
        assertThat(inboxLetterResponses.size()).isEqualTo(3);
    }
```

전체 편지를 조회하는 로직에서 3개를 insert한 후 이를 전체 조회해서 size를 비교하는 테스트코드를 작성하던 중 실패했습니다. 결과는 다음과 같이 나왔는데 편지에 대해서 insert쿼리가 1번만 발생한 것으로 보입니다. 실제로 전체 조회 후 size를 조회하면 1개가 나왔습니다. 

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-08-24-17%3A24%3A00.png">

기본적으로 save가 identical하게 작동하고, flush를 진행하지 않아 발생한 것으로 유추되었는데 계속 실수가 발생하는 부분이기에 내부 동작 원리를 직접 확인해보겠습니다.

## JpaRepository의 구조

기본적으로 `JpaRepository` 는 다음과 같은 구조를 갖고 있습니다. 여기서 save는 `CrudRepository` 에서 인터페이스를 제공합니다. 참고로 `PagingAndSortingRepository`는 docs에서 나와있듯 `CrudRepository` 의 Extension으로, pagination과 sorting을 이용한 조회 method를 제공합니다. 

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-08-24-17%3A24%3A29.png">

### CrudRepository

`CrudRepository`에서 `save`는 다음과 같이 인터페이스를 제공합니다. 여기서 docs의 설명이 굉장히 중요한데, 인스턴스가 변했을 수 있으므로, **저장 작업(insert query)이 `save`에서 return된 인스턴스를 사용한다고 되어있습니다**. 제가 겪은 이슈에서는 이 return된 instance가 중복된 것으로 유추할 수 있습니다. 

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-08-24-17%3A24%3A49.png">

구현체인 `SimpleJpaRepository`로 들어가보면 다음과 같습니다. 

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-08-24-17%3A25%3A12.png">

만약 identity가 새로운 entity라면 persist한 후 entity를 리턴하는데, 그렇지 않다면 merge를 진행하게 됩니다. merge를 진행하면 Persistent Context에 객체가 추가되지 않기 때문에 이후에 save가 진행되는 객체가 insert되지 않게됩니다.

### 디버깅을 통한 실제 작동 확인

 실제로 문제를 파악하기 위해서 디버깅을 진행해보면. 먼저 save가 실행 지점에 breakpoint를 잡고 이후 내부 동작을 확인하기 위해 Spring-data-jpa의 jar에 들어가서 `SimpleJpaReposiotry` 에서 breakpoint를 잡으면 다음과 같습니다.

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-08-24-17%3A35%3A20.png">

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-08-24-17%3A36%3A20.png">



디버깅을 실행하면 다음과 같습니다.

### 첫 번째 `save`

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-08-24-17%3A26%3A32.png">

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-08-24-17%3A26%3A50.png">

### 두 번째 `save`

두번 째에 대해서는 Persistent Context에 이미 존재하기 때문에 merge를 진행하게 됩니다. 이는 곧 identity가 똑같은 객체가 변경되었을 때 Persistent Context에 적용되고 이후 flush를 진행하면 쿼리가 발생하게 됩니다.

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-08-24-17%3A27%3A16.png">

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-08-24-17%3A27%3A32.png">

뿐만 아니라 해결책으로 `saveAndFlush` 를 사용하면 되지 않나? 싶어서 진행했는데도 통과하지 않았습니다. 이는 flush는 Persistent Context를 지우는 것이 아니라, Persistent Context의 내용을 DB에 쿼리를 날려 반영하는 것인 것도 놓쳤던 것 같습니다. 결국에는 identity가 다른 객체를 각각 만들어 save를 진행해 해결할 수 있었습니다. 

```java
@Test
void retrieveInboxAllLettersTest() {

    for (int i = 0; i < 3; i++) {
        Letter letter = new Letter("content", user);
        letter.send(user1);
        letterRepository.save(letter);
    }
    List<InboxLetterResponse> inboxLetterResponses = letterService.retrieveInboxAllLetters(user1.getId());
    assertThat(inboxLetterResponses.size()).isEqualTo(3);
}
```

굉장히 기본적인 내용인데 생각보다 놓치기 쉬운 부분이기에 글을 작성합니다. JPA에서는 객체의 Identity(객체의 id값)를 통해 새로운 객체임을 구분하고 save메소드에서는 이를 기준으로 insert가 작동하기 때문에 이를 유의해야 할 것 같습니다.