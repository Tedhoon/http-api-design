# HTTP API 디자인 가이드

## 들어가며

이 가이드에선 HTTP+JSON API 디자인 사례를 설명하며,
이는 [히로쿠 플랫폼 API](https://devcenter.heroku.com/articles/platform-api-reference)를 작업하며 뽑은 내용을 기반으로 하고 있다.

이 가이드는 해당 API에 관한 추가적인 내용을 다룰뿐만 아니라, 히로쿠의 새로운 내부 API의 가이드이기도 하다.
우리는 이 글이 히로쿠 밖의 API 설계자에게도 관심을 받을 수 있길 바란다.

여기서 우리의 목적은 디자인 [바이크쉐딩](http://en.wiktionary.org/wiki/bikeshedding)을 피해, 비즈니스 로직에 일관성 있게 집중하는 데 있다.
우리는 _훌륭하고, 일관성 있고, 잘 문서화된 방식_으로 API를 디자인하는 방법을 찾고자 노력하고 있으며, 이 방법이 _유일하고 이상적인 방식_일 필요는 없다고 생각한다.  

우린 여러분이 HTTP+JSON API의 기본적인 요소를 잘 알고 있다고 가정했으며, 따라서 이 가이드에선 기본적인 세부 사항을 모두 다루진 않는다.

이 가이드의 작성에 [참여](../CONTRIBUTING.md)하고 싶다면 언제든 환영이다.

## Contents

*  [올바른 상태 코드를 반환하라](#올바른-상태-코드를-반환하라)
*  [가능하다면 전체 리소스를 모두 제공하라](#가능하다면-전체-리소스를-모두-제공하라)
*  [요청의 본문에 직렬화된 JSON을 허용하라](#요청의-본문에-직렬화된-json을-허용하라)
*  [리소스 (UU)ID를 제공하라](#리소스-uudi를-제공하라)
*  [표준 타임스탬프를 제공하라](#표준-타임스탬프를-제공하라)
*  [ISO8601 포맷에 맞춘 UTC 시간을 사용하라](#iso8601-포맷에-맞춘-utc-시간을-사용하라)
*  [일관성 있는 경로 포맷을 사용하라](#일관성-있는-경로-포맷을-사용하라)
*  [경로와 속성은 소문자로 만들어라](#경로와-속성은-소문자로-만들어라)
*  [외래 키 관계를 중첩시켜라](#외래-키-관계를-중첩시켜라)
*  [편의성을 위해 ID가 아닌 역참조를 지원하라](#편의성을-위해-id가-아닌-역참조를-지원하라)
*  [구조화된 오류를 생성하라](#구조화된-오류를-생성하라)
*  [Etag 캐싱을 지원하라](#etag-캐싱을-지원하라)
*  [요청 ID로 요청을 추적하라](#요청-id로-요청을-추적하라)
*  [범위를 지정해 페이지를 만들라](#범위를-지정해-페이지를-만들라)
*  [빈도 제한 상태를 보여줘라](#빈도-제한-상태를-보여줘라)
*  [승인 헤더로 버전을 매겨라](#승인-헤더로-버전을-매겨라)
*  [경로 중첩을 최소화하라](#경로-중첩을-최소화하라)
*  [기계가 읽을 수 있는 JSON 스키마를 제공하라](#기계가-읽을-수-있는-json-스키마를-제공하라)
*  [사람이 읽을 수 있는 문서를 제공하라](#사람이-읽을-수-있는-문서를-제공하라)
*  [실행할 수 있는 예제를 제공하라](#실행할-수-있는-예제를-제공하라)
*  [안정성의 정도를 나타내라](#안정성의-정도를-나타내라)
*  [TLS을 요구하라](#tls을-요구하라)
*  [보기 좋게 출력되는 JSON을 기본으로 하라](#보기-좋게-출력되는-json을-기본으로-하라)

### 올바른 상태 코드를 반환하라

각 응답으로 적절한 HTTP 상태 코드를 반환하라. 성공적인 응답에는 반드시 이 가이드에 맞춰 코드를 지정해야 한다.

* `200`: `GET` 호출과 `DELETE` 또는 `PATCH` 호출에 따른 요청이 성공했고, 동시에 완료됐다.
* `201`: `POST` 호출에 따른 요청이 성공했고, 동시에 완료됐다.
* `202`: `POST`나 `DELETE`나 `PATCH` 호출을 받았고, 비동기적으로 수행될 예정이다.
* `206`: `GET`이 성공했지만 부분적인 응답만이 반환됐다: [범위에 관한 위의 내용](#페이지를-범위로-지정하라)을 살펴보자

사용자 오류와 서버 오류에 따른 상태 코드에 관한 도움을 위해선 [HTTP 응답 코드 스펙](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)를 참조하자.

### 가능하다면 전체 리소스를 모두 제공하라

기회가 된다면 항상 응답에 전체 리소스 표현(즉, 모든 속성을 갖고 잇는 객체)을 제공하자.
`PUT`/`PATCH`와 `DELETE` 요청을 포함해, 200과 201 응답에는 항상 전체 리소스를 제공하자.
예를 살펴보자.

```
$ curl -X DELETE \  
  https://service.com/apps/1f9b/domains/0fd4

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
...
{
  "created_at": "2012-01-01T12:00:00Z",
  "hostname": "subdomain.example.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

202 응답에는 전체 리소스 표현이 포함되지 않는다.
예를 살펴보자.

```
$ curl -X DELETE \  
  https://service.com/apps/1f9b/dynos/05bd

HTTP/1.1 202 Accepted
Content-Type: application/json;charset=utf-8
...
{}
```

### 요청의 본문에 직렬화된 JSON을 허용하라

양식에 따라 인코딩된 데이터를 대신하거나 그에 덧붙여, `PUT`/`PATCH`/`POST` 요청 본문에 직렬화된 JSON을 허용하자.
이는 JSON으로 직렬화된 응답 본문과 잘 어울리는 대칭성을 부여해준다.
예를 살펴보자.

```
$ curl -X POST https://service.com/apps \
    -H "Content-Type: application/json" \
    -d '{"name": "demoapp"}'

{
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "name": "demoapp",
  "owner": {
    "email": "username@example.com",
    "id": "01234567-89ab-cdef-0123-456789abcdef"
  },
  ...
}
```

### 리소스 (UU)ID를 제공하라

각 리소스에는 기본적으로 `id` 속성을 부여하자. 다른 좋은 이유가 없다면, UUID를 사용하도록 하자.
서비스의 여러 인스턴스나 서비스 내부의 다른 리소스와 전역적으로 유일하게 구분되지 않는 ID는 사용해선 안되며, 특히 자동으로 증가하는 ID를 주의하자.

UUID는 `8-4-4-4-12` 포맷에 맞춰 소문자로 표현하자.
예를 살펴보자.

```
"id": "01234567-89ab-cdef-0123-456789abcdef"
```

### 표준 타임스탬프를 제공하라

리소스에 `created_at`과 `updated_at` 타임스탬프를 기본으로 제공하자.
예를 살펴보자.

```json
{
  ...
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T13:00:00Z",
  ...
}
```

일부 리소스에선 이런 타임스탬프가 맞지 않을 수도 있으며, 그런 경우에는 생략할 수 있다.

### ISO8601 포맷에 맞춘 UTC 시간을 사용하라

시간은 UTC만을 허용하고 UTC만을 반환토록 하자. 시간은 ISO8601 포맷에 맞추자.
예를 살펴보자.

```
"finished_at": "2012-01-01T12:00:00Z"
```

### 일관성 있는 경로 포맷을 사용하라

#### 리소스 이름

해당 리소스가 시스템 내에서 싱글턴으로 존재하지 않는다면(예를 들어, 대부분의 시스템에선 하나의 사용자가 하나의 계정만을 가질 수 있다), 해당 리소스의 복수형 버전을 사용하도록 하자.
이는 특정 리소스를 참조하는 방식을 일관되게 유지할 수 있도록 해준다.

#### 액션

개별 리소스에 대한 특별한 액션이 필요 없는 엔드포인드 레이아웃을 선호하자.
특별한 액션이 필요한 경우엔 이를 표준화된 `action` 접두사 아래에 위치시켜서, 다음과 같이 명확히 구분해 표현토록 하자.

```
/resources/:resource/actions/:action
```

예를 살펴보자.

```
/runs/{run_id}/actions/stop
```

### 경로와 속성은 소문자로 만들어라

소문자와 가운데 줄로 구분된 경로 이름을 사용해 호스트 이름과 그 모습을 맞추도록 하자.
예를 살펴보자.

```
service-api.com/users
service-api.com/app-setups
```

속성도 마찬가지로 소문자를 사용하자. 다만 자바스크립트에서 타입 지정하기 위해 따옴표로 둘러쌀 필요가 없도록 밑줄을 사용해 구분하자.
예를 살펴보자.

```
service_class: "first"
```

### 외래 키 관계를 중첩시켜라

외래 키 참조는 중첩된 객체로 직렬화하자.
예를 살펴보자.

```json
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0..."
  },
  ...
}
```

다음은 안 좋은 예다.

```json
{
  "name": "service-production",
  "owner_id": "5d8201b0...",
  ...
}
```

이 접근법은 응답의 구조를 변경하거나 최상위 수준의 응답 필드를 추가할 필요 없이, 관련 리소스에 관한 추가 정보를 끼워넣을 수 있도록 해준다.
예를 살펴보자.

```json
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0...",
    "name": "Alice",
    "email": "alice@heroku.com"
  },
  ...
}
```

### 편의성을 위해 ID가 아닌 역참조를 지원하라

때론 최종 사용자에게 리소스 식별을 위해 ID를 제공하는 편이 불편할 수도 있다.
예를 들어 사용자는 히로쿠 앱 이름을 기준으로 생각하지만 해당 앱은 UUID를 통해 식별된다.
이런 경우, 여러분은 ID나 이름을 모두 허용하는 방안을 원할 수도 있다.
예를 살펴보자.

```
$ curl https://service.com/apps/{app_id_or_name}
$ curl https://service.com/apps/97addcf0-c182
$ curl https://service.com/apps/www-prod
```

ID를 배제하고 이름만을 허용해선 안된다.

### 구조화된 오류를 생성하라

요류가 발생하면 일관성 있고 구조화된 응답 본문을 생성하자.
기계가 읽을 수 있는 오류 `id` 및 사람이 읽을 수 있는 오류 `message`와 선택사항으로써 클라이언트에게 오류에 관한 정보와 해결 방안을 알려줄 `url`을 포함시키자.
예를 살펴보자.

```
HTTP/1.1 429 Too Many Requests
```

```json
{
  "id":      "rate_limit",
  "message": "Account reached its API rate limit.",
  "url":     "https://docs.service.com/rate-limits"
}
```

오류 포맷과 클라이언트가 접할 수 있는 오류 `id`를 문서화하자.

### Etag 캐싱을 지원하라

모든 응답에 `ETag` 헤더를 포함시켜서 반환된 리소스의 정확한 버전을 알 수 있도록 하자.
사용자는`If-None-Match` 헤더를 사용해 일련의 요청 과정에서 예전 버전임을 반드시 확인할 수 있어야 한다.

### 요청 ID로 요청을 추적하라

각 API 응답에 UUID 값을 담아서 만들어진 `Request-Id` 헤더를 포함시키자.
서버와 클라이언트 모두가 이 값을 로그로 남긴다면 요청을 추적하고 디버깅하는 데 유용하다.

### 범위를 지정해 페이지를 만들라

많은 양의 데이터가 만들어지는 응답은 여러 페이지로 나누도록 하자.
`Content-Range` 헤더를 사용해 페이지를 나누는 요청을 전달하자.
요청과 응답의 헤더, 상태 코드, 제한, 순서, 페이지 이동 등의 세부 사항은 [범위에 관한 히로쿠 플랫폼 API](https://devcenter.heroku.com/articles/platform-api-reference#ranges)의 예제를 살펴보자.

### 빈도 제한 상태를 보여줘라

클라이언트의 빈도 제한 요청은 서비스의 안정성을 보호하고 다른 클라이언트를 위한 높은 서비스 품질을 유지하도록 해준다.
여러분은 [토큰 버켓 알고리즘](http://en.wikipedia.org/wiki/Token_bucket)을 사용해 요청 제한의 크기를 정할 수 있다.

각 요청의 `RateLimit-Remaining` 응답 헤더를 통해 남은 요청 토큰의 수를 반환하자.

### 승인 헤더로 버전을 매겨라

처음부터 API의 버전을 매기자. `Accepts` 헤더를 사용해 버전 정보와 사용자 정의 컨텐츠 타입을 전달하자.
예를 살펴보자.

```
Accept: application/vnd.heroku+json; version=3
```

버전의 기본값을 지정하지 않는 편이 좋으며, 그 대신 클라이언트가 명시적으로 자신을 특정 버전에 맞춰 사용토록 하자.

### 경로 중첩을 최소화하라

중첩된 부모/자식 리소스 관계를 포함한 데이터 모델에선 경로가 깊숙히 중첩될 수 있다.
예를 살펴보자.

```
/orgs/{org_id}/apps/{app_id}/dynos/{dyno_id}
```

루트 경로에 리소스가 위치하는 편을 선택해 중첩되는 깊이를 제한하자.
중첩은 범위 경계 지정 컬렉션(scoped collection)을 나타내기 위해 사용하자.
위와 같이 dyno가 app에 속해있고, 이는 다시 org에 속한 경우엔 다음과 같이 나타낼 수 있다.

```
/orgs/{org_id}
/orgs/{org_id}/apps
/apps/{app_id}
/apps/{app_id}/dynos
/dynos/{dyno_id}
```

### 기계가 읽을 수 있는 JSON 스키마를 제공하라

기계가 읽을 수 있는 스키마를 제공해 여러분의 API를 정확히 지정할 수 있도록 해주자.
스키마를 관리하기 위해 [prmd](https://github.com/interagent/prmd)를 사용하고, `prmd verify`로 유효성을 검사해 확인하자.

### 사람이 읽을 수 있는 문서를 제공하라

클라이언트 개발자가 여러분의 API를 이해하기 위해 사용할 수 있도록 사용자가 읽을 수 있는 문서를 제공하자.

위에서 설명한 바와 같이 prmd를 사용해 스키마를 만들었다면, 여러분은 `prmd doc`로 손쉽게 마크다운 문서를 생성할 수 있다.

엔드포인트 세부 사항과 함께, 다음과 같은 정보를 담고 있는 API 개요를 제공하자.

* 인증 토큰을 취득하고 사용하는 과정을 포함한 인증 항목.
* 원하는 API 버전을 선택하는 방법을 포함한 API 안정성과 버저닝 항목.
* 요청과 응답의 공통 헤더 항목.
* 오류 직렬화 포맷 항목.
* 클라이언트가 API를 사용하는 예제를 여러 언어로 다룬 항목.

### 실행할 수 있는 예제를 제공하라

사용자가 자신의 터미널에 바로 타이핑해서 API 호출의 동작을 확인할 수 있도록 실행할 수 있는 예제를 제공하자.
API를 사용해 보는 데 드는 사용자의 노력을 최소화하기 위해, 이런 예제는 가능한 최대한 그대로 따라서 사용할 수 있도록 제공돼야 한다.
예를 살펴보자.

```
$ export TOKEN=... # acquire from dashboard
$ curl -is https://$TOKEN@service.com/users
```

[prmd](https://github.com/interagent/prmd)를 사용해 마크다운 문서를 생성했다면 각 엔드포인트에 해당하는 예제를 추가적인 노력 없이 구할 수 있다.

### 안정성의 정도를 나타내라

prototype/development/production과 같은 플래그를 통해 API의 안정도나, API의 성숙도와 안정도에 따라 해당되는 엔드포인트 등을 나타내자.

[히로쿠 API 호환성 정책](https://devcenter.heroku.com/articles/api-compatibility-policy)을 살펴보면 안정성과 변경 관리 접근법의 한 예를 알아볼 수 있다.

일단 여러분의 API가 생산 준비와 안정 단계로 선언됐다면, 해당 API 버전으론 하위 호환성에 맞지 않는 변경을 해선 안된다.
하위 호환성이 없는 변경을 해야만 한다면 버전 번호를 높혀 새로운 API를 만들자.

### TLS을 요구하라

예외를 두지 말고 API에 접근하기 위한 TLS를 요구하자.
TLS를 사용해도 좋은 때와 그렇지 않은 때를 식별하거나 설명하려는 일은 전혀 의미가 없다.
단순히 모든 것에 대해 TLS을 요구하자.

### 보기 좋게 출력되는 JSON을 기본으로 하라

사용자가 여러분의 API를 처음 확인하는 순간은 보통 커맨드 라인에서 curl을 사용할 때다.
만약 API 응답이 보기 좋게 출력된다면 이를 이해하기가 훨씬 쉬워질 것이다.
이런 개발자를 위해 보기 좋게 출력되는 JSON 응답의 예를 살펴보자.

```json
{
  "beta": false,
  "email": "alice@heroku.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "last_login": "2012-01-01T12:00:00Z",
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

다음은 안 좋은 예다.

```json
{"beta":false,"email":"alice@heroku.com","id":"01234567-89ab-cdef-0123-456789abcdef","last_login":"2012-01-01T12:00:00Z", "created_at":"2012-01-01T12:00:00Z","updated_at":"2012-01-01T12:00:00Z"}
```

마지막에 따라오는 줄바꿈을 포함시켰는지 확인해서 사용자의 터미널 프롬프트가 가득차지 않도록 하자.

대부분의 API는 보기 좋게 출력되는 응답은 성능 측면에서 전혀 문제가 없다.
성능에 민감한 API에 대해선 특정 엔드포인트(예, 매우 높은 트래픽이 몰리는)나 특정 클라이언트(예, UI 없는 프로그램)에선 보기 좋게 출력되지 않도록 하는 방안을 고려해볼 수 있다.
