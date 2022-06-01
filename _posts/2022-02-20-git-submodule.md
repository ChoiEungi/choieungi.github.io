---
layout: post
title: "Spring Boot에서 git submodule로 민감 정보(yml) 관리"
tags: [Spring Boot, GIST 청원, Git]
date: 2022-02-20 09:30:00 +0900
categories: [Development, Git]

---



# 서브 모듈을 통해 민감 정보 관리

### 3줄 요약

- 민감정보 데이터를 담을 private repo 생성
- 민감정보가 담긴 private 레포지토리를 public 레포지토리의 서브모듈로 `git add submodule ${서브 모듈로 등록할 github repository의 주소}` 을 사용해 등록한다.
- 원격 서브모듈 레포지토리에 있는 파일들을 `git submodule update --remote`을 이용해  로컬에 있는 서브모듈 폴더로 가져 온다.

++ gradle이나 github action으로 서브모듈을 잘 사용한다.



### 구체적 과정

- privates Repo 생성

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-02-20-21%3A18%3A24.png">

- submodule 등록한다. 

그렇다면 다음 나와있듯 /be 경로(프로젝트의 최상단 경로)에 PDG-privates(submodule repo 이름)라는 폴더가 생성된다.(submodule의 내용들) 

`git add submodule ${서브 모듈로 등록할 github repository의 주소}`



<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-02-20-21%3A19%3A14.png">



- `.gitmodules` 추가

submodule의 path(file 명)와 url(github url)을 추가해준다. 단 여기서 submodule의 default branch가 master가 아니라면 반드시 branch

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-02-20-21%3A19%3A38.png">



- `git submodule update --remote `

git submodule 방식은 branch의 hash를 작성하는 방식이다. 그렇기 때문에 `git submodule update --remote` 을 진행하면 submodule의 내용이 update 된다. 본 프로젝트에는 hash가 변경 된다.  이후 반드시 본 프로젝트의 git commit을 진행해야 hash가 제대로 업데이트 된다. 다시 말해, `git commit -am "message"` 를 진행하고 Push를 해야 한다!



### Example

현재 프로젝트는 다음과 같이 설정돼 있다.

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-02-20-21%3A23%3A48.png">

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-02-20-21%3A24%3A10.png">





이후 협업을 진행하거나 외부에서 submodule을 수정했다는 것을 가정하고 Remote에서 다음과 같이 변경한다.

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-02-20-21%3A25%3A03.png">



이후 다음 커맨드를 입력한다.

```
git submodule update --remote
```



그렇다면 다음과 같이 hash 값이 checkout 됐다고 나온다.

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-02-20-21%3A25%3A45.png">





실제로 PDG-privates 폴더 안의 내용이 remote와 같이 변경된다. git diff 를 통해 hash를 확인해봐도 9330cf ~로 변경된 것을 볼 수 있다.

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-02-20-21%3A26%3A24.png">

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-02-20-21%3A22%3A38.png">





이를 커밋하고  push하면

<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-02-20-21%3A27%3A07.png">



<img src="https://raw.githubusercontent.com/ChoiEungi/git-blog-image/upload/2022-02-20-21%3A27%3A46.png">



### Gradle를 이용해 local에서 submodules의 내용을 빌드 시 가져오기

로컬 privates 를 받아올 때, gradle를 사용하면 편리하게 submodule의 내용을 가져올 수 있다.

```gradle
task copyPrivate(type: Copy) {
    copy {
        from './PDG-privates'
        include "*.yml"
        into 'src/main/resources/privates'
    }
}
```

이를 설정하고 build시 submodule의 yml파일들을 'src/main/resources/privates'로 가져온다. **이 때, 반드시 'src/main/resources/privates'를 .gitignore에 추가해줘야 한다.**





### CI/CD in Github Action

```yaml
- name: Checkout 
		uses: actions/checkout@v1 
		with:
		  token: ${{ secrets.GITHUB_TOKEN }} 
		  submodules: true
```

를 workflow file에 추가해주면 된다.



### Error

**Problem**

- branch를 찾을 수 없다고 표시됐다. 아마 default가 master라서 그런듯 하다.

```
fatal: Needed a single revision
Unable to find current origin/HEAD revision in submodule path 'PDG-privates'
```

**Solution**

- branch가 main인데 설정이 HEAD로 돼있었다.
- `.gitmodule`에서 branch를 main으로 변경해줬더니 성공했다.

```
[submodule "PDG-privates"]
   path = PDG-privates
   url = <https://github.com/2022-solution-challenge/PDG-privates>
   branch=main
```



### Reference

[Git - 서브모듈](https://git-scm.com/book/ko/v2/Git-도구-서브모듈)

[Github Action 에서 Submodule 설정 방법](https://lelecoder.com/152)
