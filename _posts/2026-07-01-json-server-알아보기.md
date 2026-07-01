---
layout: post
title: json-server 알아보기
date: 2026-07-01 15:53 +0900
categories: [살펴보기, Backend]
tags: [backend, json-server]
---

[json-server](https://www.npmjs.com/package/json-server)는 JSON 파일만으로 가짜 REST API 서버를 빠르게 만드는 도구이다.  
`db.json` 같은 파일을 데이터 저장소처럼 사용하고, 그 파일을 기반으로 `GET`, `POST`, `PUT`, `PATCH`, `DELETE` 같은 기본 CRUD 요청을 처리한다.  
그래서 백엔드 API가 완성되기 전에도 서버가 있는 것처럼 프론트엔드 개발이나 API 연동 테스트를 진행할 수 있다.

## 주요 기능

### CRUD

`db.json`의 **최상위 key가 URL 경로**가 된다.

| 동작      | 메서드 | URL        |
| --------- | ------ | ---------- |
| 전체 조회 | GET    | `/todos`   |
| 단건 조회 | GET    | `/todos/1` |
| 생성      | POST   | `/todos`   |
| 전체 수정 | PUT    | `/todos/1` |
| 부분 수정 | PATCH  | `/todos/1` |
| 삭제      | DELETE | `/todos/1` |

```js
// POST - 항목 생성
fetch("http://localhost:3001/todos", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ title: "새 항목", done: false })
});

// PATCH - 부분 수정
fetch("http://localhost:3001/todos/1", {
  method: "PATCH",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ done: true })
});

// DELETE - 삭제
fetch("http://localhost:3001/todos/1", { method: "DELETE" });
```

중첩된 객체는 별도 경로로 접근할 수 없고, 상위 리소스 응답에 포함되어 전달된다.  
단, foreign key로 연결된 리소스는 중첩 경로로 접근할 수 있다.  
필드명이 `{리소스명(단수)}Id` 형식이면 json-server가 자동으로 foreign key로 인식한다. (ex. `users` 리소스 → `userId`)

```json
{
  "users": [{ "id": 1, "name": "Alice" }],
  "posts": [{ "id": 1, "title": "글", "userId": 1 }]
}
```

```
// userId가 1인 posts 반환
http://localhost:3001/users/1/posts
```

### 쿼리 파라미터

필터링, 정렬, 검색, 페이지네이션을 쿼리 파라미터로 지원한다.

```
// 필터링
http://localhost:3001/todos?done=false

// 정렬
http://localhost:3001/todos?_sort=id&_order=desc

// 전체 필드 검색
http://localhost:3001/todos?q=키워드

// 특정 필드 검색
http://localhost:3001/todos?title_like=키워드

// 페이지네이션
http://localhost:3001/todos?_page=1&_limit=10
```

## 기본 사용 흐름

1. `json-server` 설치

   전역 설치하거나, 설치 없이 `npx`로 바로 실행할 수 있다.

   ```zsh
   $ npm install -g json-server
   ```

2. `db.json` 파일 준비
3. json-server 실행

   ```zsh
   $ json-server --watch db.json --port 3001
   # npx 사용 시 (설치 없이 바로 실행)
   $ npx json-server --watch db.json --port 3001
   ```

4. API 호출 테스트
   - URL로 호출

     ```
     http://localhost:3001/todos
     ```

   - fetch로 호출

     ```js
     fetch("http://localhost:3001/todos")
       .then((response) => {
         if (!response.ok) throw new Error("네트워크 응답에 문제가 있습니다.");
         return response.json();
       })
       .then((data) => console.log("성공:", data))
       .catch((error) => console.error("오류 발생:", error));
     ```

## 주의할 점

`json-server`는 실서비스용이 아니라 **개발/테스트용**이기 때문에, 인증, 권한, 복잡한 비즈니스 로직, 대규모 데이터 처리 같은 실제 백엔드 기능을 대신할 수는 없다.
