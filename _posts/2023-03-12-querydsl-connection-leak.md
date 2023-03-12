---

layout: post

title: "querydsl의 transform 메서드에서 발생하는 connection leak 현상"

tags: [Spring, querydsl]

date: 2023-03-12 23:30:00 +0900

categories: [Development]

---





### 문제 상황

회사에서 모든 환불은 어드민 서버를 거쳐 환불 서버에 환불 요청을 보내 환불 프로세스가 진행된다. 하지만 환불 서버에서 요청을 제대로 보내고 환불을 완료했지만, 어드민 서버에서 히스토리를 DB에 기록하는 작업이 제대로 이뤄지지 않았다. 그렇기에 어드민 히스토리와 환불 기록의 불일치가 발생했고, 이 운영 이슈를 해결하는 과정을 남기려고 한다.

실제로 핀포인트 로그는 다음과 같이 나타났다.

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2023-03-12-23%3A29%3A19.png">

대략적으로 보면 알듯, 모두 30초(혹은 그 이상)에서 오류가 발생한 이력이 있는데 이는 어딘가에서 Timeout이 발생했음을 알 수 있다. 더 구체적으로 들어가서 확인해보면 에러가 다음과 같이 발생했다.

> HikariPool-1 - Connection is not available, request timed out after 30000ms.



### 해결 과정

처음에 생각한 문제는 슬로우 쿼리가 커넥션을 오래 잡거나 배포 중 ECS 오토스케일링이 활성화되어 DB 커넥션이 부족해진 이유인 줄 알았다. 하지만 AWS 로그를 직접 확인해본 결과 당시 배포가 이뤄지지 않았으며 DB 커넥션 개수는 넉넉했었고 쿼리들도 문제가 없었다.

그렇기에 인프라적 문제보다는 어플리케이션 서버의 문제라고 생각했고, 어플리케이션 로그와 코드를 살펴봤다. 그 결과 문제는 querydsl에서 `transform()` 메서드를 잘못 사용하고 있어 발생했음을 확인할 수 있었다. 실제로 문제 상황과 비슷한 상황을 개발 서버에서 재현했을 때 `transform()` 메서드를 동시에 호출할 때 여러 요청들이 쿼리가 실행된 이후에도 커넥션을 계속 물고 있었으며 hikari의 모든 커넥션을 물게 되면, 다른 요청들은 hikari로부터 커넥션을 대기하게 되고 타임 아웃(30초)가 발생했다.

querydsl에서 `transform()` 메서드는 쿼리 결과를 grouping해서 Map으로 변환해주는 기능을 제공한다. 하지만 querydsl에서 query가 종료될 때 사용되는 메서드들이 `queryTerminatingMehtods`에 존재하는 메서드들이라면 JPA EntityManager를 close해준다. `queryTerminatingMehtods`에 존재하는 항목은 다음과 같다.

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2023-03-12-23%3A29%3A50.png">



querydsl에서 자주 쓰는 `fetch()`나 `fetchOne()`같은 메서드는 `queryTerminationMethods`가 위의 메서드 리스트에 존재한다.



<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2023-03-12-23%3A35%3A39.png">

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2023-03-12-23%3A35%3A58.png">

반면, 쿼리가 종료되는 메서드가 존재하는 `ResultTransformer` 인터페이스의 transfrom 메서드가 사용하는 query는 모두 `iterate()`로 종료된다. 따라서 `queryTerminationMethods`의 메서드가 아니며, EntityManager가 제대로 커넥션을 닫히지 않게 된다.



<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2023-03-12-23%3A36%3A44.png">







### 해결 방법

문제를 찾는 데에 비해 해결하는 방법은 굉장히 간단했다. querydsl **`transform()` 메서드를 사용하는 쿼리에 `@Transactional(readOnly=true)`를 붙여주면 된다.** querydsl도 결국 JPA를 기반으로 만들어진 라이브러리이며, `@Transactional` 을 갖는 메서드가 끝날 때 `JpaTransactionManager` 의 `doCleanupAfterCompletion()`을 통해 커넥션을 모두 정리해준다.



### 아쉬운 점

transform이 connection을 계속 물고 있어 오류가 발생하는 부분은 이해했지만, 결국에는 Hikari pool로 커넥션을 언젠가 돌려주게 된다. 이 돌려주는 원리가 hikari의 max-lifetime(기본 180초)로 인해 돌려주는 것인지, OS에 의해 좀비 스레드가 정리되는 것인 지 명확히 알아내지는 못했다.

또, querydsl 5.0.0 기준으로 `transform()` 메서드에서 커넥션이 왜 반납이 안되는 지 내부 원리를 구체적이고 명확히 알아보려 한다. 관련해서는 다음 글에서 풀어내 보자.



### 참고

- https://github.com/querydsl/querydsl/issues/3089
- https://colin-d.medium.com/querydsl-에서-db-connection-leak-이슈-40d426fd4337
- https://cljdoc.org/d/hikari-cp/hikari-cp/3.0.1/doc/readme
