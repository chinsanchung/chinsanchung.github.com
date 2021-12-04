---
title: "로그인 기능 구현하기"
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
description: 현재 사용하고 있는 로그인 방식에 대해 정리했습니다.
excerpt: 현재 사용하고 있는 로그인 방식에 대해 정리했습니다.
tags:
  - express
  - javascript
---

어떤 과정을 거쳐 로그인을 수행하는지, 그리고 새로고침을 해도 로그인 상태를 유지하기 위한 방법을 정리하고자 이 글을 작성합니다.

## 1. 주요 라이브러리

React, Express, passport-local, passport-jwt, bcrypt, jsonwebtoken, MongoDB

## 2. 클라이언트 화면

### 2.1 로그인 화면

계정과 비밀번호를 입력하는 로그인 페이지입니다. 이메일 로그인이라 가정합니다.

1. 이메일과 비밀번호를 입력합니다.
2. 이메일이 맞는지, 그리고 비밀번호를 입력했는지 확인합니다.
3. 로그인을 수행합니다. 성공할 경우 context 에 사용자 정보를 저장하고, 메인 페이지로 이동합니다.

```javascript
function Login() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [store, dispatch] = authContext();

  // 이메일, 비밀번호 입력
  const onChangeEmail = useCallback((e) => {
    setEmail(e.target.value);
  }, []);
  const onChangePassword = useCallback((e) => {
    setPassword(e.target.value);
  }, []);

  // 양식 체크 및 제출
  const checkEmailValidation = useCallback(() => {
    const regex =
      /^[0-9a-zA-Z]([-_.]?[0-9a-zA-Z])*@[0-9a-zA-Z]([-_.]?[0-9a-zA-Z])*.[a-zA-Z]{2,3}$/i;

    return regex.test(email);
  }, [email]);

  const onStartLogin = useCallback(async () => {
    try {
      if (checkEmailValidation && password !== "") {
        const response = await axios.post("/login", { email, password });
        const { _id, name } = response.data;
        dispatch({ type: "LOGIN", payload: { _id, name } });
      } else alert("양식에 맞춰 작성해주세요.");
    } catch (error) {
      console.log("로그인 에러:", error?.response?.data);
      alert(error?.response?.data);
    }
  }, [email, password, checkEmailValidation]);

  // 로그인 성공 시 메인 페이지로 이동
  useEffect(() => {
    if (store.isLogin) history.push("/main-page");
  }, [auth]);

  return (
    <form onSubmit={onStartLogin}>
      <input
        type="text"
        name="email"
        id="email"
        placeholder="이메일을 입력하세요"
        value={email}
        onChange={onChangeEmail}
      />
      <input
        type="password"
        name="passowrd"
        id="password"
        placeholder="비밀번호를 입력하세요"
        value={password}
        onChange={onChangePassword}
      />
    </form>
  );
}
```

### 2.2 로그인 유지하기

context 로 저장한 유저 정보는 새로고침을 할 경우 초기화되기에, 다른 방법이 필요합니다. 새로고침 시 refreshToken 쿠키가 있으면 그것을 서버에 요청해 검증한 후 일치하면 다시 로그인 처리를 합니다.

```javascript
// 첫 화면인 App.js

function App() {
  const [store, dispatch] = authContext();

  const startRefreshLogin = useCallback(async () => {
    try {
      const response = await axios.post("/refresh-login");
      const { accessToken, _id, name } = response.data;

      axios.defaults.headers.common["Authorization"] = accessToken;
      dispatch({ type: "LOGIN", payload: { _id, name } });
    } catch (error) {
      console.log("새로고침 에러", error?.response?.data);
      alert(error?.response?.data);
    }
  }, []);

  useEffect(() => {
    if (!store.isLogin) {
      startRefreshLogin();
    }
  }, []);

  // ... 그 외 내용은 생략합니다.
}
```

## 3. 백엔드

### 3.1 bcrypt 를 이용한 암호화

bcrypt 를 사용해서 암호화 및 복호화를 하고 있습니다. `createHash`로 비밀번호를 해시로 암호화하고, `comparePassword`로 클라이언트에서 입력한 비밀번호와 데이터베이스에 저장한 비밀번호(해시)를 비교합니다.

```javascript
import bcrypt from "bcrypt";

const saltRounds = 10;

export const createHash = (password) => {
  return bcrypt.hashSync(password, saltRounds);
};
export const comparePassword = (data, encrypted) => {
  return bcrypt.compareSync(data, encrypted);
};
```

### 3.2 passport 설정

Passport 는 사용자 인증을 구현해주는 라이브러리로,

[Passport로 회원가입 및 로그인하기](https://www.zerocho.com/category/NodeJS/post/57b7101ecfbef617003bf457), [passport-local](http://www.passportjs.org/docs/downloads/html/), [bcrypt - npm](https://www.npmjs.com/package/bcrypt)을 참고했습니다.

```javascript
import passport from "passport";
import { Strategy as LocalStrategy } from "passport-local";
import { Strategy as JWTStrategy, ExtractJwt } from "passport-jwt";
import User from "../model/User";
import { createHash, comparePassword } from "../util/passwordFunctions";

const refreshPrivateKey = "refresh token 의 비밀 키";

const configure = () => {
  // 로그인 시 실행합니다.
  passport.use(
    "login",
    new LocalStrategy(
      {
        usernameField: "email",
        passwordField: "password",
        session: false,
        passReqToCallback: false,
      },
      (username, password, done) => {
        User.findOne({ email: username })
          .then((user) => {
            if (!user) {
              return done(null, false, {
                message: "해당 이메일로 등록한 계정이 없습니다.",
              });
            }
            if (!comparePassword(password, user.password)) {
              return done(null, false, {
                message: "해당 계정의 비밀번호와 일치하지 않습니다.",
              });
            }
            return done(null, user);
          })
          .catch((error) => done(error));
      }
    )
  );

  // 새로고침 시 실행합니다.
  passport.use(
    "refresh-login",
    new JWTStrategy(
      {
        secretOrKey: "refreshPrivateKey",
        jwtFromRequest: (req) => {
          if (req && req.cookies) return req.cookies["refreshToken"];
          else return null;
        },
      },
      (payload, done) => {
        User.findOne({ _id: payload.subject })
          .then((user) => {
            if (!user)
              return done(null, false, {
                message: "해당하는 계정이 없습니다.",
              });
            if (payload.refreshToken !== user.refresh_token) {
              // 데이터베이스에 저장한 토큰과 비교합니다.
              return done(null, false, {
                message: "갱신 토큰이 불일치합니다.",
              });
            }
            return done(null, user);
          })
          .catch((error) => done(error));
      }
    )
  );
};
```

이후 express app 을 실행할 때 해당 파일을 불러와서 실행합니다.

```javascript
// app.js
import express from "express";
import config from "./passport";

const app = express();

// passport 를 실행합니다.
config();

app.listen(4000, () => console.log("start server"));
```

### 3.3 로그인 서버 함수

로그인 함수입니다. Passport 에서 로그인 처리를 전부 완료했기에, 새로운 토큰을 만들어 사용자 데이터베이스에 refreshToken 값을 갱신하고, 그것을 쿠키로 만들어줍니다.

토큰에는 사용자의 ObjectId 가 담긴 subject, 그리고 refreshToken 문자열을 활용합니다. 이 문자열은 데이터베이스에 저장해서, 향후 복호화한 토큰과 비교하는 용도로 사용하게 됩니다.

쿠키를 만들 때 `httpOnly`를 넣은 이유는 Document.cookie API에서의 접근을 막고, 서버에게 전송만을 허용하도록 만들기 위해서입니다.

- res.cookie 에 대한 자세한 설명은 [express - res.cookie](http://expressjs.com/en/api.html#res.cookie)에서 확인하실 수 있습니다.

```javascript
const refreshPrivateKey = "refresh token 의 비밀 키";
const accessPrivateKey = "access token 의 비밀 키";

const postLogin = async (req, res) => {
  try {
    const refreshToken = `${req.user._id},${new Date.getTime()}`;
    // 암호화한 refresh token 을 발급합니다.
    const encryptedToken = jsonwebtoken.sign(
      { subject: req.user._id, refreshToken },
      refreshPrivateKey,
      { expiresIn: "14d" }
    );
    // 사용자의 데이터베이스 갱신
    await User.updateOne(
      { _id: req.user._id },
      { $set: { refresh_token: refreshToken } }
    );
    // access token 을 발급합니다.
    const accessToken = jsonwebtoken.sign(
      { subject: req.user._id, refreshToken },
      accessPrivateKey,
      { expiresIn: "1h" }
    );

    res.cookie("accessToken", accessToken, { httpOnly: true });
    // 만료일을 14일로 잡았습니다.
    res.cookie("refreshToken", encryptedToken, {
      httpOnly: true,
      maxAge: 1209600000,
    });
    res.json({ _id: req.user._id, name: req.user.name });
  } catch (error) {
    console.log("로그인 에러", error);
    res.status(500).send("로그인 과정에서 에러가 발생했습니다.");
  }
};
```

다음으로 새로고침으로 사용자를 검증하는 함수입니다. Passport 에서 쿠키와 데이터베이스의 refresh token 을 비교를 완료한 상황입니다. access token 을 새로 갱신해서 저장합니다.

```javascript
const postRefreshLogin = async (req, res) => {
  try {
    const accessToken = jsonwebtoken.sign(
      { subject: req.user._id, refreshToken: req.user.refresh_token },
      accessPrivateKey,
      { expiresIn: "1h" }
    );
    // 새로운 access token 을 발급합니다.
    res.cookie("accessToken", accessToken, { httpOnly: true });

    res.json({ _id: req.user._id, name: req.user.name });
  } catch (error) {
    console.log("새로고침 검증 에러", error);
    res.status(500).send("새로고침 로그인 과정에서 에러가 발생했습니다.");
  }
};
```

### 3.4 Route 설정

URL 에 작성한 서버 함수를 연결하는데, 그 전에 미들웨어로 Passport 를 실행합니다.

```javascript
import passport from "passport";
import { postLogin, postRefreshLogin } from "../controller/Auth";

router.post(
  "/login",
  passport.authenticate("login", { session: false }, (error, user, info) => {
    if (error) res.status(500).json(error);
    if (!user) {
      res.status(401).send(info.message);
    } else {
      req.user = user;
      next();
    }
  }),
  controller.postLogin
);

router.post(
  "/refresh-login",
  passport.authenticate(
    "refresh-login",
    { session: false },
    (error, user, info) => {
      if (error) res.status(500).json(error);
      if (!user) {
        res.status(401).send(info.message);
      } else {
        req.user = user;
        next();
      }
    }
  ),
  controller.postRefreshLogin
);
```

## 4. 번외 - access token 검증하기

access token 을 검증하는 미들웨어를 작성합니다. 로그인이 필요한 기능을 수행하기 전에 미들웨어로 넣어 검증합니다.

```javascript
const accessPrivateKey = "access token 의 비밀 키";

const getCheckAccessToken = async (req, res) => {
  try {
    const {
      headers: { authorization },
    } = req;
    if (!authorization) {
      res.status(401).send("토큰이 존재하지 않습니다.");
    } else {
      const verifiedData = jsonwebtoken.verify(authorization, accessPrivateKey);
      const { subject, refreshToken } = verifiedData;
      const user = await User.findById(subject).select("refresh_token").lean();
      if (refreshToken !== user?.refresh_token) {
        res.status(401).send("refresh token 이 만료되었습니다.");
      } else {
        req.user = user;
        next();
      }
    }
  } catch (error) {
    console.log("access token 이 만료되었습니다.");
  }
};
```
