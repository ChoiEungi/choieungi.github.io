---

layout: post

title: "[Thinking] MSA에 대한 단상"

tags: [Development, Thinking]

date: 2022-08-09 18:39:00 +0900

categories: [생각]

---



SW 마에스트로 과정에서 너무나도 영광스럽게 조대협 멘토님의 k8s 멘토링을 4회 가량을 듣게 되었다. 오늘이 첫번째 강의였고 들으면서 기술에 대한 깨달음이 꽤나 커서 글을 작성한다. 뿐만 아니라 블로그에 좋은 글을 써야 한다는 강박이 계속되어 지금이라도 작은 글들을 조금씩 써나가는 연습을 해보려 한다.

k8s를 배우기 앞서 k8s가 가장 잘 활용될 수 있는 구조인 MSA와 그 기반이 되는 Container에 대해서 먼저 다뤘다. 여기서 누차 강조하는 것은 기술을 배우기 전에 그 Background를 알고 **그 기술을 왜 사용하는지에 대한 의문을 꾸준히 제기해야 한다는 것이다.**

먼저 MSA를 왜 사용하는가?에 대한 질문을 답한다면, 조직마다 사용하는 이유는 각기 다르겠지만 기능 단위로 나눈다는 점이었다. 나는 기능 단위로 나눈다는 것을 통해 가용성을 높이는 것(어떤 단위의 서비스가 죽어도 다른 서비스는 정상적으로 작동한다는 의미)로서 가장 크게 받아들였다. 이를 통해 서비스를 최대한 안정적으로 돌릴 수 있다는 장점이 가장 먼저 떠오르는 이유였다. 나름 타당한 이유이지만, 멘토링 시간에 들은 내용으로는 **기능 단위로 독립적으로 분리함을 통해 그 기능을 구현하는 조직의 의사결정을 명확히 위임할 수 있다는 측면이었다**. 이는 곧 의사결정의 속도가 빨라진다는 것이고, 생산성 향상으로 이어져 소프트웨어를 개발하는 방법론으로서 굉장히 좋은 방법일 수 있었다.

관련한 법칙으로 Conway’s Law도 소개해주셨다. 이는 다음과 같다.

> Any organization that designs a system (defined broadly) will produce a design whose structure is a copy of the organization's communication structure.

조직의 communication structure가 곧 조직의 system의 design에서 나타난다는 것이다. 만약 조직의 커뮤니케이션 구조가 복잡하다면 조직의 system design도 복잡해진다는 의미이다. 즉, 위계 조직과 같이 복잡한 커뮤니케이션 구조에서는 MSA를 사용한다고 하더라도 그 의미에 맞고 적합하게 사용하고 있지 못할 수 있다는 말이다. 결국 아키텍쳐의 가장 큰 원칙은 DDD와 같이 MSA를 잘 만드는 방법론보다도 팀의 구조에 맞춰 설계를 하는 것이 가장 중요함을 느낄 수 있었다.

이런 생산성 증대에 있어 궁금한 점이 있어 “그렇다면 MSA를 진행하는데 생산성 향상에 이를 수 있을 정도의 팀의 규모는 어느 정도인가?”에 대한 질문을 드렸다. 기능이 independent적인 관점에서 혼자 진행할 수도 있다고 했고 실제로 미국의 많은 스타트업에서는 작은 스타트업들도 MSA를 사용한다고 한다. 현업에서는 일반적으로 2-pizza 법칙으로 팀당 8명 내외로 꾸리고 여러 팀들을 묶어 30~40명 가량으로 운영할 수 있다고 한다. 그래서 팀마다 한달에 한번 릴리즈를 한다고 해도 4주에 4번 릴리즈가 되어 굉장히 빠르게 릴리즈를 가능하다고 하셨다.

하지만,  MSA가 꼭 정답만은 아니다. 팀의 이해관계가 맞지 않는다면 오히려 학습하는데에도 시간을 낭비하고 맞지 않는 옷을 입으려니 생산성이 저하될 수도 있다. 그렇기 때문에 비즈니스에 맞지 않는 맹목적인 기술에 대한 맹신과 오버엔지니어링은 엔지니어로서 좋은 자세는 아닌 것 같다.