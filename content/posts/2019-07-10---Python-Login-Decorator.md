---
title: pyJWT를 이용한 Python Django Login Decorator
date: "2019-07-10T23:46:37.121Z"
template: "post"
draft: false
slug: "/posts/perfecting-the-art-of-perfection/"
category: "Design Inspiration"
tags:
  - "Python3"
  - "Wecode"
  - "Django"
description: "Python3 pyJWT를 이용한 파이썬 로그인 데코레이터 구현"
---
(읽기 쉽고, 쉽게 표현하기 위해 경어를 사용한 점 양해부탁드립니다.)

며칠동안 전전긍긍하던 파이썬 로그인 데코레이터를 완료하였고, 구현하는 과정 및 결과에 대해 포스팅 해보려 한다. 개발을 배우는 입장에서 정리를 하다보니 부족한 부분이 있을 수 있으니 양해바란다.

![Nulla faucibus vestibulum eros in tempus. Vestibulum tempor imperdiet velit nec dapibus](/media/image-2.jpg)

데코레이터 구현을 위해 인터넷을 계속 찾아다녀봤지만, pyJWT를 이용한 구현은 찾아보기가 힘들었고, 오늘 배운것을 정리하는 것과 pyJWT를 통한 인증구현이 필요한 사람을 위해 정리하려 한다.  

먼저 JWT는,  
JavaScript를 이용해 유저가 웹사이트에 로그인 한 후에 발생하는 이벤트(댓글, 게시판 글쓰기 등), 여러가지 서비스를 이용하기 위한 권한을 가진 유저인지 확인하기 위해서 많이 쓰이고 있다고 한다. 
총 3개의 구문으로 되어있고, 헤더.내용.서명 으로 이루어져 있다.
(자세한 내용은 [JWT](https://jwt.io/)공식홈페이지나 다른 사이트를 참고해주길 바란다.)

혹여나 데코레이터가 뭔지 모르는 분들을 위해 설명드리자면,   
대상 함수를 wrapping하고, 이 함수 앞뒤로 꾸며질 구문들에 대해 손쉽게 재사용할 수 있도록 다른 곳에 미리 함수를 만들어 둔 것이라고만 알고 있으면 될 듯 하다. 이번 프로젝트를 예로 들자면, 회원이 댓글작성이나 특정권한을 가지고 있는지 확인하기 위해 데코레이터를 구현했다.  
ex)사용예시
```
@decorator
def comment:
  ~~~
```
(더 전문적인 설명이 필요하다면 [파이썬](https://www.python.org/dev/peps/pep-0318/) 공식홈페이지나 구글링을 해보길 권한다.)

개발순서는 회원가입, 로그인을 이미 구현한 상태이고, 데코레이터를 구현해야 하는데 먼저 목적과 로직을 디테일하게 작성한 내용을 먼저 얘기해보겠다.

###구현하고자 하는 데코레이터의 목적
웹사이트 상에서 권한(로그인, 혹은 관리자)을 가진 유저만 이용가능한 서비스를 이용할 수 곳에 한해서, 유저가 로그인한 유저인지, 이용가능한 권한을 가진 유저인지(게시판 글쓰기, 관리자페이지 이용권한, 포스팅 작성 등) 등을 확인하기 위해서 미리 선언해 둔 함수를 사용하기 위한 목적 

###세부 로직
유저로그인: 프론트엔드에게 해당 아이디로 발행한 암호화된 jwt토큰을 전달
1. 해당유저의 정보를 확인하기 위해 프론트엔드에서 HTTP 헤더를 통해 토큰을 전달 받기  
=>만약 토큰이 없다면 서버에서는 에러코드를 프론트엔드에게 전달하고, 유저의 현재 페이지를 로그인페이지로 이동해달라고 프론트엔드에게 요청
2. 백엔드는 암호화되어 전달받은 jwt토큰을 decode하고  
=> decode에서 오류가 난다면, 해당 웹페이지에서 발행한 토큰이 아니므로, INVALID_TOKEN 에러를 프론트엔드에게 전달
3. Django models.py에 저장된 데이터와 decode한 데이터가 일치하는 회원의 정보를 변수에 저장  
=> 유저의 정보가 없다면, UNKNOWN_USER라는 에러메세지를 프론트엔드에게 전달
4. request할 객체에 user변수를 저장하여 프론트엔드에게 전달
5. 위 내용이 잘 실행되는지 httpie나 다른 방법으로 테스트 및 검증진행

더 자세한 사항은 코드 밑에 써두도록 하겠다.  

###코딩 순서
(세부로직 순서에 맞게 작성)
0. pyJWT설치
`pip install pyJWT` 및 django app디렉토리에서 utils.py 파일 생성

```
def login_decorator(func):

    def wrapper(self, request, *args, **kwargs): # self > 받아온 함수를 다시 넘긴다 access token이 헤더에 들어있음> json.load가 아님(헤더에 있는 값만 할 것임) > 키 벨류로 돼있는 양식 > 
    
        if "Authorization" not in request.headers: #1)번
            return JsonResponse({"error_code":"INVALID_LOGIN"}, status=401)
        
        encode_token = request.headers["Authorization"] 

        try:
            data = jwt.decode(encode_token, wef_key, algorithm='HS256') 
            #2번)decode를 하게 될 경우 프론트엔드에 전달했던 페이로드값만 나옴(즉 로그인뷰에 바디)

            user = User.objects.get(id = data["id"])#3번
            request.user = user #4번
        except jwt.DecodeError: #2-1번 error
            return JsonResponse({
                "error_code" : "INVALID_TOKEN"
            }, status = 401) # 401에러 : 권한이 없을때 발생
        except User.DoesNotExist:#1-1번 error
            return JsonResponse({
                "error_code" : "UNKNOWN_USER"
            }, status = 401) # 401에러 : 권한이 없을때 발생

        return func(self, request, *args, **kwargs) #5번

    return wrapper
```
1. 데코레이터 구현 및 해당 유저 정보 확인
=> 데코레이터를 만들기 위해 다중 함수 구현, 받은 데이터 중에 토큰이 있는지 if문을 통해 확인
=> 401에러는 권한이 없을 때 발생

2. jwt토큰 decode   
=> 받은 토큰과 기존에 저장해두었던 SECRET_KEY와 algorithm으로 토큰을 decode
=> decode를 하게 될 경우 프론트엔드에 전달했던 페이로드값만 나옴(즉 로그인뷰에 바디)

3. decode한 토큰과 일치하는 유저 정보를 변수에 저장
=>Query문을 이용해 DB에 접근하여 원하는 데이터를 가져온다
=>get메소드를 사용할 경우 DB에서 하나의 row만 가지고 옴(가로)

4. 프론트엔드에게 받은 request.user에 3번의 자료 저장
=>프론트엔드에게 전달해주기 전 준비과정

5. 4번에 저장된 request를 데코레이터 리턴  
=> 데코레이터종료 및 프론트엔드에게 해당 유저의 정보를 리턴

6. 테스트할 변수를 views.py에 테스트변수 만들고, urls를 통해 url을 연결한 후  결과가 잘 나오는지 확인  

```pip install httpie```
=>views.py에 아래 코드 입력, urls.py에 
```
#user/views.py
from user.utils import login_decorator  

class Example(View):
    @login_decorator
    def get(self, request):
        user = User.objects.get(id = 11)
        return JsonResponse({
            'user_name'    : user.name            
        })

#user/urls.py
from .views import Example

urlpatterns = [
    path('/a', Example.as_view())
    ]
```

##결론
어떻게 보면 정말 간단한 함수처럼 보이지만 이제 막 개발을 시작하거나 데코레이터를 실전(?)에서 어떻게 써야 하는지 위 자료를 토대로 조금이나마 도움이 됐으면 합니다.