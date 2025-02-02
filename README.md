## FastAPI Best Practices <!-- omit from toc -->
스타트업에서 사용해본 주관적인 모범 사례와 관례들의 리스트입니다.

지난 몇 년간 프로덕션 환경에서 사용하면서, 개발 경험에 크게 영향을 준 좋은 결정들과 나쁜 결정들을 해왔습니다. 그중 일부는 공유할 가치가 있다고 생각됩니다. 

[해당 구조 예시 레포](https://github.com/zhanymkanov/fastapi_production_template/tree/9d548b212232b41e101b36b714e144e4e26260b0)

## 목차  <!-- omit from toc -->
- [프로젝트 구조](#프로젝트-구조)
- [비동기 라우트(Async Routes)](#비동기-라우트async-routes)
  - [I/O 집중적인 작업(I/O Intensive Tasks)](#io-집중적인-작업io-intensive-tasks)
  - [CPU 집중적인 작업(CPU Intensive Tasks)](#cpu-집중적인-작업cpu-intensive-tasks)
- [Pydantic](#pydantic)
  - [Pydantic을 적극적으로 사용하기(Excessively use Pydantic)](#pydantic을-적극적으로-사용하기excessively-use-pydantic)
  - [커스텀 베이스 모델(Custom Base Model)](#커스텀-베이스-모델custom-base-model)
  - [Pydantic BaseSettings 분리하기(Decouple Pydantic BaseSettings)](#pydantic-basesettings-분리하기decouple-pydantic-basesettings)
- [Dependencies](#dependencies)
  - [의존성 주입 그 이상(Beyond Dependency Injection)](#의존성-주입-그-이상beyond-dependency-injection)
  - [체인된 의존성(Chain Dependencies)](#체인된-의존성chain-dependencies)
  - [의존성 분리 & 재활용하기. 의존성 호출은 캐시됨(Decouple & Reuse dependencies. Dependency calls are cached)](#의존성-분리--재활용하기-의존성-호출은-캐시됨decouple--reuse-dependencies-dependency-calls-are-cached)
  - [가능하면 비동기 의존성을 선호하기(Prefer async dependencies)](#가능하면-비동기-의존성을-선호하기prefer-async-dependencies)
- [기타(Miscellaneous)](#기타miscellaneous)
  - [REST 규칙 따르기(Follow the REST)](#rest-규칙-따르기follow-the-rest)
  - [FastAPI 응답 직렬화(FastAPI response serialization)](#fastapi-응답-직렬화fastapi-response-serialization)
  - [반드시 동기 SDK를 사용해야 한다면, 쓰레드 풀에서 실행하기(If you must use sync SDK, then run it in a thread pool.)](#반드시-동기-sdk를-사용해야-한다면-쓰레드-풀에서-실행하기if-you-must-use-sync-sdk-then-run-it-in-a-thread-pool)
  - [ValueError가 Pydantic ValidationError가 될 수 있음(ValueErrors might become Pydantic ValidationError)](#valueerror가-pydantic-validationerror가-될-수-있음valueerrors-might-become-pydantic-validationerror)
  - [문서(Docs)](#문서docs)
  - [DB 키 네이밍 규칙 정하기(Set DB keys naming conventions)](#db-키-네이밍-규칙-정하기set-db-keys-naming-conventions)
  - [마이그레이션(Alembic)](#마이그레이션-alembic)
  - [DB 네이밍 컨벤션 정하기(Set DB naming conventions)](#db-네이밍-컨벤션-정하기set-db-naming-conventions)
  - [SQL이 우선, 그 뒤에 Pydantic(SQL-first. Pydantic-second)](#sql이-우선-그-뒤에-pydanticsql-first-pydantic-second)
  - [테스트 클라이언트를 처음부터 비동기로 설정하기(Set tests client async from day 0)](#테스트-클라이언트를-처음부터-비동기로-설정하기set-tests-client-async-from-day-0)
  - [Ruff 사용하기(Use ruff)](#ruff-사용하기use-ruff)
- [보너스 섹션(Bonus Section)](#보너스-섹션bonus-section)

---

## 프로젝트 구조
프로젝트를 구조화하는 방법은 다양하지만, 가장 좋은 구조는 일관되고 직관적이며, 예측 가능한 구조라고 생각합니다.

예시 프로젝트나 튜토리얼에서는 흔히 (예: crud, routers, models) 파일 유형별로 프로젝트를 구분합니다. 이런 방식은 마이크로서비스나 스코프가 적은 프로젝트에는 잘 맞지만, 많은 도메인과 모듈을 가진 모놀리식 구조에는 잘 맞지 않을 때가 있었습니다.

아래 구조는 [Netflix의 Dispatch](https://github.com/Netflix/dispatch)를 참고하여, 규모가 크고 확장 가능한 프로젝트에 적용하기 위해 몇 가지 수정을 가미한 예시입니다.

```
fastapi-project
├── alembic/
├── src
│   ├── auth
│   │   ├── router.py
│   │   ├── schemas.py  # pydantic 모델
│   │   ├── models.py  # db 모델
│   │   ├── dependencies.py
│   │   ├── config.py  # 모듈별 설정
│   │   ├── constants.py
│   │   ├── exceptions.py
│   │   ├── service.py
│   │   └── utils.py
│   ├── aws
│   │   ├── client.py  # 외부 서비스 통신을 위한 클라이언트 모델
│   │   ├── schemas.py
│   │   ├── config.py
│   │   ├── constants.py
│   │   ├── exceptions.py
│   │   └── utils.py
│   └── posts
│   │   ├── router.py
│   │   ├── schemas.py
│   │   ├── models.py
│   │   ├── dependencies.py
│   │   ├── constants.py
│   │   ├── exceptions.py
│   │   ├── service.py
│   │   └── utils.py
│   ├── config.py  # 전역 설정
│   ├── models.py  # 전역 DB 모델
│   ├── exceptions.py  # 전역 예외
│   ├── pagination.py  # 전역 모듈(예: 페이징)
│   ├── database.py  # DB 연결 관련
│   └── main.py
├── tests/
│   ├── auth
│   ├── aws
│   └── posts
├── templates/
│   └── index.html
├── requirements
│   ├── base.txt
│   ├── dev.txt
│   └── prod.txt
├── .env
├── .gitignore
├── logging.ini
└── alembic.ini
```

1. 모든 도메인 디렉터리는 `src` 폴더 내에 저장  
   1. `src/` - 앱의 최상위 레벨, 공통 모델, 설정, 상수 등 포함  
   2. `src/main.py` - FastAPI 앱을 초기화하는 프로젝트의 루트
2. 각 패키지는 고유의 `router`, `schemas`, `models` 등을 가짐  
   1. `router.py` - 각 모듈의 핵심, 모든 엔드포인트 포함  
   2. `schemas.py` - pydantic 모델  
   3. `models.py` - DB 모델  
   4. `service.py` - 모듈별 비즈니스 로직  
   5. `dependencies.py` - 라우터 의존성  
   6. `constants.py` - 모듈별 상수와 에러 코드  
   7. `config.py` - 예: 환경 변수  
   8. `utils.py` - 비즈니스 로직이 아닌 함수, 예: 응답 정규화, 데이터 변환 등  
   9. `exceptions.py` - 모듈별 예외, 예: `PostNotFound`, `InvalidUserData`
3. 패키지가 다른 패키지의 서비스나 의존성, 상수가 필요하다면 명시적인 모듈 이름으로 임포트  
   ```python
   from src.auth import constants as auth_constants
   from src.notifications import service as notification_service
   from src.posts.constants import ErrorCode as PostsErrorCode  # 예: 각 패키지의 constants 모듈에 공통 ErrorCode가 있다고 가정
   ```

---

## 비동기 라우트(Async Routes)
FastAPI는 애초에 비동기(Async) 프레임워크로 설계되었습니다. 비동기 I/O 작업에 특화되어 있어 빠른 속도를 낼 수 있습니다.

하지만 FastAPI는 비동기 라우트만을 강제하지 않으며, 동기 라우트(sync routes)도 지원합니다. 이는 초보 개발자에게 비동기와 동기를 혼동하게 만들 수 있으나, 둘은 실제로 다르게 동작합니다.

### I/O 집중적인 작업(I/O Intensive Tasks)
FastAPI 내부적으로는 [동기와 비동기 I/O 작업을 모두 효과적으로 처리](https://fastapi.tiangolo.com/async/#path-operation-functions)할 수 있습니다.
- 동기 라우트는 [threadpool](https://en.wikipedia.org/wiki/Thread_pool)에서 실행되므로, 블로킹 I/O가 이벤트 루프를 멈추지 않습니다.
- 비동기 라우트를 사용하면 `await`로 호출하며, FastAPI는 개발자가 블로킹 I/O를 수행하지 않을 것이라고 가정합니다.

문제는 이 가정을 깨고 비동기 라우트 안에서 블로킹 연산을 실행할 때 발생합니다. 비동기 라우트 내에서 `time.sleep()` 같은 블로킹 연산을 수행하면, 그 동안 이벤트 루프가 멈춰 다른 작업을 처리할 수 없습니다.

```python
import asyncio
import time

from fastapi import APIRouter


router = APIRouter()


@router.get("/terrible-ping")
async def terrible_ping():
    time.sleep(10)  # I/O를 블로킹하는 10초짜리 작업. 전체 프로세스가 멈춤
    
    return {"pong": True}

@router.get("/good-ping")
def good_ping():
    time.sleep(10)  # 10초짜리 블로킹 작업이지만, 이 라우트 전체가 별도의 스레드에서 돌아감

    return {"pong": True}

@router.get("/perfect-ping")
async def perfect_ping():
    await asyncio.sleep(10)  # 논블로킹 I/O 작업

    return {"pong": True}
```

**각 라우트 호출 시**:
1. `GET /terrible-ping`
   1. FastAPI 서버가 요청을 받아서 처리 시작
   2. `time.sleep(10)`이 실행되는 동안 이벤트 루프와 큐에 대기 중인 모든 작업이 멈춤  
      - 서버는 `time.sleep()`을 I/O 작업으로 간주하지 않으므로, 완전히 끝날 때까지 기다림  
      - 그동안 새로운 요청을 받지 못함
   3. 작업이 끝난 후 응답을 반환  
      - 응답이 끝난 뒤에야 서버가 새 요청을 처리하기 시작
2. `GET /good-ping`
   1. FastAPI 서버가 요청을 받아서 처리 시작
   2. FastAPI가 전체 `good_ping` 라우트 함수를 threadpool로 보냄. 별도의 워커 스레드가 함수 실행
   3. `good_ping`이 실행되는 동안, 메인 이벤트 루프는 다른 작업(예: 새 요청, DB 호출 등)을 계속 처리  
      - 워커 스레드는 `time.sleep`이 끝날 때까지 대기하지만, 메인 스레드(즉, FastAPI 앱)는 계속 동작
   4. `good_ping`이 끝나면, 서버는 클라이언트에 응답 반환
3. `GET /perfect-ping`
   1. FastAPI 서버가 요청을 받아 처리 시작
   2. `await asyncio.sleep(10)`을 통해 논블로킹 I/O 작업을 수행
   3. 이벤트 루프는 대기열의 다음 작업을 선택하여 처리(예: 새 요청, DB 호출 등)
   4. 10초 대기가 끝나면, 라우트 실행을 마무리하고 클라이언트에 응답 반환

> [!WARNING]
> **스레드 풀에 대한 주의**:
> - 스레드는 코루틴보다 더 많은 리소스를 필요로 하므로, async I/O보다 비용이 큽니다.
> - 스레드 풀에는 스레드 수에 제한이 있어, 너무 많이 사용하면 스레드가 고갈되어 앱이 느려질 수 있습니다. [자세히 읽기](https://github.com/Kludex/fastapi-tips?tab=readme-ov-file#2-be-careful-with-non-async-functions) (외부 링크)

### CPU 집중적인 작업(CPU Intensive Tasks)
두 번째 주의점은, `await`나 thread pool에서 실행하는 작업은 I/O 작업에 대해서만 이점이 있다는 것입니다.
- CPU 연산이 많은 작업(예: 복잡한 계산, 데이터 처리, 동영상 트랜스코딩 등)을 `await`로 감싸도, CPU가 결국 일을 해야 하므로 비동기의 이점이 없습니다. I/O 작업처럼 대기 중에 CPU를 다른 작업에 할당할 수 있는 게 아니기 때문입니다.
- CPU 집약적인 작업을 다른 스레드에서 돌려도 [GIL(Global Interpreter Lock)](https://realpython.com/python-gil/)의 영향으로 성능 향상을 기대하기 어렵습니다.
- CPU 집약적 작업을 최적화하고 싶다면, 프로세스를 분리해 별도 워커(프로세스)로 보내는 방식을 고려해야 합니다.

**혼란을 겪는 사람들이 올린 관련 StackOverflow 질문들**
1. <https://stackoverflow.com/questions/62976648/architecture-flask-vs-fastapi/70309597#70309597>  
   - [제 답변](https://stackoverflow.com/a/70309597/6927498)도 여기서 보실 수 있습니다.
2. <https://stackoverflow.com/questions/65342833/fastapi-uploadfile-is-slow-compared-to-flask>
3. <https://stackoverflow.com/questions/71516140/fastapi-runs-api-calls-in-serial-instead-of-parallel-fashion>

---

## Pydantic
### Pydantic을 적극적으로 사용하기(Excessively use Pydantic)
Pydantic은 데이터 유효성 검사와 변환에 대해 풍부한 기능을 제공합니다.

기본적인 기능(필수/비필수 필드, 기본값) 외에도, 정규식(regex), enum, 문자열 조작, 이메일 유효성 검사 등과 같은 다양한 데이터 처리 도구를 내장하고 있습니다.

```python
from enum import Enum
from pydantic import AnyUrl, BaseModel, EmailStr, Field


class MusicBand(str, Enum):
   AEROSMITH = "AEROSMITH"
   QUEEN = "QUEEN"
   ACDC = "AC/DC"


class UserBase(BaseModel):
    first_name: str = Field(min_length=1, max_length=128)
    username: str = Field(min_length=1, max_length=128, pattern="^[A-Za-z0-9-_]+$")
    email: EmailStr
    age: int = Field(ge=18, default=None)  # 18세 이상이어야 함
    favorite_band: MusicBand | None = None  # "AEROSMITH", "QUEEN", "AC/DC" 중 하나만 가능
    website: AnyUrl | None = None
```

### 커스텀 베이스 모델(Custom Base Model)
글로벌하게 통제 가능한 베이스 모델을 두면, 앱 전체의 모델을 커스터마이징하기 쉽습니다. 예를 들어, 표준 날짜 시간 형식을 일괄 적용하거나 서브클래스 전부에서 공통 메서드를 추가할 수 있습니다.

```python
from datetime import datetime
from zoneinfo import ZoneInfo

from fastapi.encoders import jsonable_encoder
from pydantic import BaseModel, ConfigDict


def datetime_to_gmt_str(dt: datetime) -> str:
    if not dt.tzinfo:
        dt = dt.replace(tzinfo=ZoneInfo("UTC"))

    return dt.strftime("%Y-%m-%dT%H:%M:%S%z")


class CustomModel(BaseModel):
    model_config = ConfigDict(
        json_encoders={datetime: datetime_to_gmt_str},
        populate_by_name=True,
    )

    def serializable_dict(self, **kwargs):
        """직렬화 가능한 필드만 포함하는 dict를 반환."""
        default_dict = self.model_dump()

        return jsonable_encoder(default_dict)
```

위 예시에서, 전역 베이스 모델은 다음을 수행합니다:
- 모든 `datetime` 필드를 일관된 포맷(명시적 타임존 포함)으로 직렬화
- 직렬화 가능한 필드만 담긴 dict를 반환하는 메서드 제공

### Pydantic BaseSettings 분리하기(Decouple Pydantic BaseSettings)
`BaseSettings`는 환경 변수를 읽어오는 혁신적인 기능을 제공하지만, 전체 앱을 하나의 `BaseSettings`로 관리하면 시간이 지날수록 복잡해지고 유지 보수가 어려워질 수 있습니다.  
이를 개선하기 위해 모듈과 도메인별로 `BaseSettings`를 분리해 사용할 수 있습니다.

```python
# src.auth.config
from datetime import timedelta

from pydantic_settings import BaseSettings


class AuthConfig(BaseSettings):
    JWT_ALG: str
    JWT_SECRET: str
    JWT_EXP: int = 5  # 분(minute)

    REFRESH_TOKEN_KEY: str
    REFRESH_TOKEN_EXP: timedelta = timedelta(days=30)

    SECURE_COOKIES: bool = True


auth_settings = AuthConfig()
```

```python
# src.config
from pydantic import PostgresDsn, RedisDsn, model_validator
from pydantic_settings import BaseSettings

from src.constants import Environment


class Config(BaseSettings):
    DATABASE_URL: PostgresDsn
    REDIS_URL: RedisDsn

    SITE_DOMAIN: str = "myapp.com"

    ENVIRONMENT: Environment = Environment.PRODUCTION

    SENTRY_DSN: str | None = None

    CORS_ORIGINS: list[str]
    CORS_ORIGINS_REGEX: str | None = None
    CORS_HEADERS: list[str]

    APP_VERSION: str = "1.0"


settings = Config()
```

---

## Dependencies
### 의존성 주입 그 이상(Beyond Dependency Injection)
Pydantic은 훌륭한 스키마 검증 도구이지만, DB나 외부 서비스를 호출해야 하는 복잡한 검증에는 한계가 있습니다.

FastAPI 문서에서는 주로 라우트에서 의존성을 DI(Dependency Injection)하는 예시를 보여주지만, 사실 의존성은 **요청 검증**에도 훌륭하게 활용할 수 있습니다.

DB 제약 조건(예: 이메일 중복 여부, 사용자 존재 여부 등)을 확인하는 로직은 의존성으로 분리해 재사용할 수 있습니다.

```python
# dependencies.py
async def valid_post_id(post_id: UUID4) -> dict[str, Any]:
    post = await service.get_by_id(post_id)
    if not post:
        raise PostNotFound()

    return post


# router.py
@router.get("/posts/{post_id}", response_model=PostResponse)
async def get_post_by_id(post: dict[str, Any] = Depends(valid_post_id)):
    return post


@router.put("/posts/{post_id}", response_model=PostResponse)
async def update_post(
    update_data: PostUpdate,  
    post: dict[str, Any] = Depends(valid_post_id), 
):
    updated_post = await service.update(id=post["id"], data=update_data)
    return updated_post


@router.get("/posts/{post_id}/reviews", response_model=list[ReviewsResponse])
async def get_post_reviews(post: dict[str, Any] = Depends(valid_post_id)):
    post_reviews = await reviews_service.get_by_post_id(post["id"])
    return post_reviews
```

`valid_post_id` 같은 검증 로직을 의존성으로 만들지 않는다면, 각 엔드포인트마다 `post_id`가 존재하는지 반복해서 확인하고 동일한 테스트 코드도 중복 작성해야 했을 것입니다.

### 체인된 의존성(Chain Dependencies)
의존성은 다른 의존성을 사용할 수 있으므로, 유사한 로직을 중복 작성하지 않고 체이닝할 수 있습니다.

```python
# dependencies.py
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt

async def valid_post_id(post_id: UUID4) -> dict[str, Any]:
    post = await service.get_by_id(post_id)
    if not post:
        raise PostNotFound()

    return post


async def parse_jwt_data(
    token: str = Depends(OAuth2PasswordBearer(tokenUrl="/auth/token"))
) -> dict[str, Any]:
    try:
        payload = jwt.decode(token, "JWT_SECRET", algorithms=["HS256"])
    except JWTError:
        raise InvalidCredentials()

    return {"user_id": payload["id"]}


async def valid_owned_post(
    post: dict[str, Any] = Depends(valid_post_id), 
    token_data: dict[str, Any] = Depends(parse_jwt_data),
) -> dict[str, Any]:
    if post["creator_id"] != token_data["user_id"]:
        raise UserNotOwner()

    return post

# router.py
@router.get("/users/{user_id}/posts/{post_id}", response_model=PostResponse)
async def get_user_post(post: dict[str, Any] = Depends(valid_owned_post)):
    return post
```

### 의존성 분리 & 재활용하기. 의존성 호출은 캐시됨(Decouple & Reuse dependencies. Dependency calls are cached)
의존성은 여러 번 재사용되더라도 한 번만 계산됩니다. 즉, FastAPI는 기본적으로 의존성의 반환 값을 요청 범위 내에서 캐시합니다.  
`valid_post_id`가 한 라우트에서 여러 번 호출되어도, 실제로는 한 번만 DB를 조회합니다.

이 사실을 활용해 의존성을 여러 작은 함수로 나누고, 더 작은 범위를 다루도록 설계하면, 재사용성이 향상됩니다.  
아래 예시에서는 `parse_jwt_data`가 세 번 호출되는 구조처럼 보이지만, 실제로는 최초 한 번만 호출됩니다.

```python
# dependencies.py
from fastapi import BackgroundTasks
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt

async def valid_post_id(post_id: UUID4) -> Mapping:
    post = await service.get_by_id(post_id)
    if not post:
        raise PostNotFound()

    return post


async def parse_jwt_data(
    token: str = Depends(OAuth2PasswordBearer(tokenUrl="/auth/token"))
) -> dict:
    try:
        payload = jwt.decode(token, "JWT_SECRET", algorithms=["HS256"])
    except JWTError:
        raise InvalidCredentials()

    return {"user_id": payload["id"]}


async def valid_owned_post(
    post: Mapping = Depends(valid_post_id), 
    token_data: dict = Depends(parse_jwt_data),
) -> Mapping:
    if post["creator_id"] != token_data["user_id"]:
        raise UserNotOwner()

    return post


async def valid_active_creator(
    token_data: dict = Depends(parse_jwt_data),
):
    user = await users_service.get_by_id(token_data["user_id"])
    if not user["is_active"]:
        raise UserIsBanned()
    
    if not user["is_creator"]:
       raise UserNotCreator()
    
    return user
        

# router.py
@router.get("/users/{user_id}/posts/{post_id}", response_model=PostResponse)
async def get_user_post(
    worker: BackgroundTasks,
    post: Mapping = Depends(valid_owned_post),
    user: Mapping = Depends(valid_active_creator),
):
    """현재 활성화된 크리에이터의 게시물을 가져옵니다."""
    worker.add_task(notifications_service.send_email, user["id"])
    return post
```

### 가능하면 비동기 의존성을 선호하기(Prefer async dependencies)
FastAPI는 동기와 비동기 의존성을 모두 지원합니다. 단순히 `await`가 없다고 해서 동기 의존성을 사용하는 것은 권장되지 않을 수 있습니다.

라우트와 마찬가지로, 동기 의존성은 thread pool에서 실행되므로, 스레드 사용에 따른 비용과 제한이 불필요하게 발생할 수 있습니다.

[참고](https://github.com/Kludex/fastapi-tips?tab=readme-ov-file#9-your-dependencies-may-be-running-on-threads) (외부 링크)

---

## 기타(Miscellaneous)
### REST 규칙 따르기(Follow the REST)
RESTful API를 디자인하면, 다음과 같은 라우트들에서 의존성을 재활용하기가 더 쉽습니다.
1. `GET /courses/:course_id`
2. `GET /courses/:course_id/chapters/:chapter_id/lessons`
3. `GET /chapters/:chapter_id`

유일한 주의점은 경로에서 동일한 변수를 사용할 때입니다:
- `GET /profiles/:profile_id`, `GET /creators/:creator_id` 처럼, 둘 다 `profile_id`가 존재하는지 검증해야 하지만 `GET /creators/:creator_id`에서는 추가로 "해당 프로필이 크리에이터인지"를 검사해야 할 수도 있습니다.
- 이 경우 `creator_id` 대신 `profile_id`라는 동일한 경로 변수를 쓰고, `profile_id`를 검증하는 의존성과 `profile_id`가 실제로 크리에이터인지 검사하는 의존성을 체인(Chain)하면 재사용성이 높아집니다.

```python
# src.profiles.dependencies
async def valid_profile_id(profile_id: UUID4) -> Mapping:
    profile = await service.get_by_id(profile_id)
    if not profile:
        raise ProfileNotFound()

    return profile

# src.creators.dependencies
async def valid_creator_id(profile: Mapping = Depends(valid_profile_id)) -> Mapping:
    if not profile["is_creator"]:
       raise ProfileNotCreator()

    return profile

# src.profiles.router.py
@router.get("/profiles/{profile_id}", response_model=ProfileResponse)
async def get_user_profile_by_id(profile: Mapping = Depends(valid_profile_id)):
    """profile_id로 프로필 조회"""
    return profile

# src.creators.router.py
@router.get("/creators/{profile_id}", response_model=ProfileResponse)
async def get_user_profile_by_id(
     creator_profile: Mapping = Depends(valid_creator_id)
):
    """profile_id가 크리에이터 프로필인지 조회"""
    return creator_profile
```

### FastAPI 응답 직렬화(FastAPI response serialization)
`response_model`과 동일한 Pydantic 객체를 직접 반환하면, 성능 최적화가 될 것으로 생각하기 쉽지만 그렇지 않습니다.

FastAPI는 먼저 해당 Pydantic 객체를 `jsonable_encoder`로 dict 형태로 변환하고, 그 결과를 다시 `response_model`로 검증한 뒤 최종적으로 JSON으로 직렬화합니다.

```python
from fastapi import FastAPI
from pydantic import BaseModel, root_validator

app = FastAPI()


class ProfileResponse(BaseModel):
    @root_validator(pre=False)
    def debug_usage(cls, values):
        print("created pydantic model")
        return values


@app.get("/", response_model=ProfileResponse)
async def root():
    return ProfileResponse()
```

**로그 출력 예시**:
```
[INFO] [2022-08-28 12:00:00.000000] created pydantic model
[INFO] [2022-08-28 12:00:00.000020] created pydantic model
```

즉, Pydantic 객체가 최소 두 번 만들어집니다.

### 반드시 동기 SDK를 사용해야 한다면, 쓰레드 풀에서 실행하기(If you must use sync SDK, then run it in a thread pool.)
외부 서비스와 통신하기 위해 동기 라이브러리를 사용해야 하는 상황이라면, 별도의 워커 스레드에서 HTTP 요청을 실행해야 합니다.

`starlette`의 `run_in_threadpool`을 사용하는 방법이 널리 알려져 있습니다.

```python
from fastapi import FastAPI
from fastapi.concurrency import run_in_threadpool
from my_sync_library import SyncAPIClient 

app = FastAPI()


@app.get("/")
async def call_my_sync_library():
    my_data = await service.get_my_data()

    client = SyncAPIClient()
    await run_in_threadpool(client.make_request, data=my_data)
```

### ValueError가 Pydantic ValidationError가 될 수 있음(ValueErrors might become Pydantic ValidationError)
클라이언트가 직접 사용하게 되는 Pydantic 스키마에서 `ValueError`를 발생시키면, FastAPI가 이를 깔끔한 ValidationError 응답으로 변환해 줍니다.

```python
# src.profiles.schemas
from pydantic import BaseModel, field_validator
import re

STRONG_PASSWORD_PATTERN = r"^(?=.*[A-Z])(?=.*[a-z])(?=.*\d)(?=.*[\W_]).+$"

class ProfileCreate(BaseModel):
    username: str
    password: str

    @field_validator("password", mode="after")
    @classmethod
    def valid_password(cls, password: str) -> str:
        if not re.match(STRONG_PASSWORD_PATTERN, password):
            raise ValueError(
                "Password must contain at least "
                "one lower character, "
                "one upper character, "
                "digit or "
                "special symbol"
            )
        return password
```

```python
# src.profiles.routes
from fastapi import APIRouter

router = APIRouter()

@router.post("/profiles")
async def get_creator_posts(profile_data: ProfileCreate):
   pass
```

**예시 응답**:

<img src="images/value_error_response.png" width="400" height="auto">

---

### 문서(Docs)
1. API가 외부 공개가 아니라면, 기본적으로 문서를 비활성화하세요. 특정 환경에서만 문서를 노출하도록 설정할 수 있습니다.
   ```python
   from fastapi import FastAPI
   from starlette.config import Config

   config = Config(".env")  # .env 파일에서 환경 변수 파싱

   ENVIRONMENT = config("ENVIRONMENT")  # 현재 환경
   SHOW_DOCS_ENVIRONMENT = ("local", "staging")  # 문서를 노출할 환경 목록

   app_configs = {"title": "My Cool API"}
   if ENVIRONMENT not in SHOW_DOCS_ENVIRONMENT:
      app_configs["openapi_url"] = None  # docs URL을 비활성화

   app = FastAPI(**app_configs)
   ```

2. FastAPI가 이해하기 쉬운 문서를 생성하도록 돕습니다.  
   - `response_model`, `status_code`, `description` 등을 잘 설정하세요.  
   - 모델과 상태 코드가 다양하다면, `responses` 라우트 속성을 사용해 다른 응답별로 문서를 추가하세요.
   ```python
   from fastapi import APIRouter, status

   router = APIRouter()

   @router.post(
       "/endpoints",
       response_model=DefaultResponseModel,  # 기본 응답 pydantic 모델
       status_code=status.HTTP_201_CREATED,  # 기본 상태 코드
       description="잘 문서화된 엔드포인트에 대한 설명",
       tags=["Endpoint Category"],
       summary="엔드포인트 요약",
       responses={
           status.HTTP_200_OK: {
               "model": OkResponse,  # 200 응답에 대한 커스텀 pydantic 모델
               "description": "Ok 응답",
           },
           status.HTTP_201_CREATED: {
               "model": CreatedResponse,  # 201 응답에 대한 커스텀 pydantic 모델
               "description": "유저 요청을 통해 무언가를 생성",
           },
           status.HTTP_202_ACCEPTED: {
               "model": AcceptedResponse,  # 202 응답에 대한 커스텀 pydantic 모델
               "description": "요청을 받아들이고, 나중에 처리",
           },
       },
   )
   async def documented_route():
       pass
   ```
   이렇게 하면 다음과 같은 문서가 생성됩니다:
   ![FastAPI Generated Custom Response Docs](images/custom_responses.png "Custom Response Docs")

### DB 키 네이밍 규칙 정하기(Set DB keys naming conventions)
SQLAlchemy의 기본 네이밍보다, DB 컨벤션에 따라 인덱스나 제약 조건 이름을 명시적으로 설정하는 것이 좋습니다.

```python
from sqlalchemy import MetaData

POSTGRES_INDEXES_NAMING_CONVENTION = {
    "ix": "%(column_0_label)s_idx",
    "uq": "%(table_name)s_%(column_0_name)s_key",
    "ck": "%(table_name)s_%(constraint_name)s_check",
    "fk": "%(table_name)s_%(column_0_name)s_fkey",
    "pk": "%(table_name)s_pkey",
}
metadata = MetaData(naming_convention=POSTGRES_INDEXES_NAMING_CONVENTION)
```

### 마이그레이션(Alembic)
1. 마이그레이션은 정적이고 되돌릴 수 있어야 합니다.
   - 마이그레이션 과정에 동적으로 생성되는 구조가 들어가면 안 됩니다. (단, 실제 데이터 자체가 동적인 것은 OK)
2. 스크립트 생성 시 이름과 슬러그를 의미 있게 만듭니다. 슬러그는 변경 내용을 간략히 설명해야 합니다.
3. `alembic.ini`에서 파일 템플릿을 `날짜_슬러그.py` 형태로 설정해 가독성을 높일 수 있습니다.

```ini
# alembic.ini
file_template = %%(year)d-%%(month).2d-%%(day).2d_%%(slug)s
```

### DB 네이밍 컨벤션 정하기(Set DB naming conventions)
일관된 이름 사용은 매우 중요합니다. 예를 들면:
1. `lower_case_snake` 사용
2. 단수형(예: `post`, `post_like`, `user_playlist`)
3. 비슷한 테이블은 접두사를 붙여 그룹화(예: `payment_account`, `payment_bill`, `post`, `post_like`)
4. 테이블 간에 일관성을 유지하되, 필요에 따라 구체적 네이밍도 OK  
   - 예: 대부분의 테이블은 `profile_id`를 쓰되, 어떤 테이블은 "크리에이터 전용"이면 `creator_id`를 쓰는 식
   - `post_view`, `post_like` 같은 테이블에서는 `post_id`를 사용하되, 특정 모듈에서는 `course_id` 등 맥락에 맞게
5. 날짜 시간 필드는 `_at`, 날짜 필드는 `_date` 접미사 사용

### SQL이 우선, 그 뒤에 Pydantic(SQL-first. Pydantic-second)
- 일반적으로, DB가 데이터 처리를 훨씬 빠르고 간단하게 처리할 수 있습니다.
- 복잡한 JOIN이나 간단한 데이터 변환은 DB 레벨에서 처리하는 편이 낫습니다.
- 계층화된 객체를 응답으로 내보낼 때, DB에서 JSON 형태로 합쳐서 가져오는 방법도 가능합니다.

```python
# src.posts.service
from typing import Any

from pydantic import UUID4
from sqlalchemy import desc, func, select, text
from sqlalchemy.sql.functions import coalesce

from src.database import database, posts, profiles, post_review, products

async def get_posts(
    creator_id: UUID4, *, limit: int = 10, offset: int = 0
) -> list[dict[str, Any]]: 
    select_query = (
        select(
            (
                posts.c.id,
                posts.c.slug,
                posts.c.title,
                func.json_build_object(
                   text("'id', profiles.id"),
                   text("'first_name', profiles.first_name"),
                   text("'last_name', profiles.last_name"),
                   text("'username', profiles.username"),
                ).label("creator"),
            )
        )
        .select_from(posts.join(profiles, posts.c.owner_id == profiles.c.id))
        .where(posts.c.owner_id == creator_id)
        .limit(limit)
        .offset(offset)
        .group_by(
            posts.c.id,
            posts.c.type,
            posts.c.slug,
            posts.c.title,
            profiles.c.id,
            profiles.c.first_name,
            profiles.c.last_name,
            profiles.c.username,
            profiles.c.avatar,
        )
        .order_by(
            desc(coalesce(posts.c.updated_at, posts.c.published_at, posts.c.created_at))
        )
    )
    
    return await database.fetch_all(select_query)

# src.posts.schemas
from typing import Any

from pydantic import BaseModel, UUID4

class Creator(BaseModel):
    id: UUID4
    first_name: str
    last_name: str
    username: str

class Post(BaseModel):
    id: UUID4
    slug: str
    title: str
    creator: Creator
    
# src.posts.router
from fastapi import APIRouter, Depends

router = APIRouter()

@router.get("/creators/{creator_id}/posts", response_model=list[Post])
async def get_creator_posts(creator: dict[str, Any] = Depends(valid_creator_id)):
   posts = await service.get_posts(creator["id"])
   return posts
```

### 테스트 클라이언트를 처음부터 비동기로 설정하기(Set tests client async from day 0)
DB를 연동한 통합 테스트를 작성하다 보면, 이벤트 루프 충돌 등의 문제가 생길 수 있습니다.  
이런 문제를 피하려면 처음부터 비동기 테스트 클라이언트를 세팅하는 것이 좋습니다. 예: [httpx](https://github.com/encode/starlette/issues/652)

```python
import pytest
from async_asgi_testclient import TestClient

from src.main import app  # FastAPI 앱


@pytest.fixture
async def client() -> AsyncGenerator[TestClient, None]:
    host, port = "127.0.0.1", "9000"

    async with AsyncClient(transport=ASGITransport(app=app, client=(host, port)), base_url="http://test") as client:
        yield client


@pytest.mark.asyncio
async def test_create_post(client: TestClient):
    resp = await client.post("/posts")

    assert resp.status_code == 201
```

만약 DB 연결이 동기식이거나, 통합 테스트를 전혀 작성하지 않을 계획이라면 상관없지만, 일반적으론 비동기 테스트 클라이언트를 권장합니다.

### Ruff 사용하기(Use ruff)
코드 스타일에 대해 신경 쓰지 않고 비즈니스 로직에 집중하려면 린터(linter)를 사용하는 것이 좋습니다.

[Ruff](https://github.com/astral-sh/ruff)는 “blazingly-fast”라는 표현 그대로, `black`, `autoflake`, `isort` 등을 대체하고 600개 이상의 린트 규칙을 지원하는 새로운 초고속 린터입니다.

`pre-commit` 훅으로도 많이 쓰이지만, 간단히 스크립트로 사용할 수도 있습니다.

```bash
#!/bin/sh -e
set -x

ruff check --fix src
ruff format src
```

---

## 보너스 섹션(Bonus Section)
몇몇 친절한 분들이 자신의 경험과 모범 사례를 공유해주셨는데, 꼭 읽어볼 가치가 있습니다.  
[Issues](https://github.com/zhanymkanov/fastapi-best-practices/issues) 섹션을 확인해 보세요.

예를 들어, [lowercase00](https://github.com/zhanymkanov/fastapi-best-practices/issues/4) 님은 Permissions & Auth, 클래스 기반 서비스 & 뷰, 작업 큐, 커스텀 응답 직렬화, `dynaconf` 설정 등에 관한 모범 사례를 자세히 설명해주셨습니다.

FastAPI를 사용하면서 겪은 경험(좋은 점, 나쁜 점 모두)이 있다면, 언제든 새 이슈를 만들어 공유해주세요. 읽는 것만으로도 큰 도움이 됩니다.
