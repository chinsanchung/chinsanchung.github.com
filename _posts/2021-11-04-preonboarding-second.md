---
title: "원티드 프리온보딩 코스 두 번째 과제 후기"
layout: single
author_profile: false
read_time: false
comments: false
share: true
related: true
categories:
  - review
toc: true
toc_sticky: true
toc_labe: 목차
description: freshcode 에서 제시해주신 상품 관리 API 과제의 제작 과정을 정리합니다.
excerpt: freshcode 에서 제시해주신 상품 관리 API 과제의 제작 과정을 정리합니다.
tags:
  - 위코드
  - 원티드
  - freshcode
---

원티드와 위코드에서 지원하는 [원티드 프리온보딩 백엔드 코스](https://www.wanted.co.kr/events/pre_onboarding_course_4)의 두 번째 과제를 수행한 경험을 정리합니다. [GitHub Repository](https://github.com/chinsanchung/preonboarding-freshcode)에서 코드를 확인하실 수 있습니다.

## 개요

이번 프로젝트는 [freshcode](https://www.freshcode.me/)에서 제시한 과제로, **freshcode** 는 "샐러드는 배고픈 다이어트 음식"이라는 편견을 깨고 대한민국 직장인의 건강한 식사 문화를 만드는 것이 목표로, 샐러드 구독·건강간편식 프리미엄 배송 서비스"를 제공하는 기업입니다.

### 과제 안내

1. 로그인 기능

사용자 인증을 통해 상품을 관리할 수 있어야 합니다.

- JWT 인증 방식을 이용합니다.
- 서비스 실행시 데이터베이스 또는 In Memory 상에 유저를 미리 등록해주세요.
- Request시 Header에 Authorization 키를 체크합니다.
- Authorization 키의 값이 없거나 인증 실패시 적절한 Error Handling을 해주세요.
- 상품 추가/수정/삭제는 admin 권한을 가진 사용자만 이용할 수 있습니다.
- 사용자 인증 / 인가

2. 상품 관리 기능

아래 상품 JSON 구조를 이용하여 데이터베이스 및 API를 개발해주세요.

- 서비스 실행시 데이터베이스 또는 In Memory 상에 상품 최소한 5개를 미리 생성해주세요.
- 상품 조회는 하나 또는 전체목록을 조회할 수 있으며, 전체목록은 페이징 기능이 있습니다.
  - 한 페이지 당 아이템 수는 5개 입니다.
- 사용자는 상품 조회만 가능합니다.
- 관리자는 상품 추가/수정/삭제를 할 수 있습니다.
- 상품 관리 API 개발시 적절한 Error Handling을 해주세요.

## 회고

### NestJs 를 이용한 로그인

이번 프로젝트부터 [NestJs](https://nestjs.com/) 프레임워크를 사용하기로 결정했습니다. 그 이유는 Express 는 자유도가 높은 대신 필요한 설정들을 직접 가져와야 하는 점, 그리고 각자 코드를 짜거나 구조를 맞추는 방식이 달라 결과물에서 약간씩 차이점이 있다는 점 떄문입니다. NestJs 는 규칙과 애플리케이션 아키텍쳐로 인해 협업에 용이하며, 타입스크립트를 지원하여 예상치 못한 에러가 발생할 가능성을 줄여줍니다.

우선 로그인을 어떻게 작성했는지를 설명하겠습니다..

#### 1. 초기 설정

bcrypt, passport, passport-local, @nestjs/passport 패키지를 설치합니다.

bcrypt 로 비밀번호 관련 설정을, passport-local 은 로그인 전략을 구현합니다. 인증 미들웨어인 passport 를 사용한 이유는 다양한 전략으로 인증을 구현할 수 있다는 점, 그리고 로그인이 성공하면 Request 객체에 유저 정보를 첨부해 활용하게 해주는 편의성 떄문입니다.

#### 2. 로그인을 다룰 모듈, 컨트롤러와 서비스 만들기

`nest g resource auth` 명령어로 모듈, 컨트롤러, 서비스를 만듭니다. 하위 명령어는 REST API, CRUD 엔트리 포인트는 하지 않도록 입력합니다.

#### 3. 로컬 전략을 작성하기

우선 로컬 전략에 사용하기 위해 authService 에서 유저를 검증하는 메소드 `validateUser`를 작성합니다.

```typescript
export class AuthService {
  async validateUser({ email, password }: LoginUserDto) {
    try {
      const user = await this.usersService.findOne(email);
      if (!user || (user && !(await bcrypt.compare(password, user.password)))) {
        // 유저가 존재하지 않을 때, 또는 유저가 존재하지만 비밀번호가 일치하지 않을 때
        // 에러를 리턴합니다.
        return {
          ok: false,
          htmlStatus: 403,
          error: "올바르지 않은 이메일 또는 비밀번호 입니다.",
        };
      }
      const loginedAt = new Date();

      await this.usersService.updateLoginedAt(email, loginedAt);
      return {
        ok: true,
        data: { email: user.email, role: user.role, loginedAt },
      };
    } catch (error) {
      return {
        ok: false,
        htmlStatus: 500,
        error: "로그인 과정에서 에러가 발생했습니다.",
      };
    }
  }
}
```

`updateLoginedAt` 메소드는 로그인 시 유저의 로그인 시각을 데이터베이스에 저장하여 토큰의 유효성을 검증할 때 사용합니다.

그 다음 auth/strategies/local.strategy.ts 파일을 만들고, passport-local 에 사용할 로컬 전략을 만듭니다. `super` 안의 usernameField, passwordField 속성으로 로그인에 입력할 필드의 이름을 수정했습니다.

```typescript
import { HttpException, Injectable } from "@nestjs/common";
import { PassportStrategy } from "@nestjs/passport";
import { Strategy } from "passport-local";
import { AuthService } from "../auth.service";

@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    super({
      usernameField: "email",
      passwordField: "password",
    });
  }

  async validate(email, password) {
    const loginUserDto = { email, password };
    const result = await this.authService.validateUser(loginUserDto);
    if (result.ok) {
      return {
        email: result.data.email,
        role: result.data.role,
        loginedAt: result.data.loginedAt,
      };
    } else {
      throw new HttpException(result.error, result.htmlStatus);
    }
  }
}
```

마지막으로 작성한 로컬 전략을 auth.module.ts 모듈에 프로바이더로 등록합니다.(프로바이더로 등록하면 컨트롤러에서 생성자를 통해 주입하여 사용할 수 있습니다.) 또한 @nestjs/passport 의 `PassportModule` 을 가져와야 @nestjs/passport 에서 지원하는 기능을 auth.service.ts, auth.controller.ts 에서 사용할 수 있습니다.

```typescript
@Module({
  imports: [
    // ...
    PassportModule,
  ],
  providers: [AuthService, LocalStrategy],
})
export class AuthModule {}
```

#### 4. 로그인을 위한 로컬 가드 만들기

가드는 클라이언트의 요청을 컨트롤러에서 받아들이기 전에 특정한 기능을 수행합니다. 여기서는 @nestjs/passport 의 `AuthGuard` 를 이용한 로컬 전략을 가드에서 실행하도록 작성합니다.

local-auth.guard.ts 에서 `LocalAuthGuard` 클래스를 생성합니다. NestJs 공식 문서에서는 `AuthGuard('local')` 자체를 가드로 사용하는 대신, 자신만의 클래스를 만드는 것을 권장합니다.

```typescript
import { Injectable } from "@nestjs/common";
import { AuthGuard } from "@nestjs/passport";

@Injectable()
export class LocalAuthGuard extends AuthGuard("local") {}
```

#### 5. 토큰을 발급하고 응답으로 보내기

로그인이 성공했을 경우 토큰을 발급합니다.

```typescript
import { JwtService } from "@nestjs/jwt";

export class AuthService {
  constructor(private jwtService: JwtService) {}
  async login(user) {
    const payload = { user };
    return {
      access_token: this.jwtService.sign(payload),
    };
  }
}
```

##### jsonwebtoken

jsonwebtoken 은 JSON 형식의 데이터를 저장합니다. 토큰의 종류와 해시 알고리즘의 정보가 담긴 헤더, 토큰의 내용물을 인코딩한 페이로드, 그리고 토큰의 유효성을 검증하는 시그니처로 구성되어 있습니다.

- 시그니처는 비밀 키로 만드는데, 토큰을 위조할 위험 때문에 비밀 키를 숨겨야 합니다. 이후 @nestjs/config 를 통해 안전하게 비밀 키를 불러오는 방법을 설명하겠습니다.
- 토큰의 내용을 누구든 확인할 수 있기에 노출해도 괜찮은 정보만을 담아야합니다.

jsonwebtoken 의 단점은 내용에 따라 용량이 커질 가능성이 있다는 것입니다. 그러면 요청할 때마다 토큰을 주고받아 서버에 부담이 커질 것입니다. 또한 한 번 토큰을 발급하면 지정한 시간이 되기 전까진 만료시킬 수 없습니다.

#### 6. 발급한 토큰을 응답으로 보내기

마지막으로 컨트롤러입니다. 이전에 만든 로컬 가드를 데코레이터로 붙여 로컬 전략을 수행한 후에 토큰을 발급하도록 합니다.

```typescript
export class AuthController {
  @UseGuards(LocalAuthGuard)
  @Post("login")
  async login() {
    return this.authService.login(req.user);
  }
}
```

### 로그인 인증 구현하기

로그인을 통해 발급한 토큰을 해석해 유효성을 검증하고, Request 객체에 유저 정보를 담아 클라이언트의 요청을 처리하기 위해선 로그인 인증이 필요합니다.

#### 1. 초기 설정

1. 토큰을 발급하는데 필요한 passport-jwt, @nestjs/jwt, @nestjs/config 패키지를 설치합니다. 특히 @nestjs/config 패키지는 env 파일을 auth 모듈 내에서 불러올 때 사용합니다.

2. .env 파일을 만들어 jsonwebtoken 을 위한 비밀 키를 저장합니다.

3. app.module.ts 에서 @nestjs/config 으로 env 관련 설정을 작성합니다.

```typescript
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: ".env",
    }),
    // ...
  ],
  // ...
})
export class AppModule {}
```

전역으로 사용을, 파일의 경로는 `.env`로 설정했습니다. 참고로 process.env.NODE_ENV 가 무엇인지에 따라 다른 env 파일을 경로로 지정할 수도, `validationSchema` 속성으로 파일 내용에 대해 유효성 검사를 시행할 수도 있습니다.

```
envFilePath: process.env.NODE_ENV === "dev" ? ".env.dev" : ".env.test",
ignoreEnvFile: process.env.NODE_ENV === "prod",
validationSchema: Joi.object({
  NODE_ENV: Joi.string().valid("dev", "prod").required(),
  JWT_SECRET: Joi.string().required(),
})
```

#### 2. jwt 모듈을 가져와 토큰에 대한 설정하기

auth.module.ts 에서 @nestjs/jwt 의 JwtModule 을 가져옵니다. 비밀 키를 입력하고, 토큰에 대한 여러 속성을 지정합니다.

```typescript
@Module({
  imports: [
    JwtModule.registerAsync({
      imports: [ConfigModule],
      useFactory: async (configService: ConfigService) => ({
        secretOrPrivateKey: configService.get("JWT_SECRET"),
        signOptions: { expiresIn: "1h" },
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AuthModule {}
```

`registerAsync` 메소드를 쓴 이유는 비동기로 env 의 비밀 키를 가져오기 위해서입니다. (비밀 키는 언제나 안전하게 숨겨야 합니다.) @nestjs/config 에서 `ConfigModule` 을 불러온 후, `configService.get` 으로 비밀 키를 가져옵니다.

#### 3. JWT 전략 작성하기

다음으로 토큰을 검증 및 해석하는 `JwtStrategy`를 작성합니다. 앞서 토큰은 한 번 작성하면 만료시킬 방법이 없다고 했는데, 유저 데이터의 loginedAt 컬럼과 토큰의 loginedAt 을 비교하여 로그아웃 여부를 확인하는 방법을 택했습니다.

```typescript
export class JwtStrategy extends PassportStrategy(Strategy) {
  async validate(payload: any) {
    const { email, role, loginedAt } = payload.user;
    const user = await this.usersService.findOne(email);
    const tokenLoginedAt = new Date(loginedAt).getTime();
    const userLoginedAt = new Date(user.loginedAt).getTime();
    if (tokenLoginedAt !== userLoginedAt) {
      throw new UnauthorizedException("올바르지 않은 토큰입니다");
    }
    return { email, role, loginedAt };
  }
}
```

이 전략을 auth 모듈의 프로바이더로 등록합니다.

```typescript
@Module({
  imports: [
    // ...
    PassportModule,
  ],
  providers: [AuthService, LocalStrategy, JwtStrategy],
})
export class AuthModule {}
```

#### 4. JWT 가드를 만들기

마지막으로 JWT 전략을 수행할 가드를 만듭니다. 로컬 가드와 마찬가지로 특정 요청을 수행하기 전에 토큰을 검증하고, Request 객체에 유저 정보를 담게 됩니다.

```typescript
import { Injectable } from "@nestjs/common";
import { AuthGuard } from "@nestjs/passport";

@Injectable()
export class JwtAuthGuard extends AuthGuard("jwt") {}
```

완성입니다. 사용하려면 `@UseGuards(JwtAuthGuard)` 데코레이터를 컨트롤러 메소드 위에 선언하면 됩니다.

### 로그아웃하기

로그아웃을 하면 유저의 loginedAt 을 null 로 바꿉니다. 비록 토큰 자체는 만료되지 않더라도 토큰의 loginedAt 이 유저와 다르므로 만료된 것으로 처리할 수 있습니다.

## 프로젝트를 마치며

이번 프로젝트에서 얻은 성과는 NestJs 로 API 를 제작하는 과정을 확인한 것, 그리고 데이터베이스를 SQLite3 으로 정하면서 TypeORM 을 처음으로 접했는데 어떻게 데이터베이스에 접속하고 제어하는지 직접 코드로 작성하면서 확인한 것입니다. 특히 엔티티를 선언할 떄 MongoDB 스키마와 달리 1:N 인지, N:N 인지를 명확히 해야 한다는 점을 알았습니다. TypeORM 사용법에 대해 더 알아야겠다고 생각했습니다.
