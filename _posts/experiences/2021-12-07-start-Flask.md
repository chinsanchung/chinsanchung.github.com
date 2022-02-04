---
title: "Flask 초기 환경 설정"
layout: single
author_profile: false
read_time: false
comments: false
share: true
related: true
categories:
  - experience
toc: true
toc_sticky: true
toc_labe: 목차
description: 파이썬 Flask 를 설치하면서 겪은 환경 설정을 정리합니다.
excerpt: 파이썬 Flask 를 설치하면서 겪은 환경 설정을 정리합니다.
tags:
  - python
  - Flask
---

지금까지 TypeScript, Node.js, NestJs 로 백엔드를 개발했는데, 프리온보딩 과정을 진행하면서 파이썬을 백엔드 개발에 활용하는 기업이 꽤 많다는 것을 알았습니다. 그래서 이번에 [Flask(이하 플라스크)](https://flask-docs-kr.readthedocs.io/ko/latest/index.html) 를 설치하여 어떤 방식으로 개발하는지 알아보려 합니다.

## 초기 설정

우선 가장 처음 설치해야 하는 도구는 virtualenv 입니다.

### virtualenv

파이썬 프로젝트마다 다른 파이썬 버젼을 사용하는 경우가 많은데(예시로 플라스크는 파이썬 3.x 버전을 지원하지 않습니다.) 데스크톱의 파이썬을 매번 바꾸는 것은 어렵고 번거롭습니다. 그래서 [virtualenv](https://virtualenv.pypa.io/en/latest/)으로 가상 환경을 만들어 한 데스크톱 안에 여러 버전의 파이썬과 라이브러리를 설치해 운영하는 것이 필요합니다.

공식 문서에서는 `sudo pip install virtualenv`으로 설치하지만, 제 맥북에서는 "pip: command not found" 에러가 발생했습니다. `pip` 대신 python3 과 함께 자동으로 설치되는 `pip3`으로 virtualenv 를 설치했습니다.

설치가 완료되면, 프로젝트를 수행할 폴더에 venv 로 가상 환경을 설정합니다.

```bash
$ mkdir myproject
$ cd myproject
$ virtualenv venv
```

이제 아래의 명령어로 가상 환경을 실행할 수 있습니다.

```bash
# Mac OS
$ . venv/bin/activate
# 윈도우
$ . venv/bin/activate
```

가상 환경을 실행할 때마다 activate 명령을 입력하는 것은 번거롭습니다. 필요한 명령을 한 파일에 모아 실행시켜주는 [virtualenvwrapper](https://virtualenvwrapper.readthedocs.io/en/latest/)가 필요한 시점입니다.

### virtualenvwrapper

virtualenvwrapper 으로 배치 파일을 만들어 프로젝트에 필요한 명령 및 환경 변수를 설정할 수 있습니다.

`sudo pip3 install virtualenvwrapper`으로 도구를 설치합니다.

프로젝트 폴더에 가상 환경을 설정했기에 "/Users/username/Documents/Code/flask_project/venv/bin" 경로에 virtualenvwrapper.sh 파일을 작성합니다.(프로젝트의 경로는 임의로 설정하시면 됩니다.)

> 편의성을 위해 앞으로 `/Users/username/Documents/Code/flask_project` 을 `프로젝트 경로`라고 줄여서 작성하겠습니다.

```bash
#!/bin/zsh

cd /Users/username/Documents/Code/flask_project
source /Users/username/Documents/Code/flask_project/venv/bin/activate
```

- `chmod a+x 프로젝트 경로/venv/bin/virtualenvwrapper.sh`을 터미널에 입력해 실행 권한을 부여합니다. ("zsh: permission denied" 에러를 방지합니다.)
- 그 다음 `source 프로젝트 경로/venv/bin/virtualenvwrapper.sh`을 터미널에 입력해 엔터를 누르면 virtualenvwrapper.sh 를 실행합니다.

## 플라스크 설치하기

가상 환경을 실행한 상태에서 `pip install Flask`를 명령어로 입력해 가상 환경에서 플라스크를 설치합니다.

플라스크를 사용하면서 여러 환경 변수를 적용할 텐데, 프로젝트를 실행할 때마다 환경 변수를 입력하는 것은 번거롭습니다. 이번에도 virtualenvwrapper.ts 에 환경 변수를 설정할 수 있습니다.

```bash
#!/bin/zsh

cd /Users/username/Documents/Code/flask_project
export FLASK_APP=test.py
export FLASK_ENV=development
source /Users/username/Documents/Code/flask_project/venv/bin/activate
```

- `export FLASK_APP=test.py`: FLASK_APP 환경 변수가 지정되지 않은 경우 자동으로 app.py를 기본 애플리케이션으로 인식합니다. 예시로 app.py 대신 test.py 를 기본 애플리케이션으로 하고 싶을 때 환경 변수로 입력합니다.
- `export FLASK_ENV=development`: 개발 환경으로 설정해 디버그 모드를 활성화합니다.

virtualenvwrapper.sh 파일을 터미널에서 실행한 후 `flask run`을 입력하면 위의 명령을 적용하여 플라스크 앱을 실행합니다.

## 기본 설정

### 앱 만들기

플라스트 앱 객체를 생성합니다. 다만, 전역 객체를 사용하지 않고 애플리케이션 팩토리를 사용하여 순환 참조 등의 에러를 방지합니다. (참조: [점프 투 플라스크 2-02](https://wikidocs.net/81504), [Flask Factory Pattern to set up your project.](https://itnext.io/flask-factory-pattern-to-setup-your-project-8fe7d6b23247))

```python
# app/__init__.py
from flask import Flask

def create_app():
  app = Flask(__name__)

  return app
```

`__name__`으로 애플리케이션 이름을 설정합니다. 애플리케이션으로 시작하는지, 아니면 모듈로 임포트하는지를 구분해야 하기 때문입니다.

### 환경 변수 설정하기

config.py 파일을 생성하고, 거기에 입력한 설정을 플라스크 앱에 등록합니다. 상세한 설정은 [설정 다루기](https://flask-docs-kr.readthedocs.io/ko/latest/config.html)에서 확인하실 수 있습니다.

```python
# config.py
DEBUG = True
SECRET_KEY = 'key'
```

```python
import config

# app/__init__.py
def create_app():
  app = Flask(__name__)
  app.config.from_object(config)
  # ...
```

**ENV 환경 변수 활용**

AWS 등 외부 데이터베이스의 설정은 숨겨서 불러야 합니다. `pip install python-dotenv`로 python-dotenv 패키지를 설치하면 불러올 수 있습니다.

```python
# config.py
import os
from dotenv import load_dotenv

load_dotenv(verbose=True)

SECRET_KEY = os.getenv('SECRET_KEY')
```

`verbose=True`로 env 안의 값의 누락 여부를 경고로 띄울 수 있습니다. [참고](https://saurabh-kumar.com/python-dotenv/reference/dotenv/)

## 데이터베이스 설정

우선 SQLAlchemy 데이터베이스를 처리하도록 도와주는 Flask-Migrate 를 `pip install Flask-Migrate`으로 설치합니다.

```python
# config.py

## 1. SQLite3 일 경우
import os

BASE_DIR = os.path.dirname(__file__)

SQLALCHEMY_DATABASE_URI = 'sqlite:///{}'.format(os.path.join(BASE_DIR, 'database.db'))
## 2, AWS RDS for MySQL 일 경우
SQLALCHEMY_DATABASE_URI = (
    "mysql+pymysql://{username}:{password}@{db_host}:{db_port}/{db_name}".format(
        username=os.getenv("RDS_USERNAME"),
        password=os.getenv("RDS_PASSWORD"),
        db_host=os.getenv("RDS_HOST"),
        db_port=os.getenv("RDS_PORT"),
        db_name=os.getenv("RDS_DB_NAME"),
    )
)
```

```python
# app/__init__.py
from flask import Flask
from flask_migrate import Migrate
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()
migrate = Migrate()

def create_app():
  db.init_app(app)
  migrate.init_app(app, db)
```

### 모델

models.py 에 모델 작성하기을 작성합니다. 데이터 타입은 [Column and Data Types](https://docs.sqlalchemy.org/en/14/core/type_basics.html)을 참고하시면 됩니다.

```python
from app import db

class User(db.Model):
  id = db.Column(db.Integer, primary_key=True)
  email = db.Column(db.String(200), nullable=False)
  password = db.Column(db.String(200), nullable=False)
  created_at = db.Column(db.DateTime(), nullable=False)
```

모델을 기반으로 실제 데이터베이스를 생성합니다.

1. `flask db init`을 최초로 실행하여 데이터베이스를 초기화.
2. `flask db migrate` -> 데이터베이스 변경을 위한 리비전 파일 생성
3. `flask db upgrade` -> 리비전 파일 실행 -> .db 파일 생성.

앞으로 새로운 모델을 만들 때마다 `flask db migrate`, `flask db upgrade`을 실행해서 db 파일을 갱신합니다.

### 데이터 조회

[SQLAlchemy 공식 문서](https://docs.sqlalchemy.org/en/13/orm/query.html)

#### 1. 특정 키워드로 검색하기

`filter`, `filter_by` 두 메소드로 검색할 수 있습니다. `filter_by`는 컬럼의 이름을 인자로 바로 입력할 수 있고, `filter`는 다양한 조건을 추가할 수 있습니다.

```python
# filter_by
User.query.filter_by(email='test@gmail.com').all()
# filter
User.query.filter(User.email == 'test@gmail.com').all()
```

`like`를 추가하여 일부 문자열이 들어간 값도 조회할 수 있습니다.

```python
User.query.filter(User.email.like('%{}%'.format('test')))
```

join 으로 두 테이블을 합친 후에 검색할 수 있습니다. 예시로 게시글 제목에 get 이 들어가는 동시에 유저의 아이디가 10인 게시글을 검색합니다.

```python
Board.query.join(User).filter(
  User.id == 10),
  Board.title.like('%{}%'.format('get'))
).all()
```

마지막의 `all()`, `first()` 메소드는 전체를 보여줄 것인지, 아니면 그 중 하나만을 보여줄 것인지를 설정합니다. 만약 특정한 한 개만 조회하는 것이라면 `first()`를 사용합니다.

#### 2. 목록 조회하기

이번에는 pagination 기능을 포함하여 게시글 목록을 조회합니다.

```python
page = args['page']
board_list = Board.query.order_by(Board.create_date.desc())
board_list = board_list.paginate(page, per_page=10)
```

게시글의 작성일에 대한 내림차순으로 정렬하고, `paginate`를 이용해 특정 페이지의 목록을 가져옵니다.

#### 데이터를 클라이언트로 전송하기 위한 변환 과정

주의할 점은, 처음 `all()`, `first()`로 얻은 결과는 딕셔너리가 아닌 SQLAlchemy 타입입니다. 이 타입 그대로 응답으로 보낼 경우 `Object of type Board is not JSON serializable` 에러가 발생합니다.

또한, SQLAlchemy 형식의 데이터에서 특정 컬럼의 값을 가져오려고 할 떄도 `Board object is not subscriptable` 에러가 발생합니다.

```python
board = Board.query.filter(Board.title.like('%{}%'.format('one')))
# created_at 컬럼의 값을 조회할 때 에러가 발생합니다.
print(board.created_at)
```

이것을 해결하기 위해 데이터의 타입을 딕셔너리로 변경할 필요가 있습니다. stack overflow ["How to convert SQLAlchemy row object to a Python dict?"](https://stackoverflow.com/a/1960546)을 참고했습니다.

```python
board_dict = dict()

for column in board.__table__.columns:
  dict[column.name] = getattr(board, column.name)
```

만일 목록일 경우, 반복문을 두 번 반복하여 딕셔너리로 만든 게시글을 리스트에 추가합니다.

```python
result = list()
for board in board_list:
    board_dict = dict()
    for column in board.__table__.columns:
        board_dict[column.name] = getattr(board, column.name)
    result.append(board_dict)
```

`getattr` 함수로 board 객체로부터 column.name 속성의 값을 구하고, 임의의 딕셔너리에 저장합니다.

### 데이터 생성

예시로 유저를 생성하겠습니다.

```python
# users/users.service.ts
import datetime import datetime
from werkzeug.security import generate_password_hash
from ..models import User
from .. import db

def create_user(args):
  new_password = generate_password_hash(args.password)
  new_user = User(
    email=args.email,
    password=new_password,
    created_at=datetime.now()
  )
  db.session.add(new_user)
  db.session.commit()

  return new_user
```

`commit()`까지 완료해야 실제 데이터베이스에 입력한 정보를 저장합니다.

## flask_restx 패키지로 REST API 설정하기

REST API 를 제작하는 것을 돕는 flask_restx 패키지를 `pip install flask_restx`으로 설치합니다. 라우트 설정, 요청 값을 검증하고 가져오는 일 등 다양한 작업을 할 수 있습니다.

### 컨트롤러 생성하기

`@ns.route`로 URI 를 설정합니다. 참고로, 단일 URI 를 할 때는 빈 문자열을 입력해야 합니다. 그러지 않으면 아래와 같은 에러가 발생합니다.

> The URL was defined with a trailing slash so Flask will automatically redirect to the URL with the trailing slash if it was accessed without one.

```python
# app/users/users_controller.ts

from flask import jsonify, make_response, json
from flask_restx import Namespace, Resource, reqparse
from .users_service import UsersService

ns = Namespace("users")

@ns.route("")
class CreateUser(Resource):
    def __init__(self, *args, **kwargs):
      super().__init__(*args, **kwargs)
      self.service = UsersService()

    def post(self):
      parser = reqparse.RequestParser()
      parser.add_argument('email', type=str)
      parser.add_argument('password', type=str)
      args = parser.parse_args()

      new_user = service.create_user(args)
      return test_list

```

여기서 가장 눈여겨볼 것은 `parser`입니다. 플라스크의 Request 객체의 값에 접근하기 위해 사용합니다. `parser.add_argument()`으로 어떤 값을 찾아와야 하는지를 미리 설정해야합니다.

### 라우트 설정

플라스크 앱에서 라우팅을 설정합니다.

```python
# app/__init__.py

from flask import Flask
from flask_restx import Api

def create_app():
    app = Flask(__name__)
    app.debug = True
    app.config.from_object(config)

    # ORM
    db.init_app(app)
    migrate.init_app(app, db)
    from . import models

    # flask_restx
    api = Api(app)


    from .users import users_controller as user

    api.add_namespace(user.ns, "/users")

    return app
```

## 참고 문서

- [플라스크 문서 한글판](https://flask-docs-kr.readthedocs.io/ko/latest/)
- [점프 투 플라스크](https://wikidocs.net/book/4542)
