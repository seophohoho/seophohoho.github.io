---
layout: post
title: '동시에 가입하면 왜 500 에러가 뜰까? Race Condition 해결기'
sub-title: 'Check-then-act 패턴과 비동기 환경에서 발생하는 Race Condition 문제 해결기'
date: 2026-02-01 20:18:00 +0900
tags: [poposafari, Node.js, Race Condition]
---

프로젝트 poposafari의 회원가입 API를 개발하던 중, 문득 한 가지 의문이 생겼습니다.

> **동시에 동일한 `username`으로 회원가입 요청을 받으면, 어떻게 될까?**
{: .callout .question}



## 기존 로직 살펴보기

먼저 기존에 작성했던 회원가입 로직을 보겠습니다.
```typescript
// Local Register service logic
async registerLocal(req: AuthLocalReq): Promise<TokenPair> {
  const { username, password } = req;
  
  // 1. Check: 기존 유저가 있는지 확인
  const existingAuth = await this.authRepository.findByProviderAndProviderId(...);
  if (existingAuth) throw new AppError(..., 409);

  // ... 트랜잭션 및 암호화 로직 ...

  // 2. Act: 없으면 가입 진행 및 저장
  const auth = await this.authRepository.create(..., hashedPassword);
  await queryRunner.commitTransaction();
}
```

전형적인 **check-then-act 패턴**입니다. 먼저 중복을 확인(Check)하고, 없으면 생성(Act)합니다. 순차적인 흐름에서는 아무런 문제가 없어 보입니다.

하지만 정말 그럴까요? 실제로 테스트해보기로 했습니다.

## 동시성 테스트 설계하기

100명의 유저가 동일한 username으로 동시에 가입 버튼을 누르는 상황을 가정했습니다. 이를 테스트하기 위해 어떤 도구를 사용할지 고민했습니다.

**고려했던 옵션들:**
- **Apache JMeter, k6** 같은 부하 테스트 도구 → 간단한 검증에는 과했고, 설정이 복잡했습니다
- **단순 for문 반복** → 실제로는 순차 실행되어 "동시성"을 재현할 수 없었습니다
- **setTimeout으로 타이밍 조절** → 밀리초 단위로 정확한 동시 실행을 보장하기 어려웠습니다
- **Promise.allSettled** → 모든 결과를 받아 분석하기 좋지만, 실패를 빠르게 감지하기는 어려웠습니다

결국 **Promise.all**을 선택했습니다. 진짜 동시 실행을 보장하면서도, 코드가 간결하고 즉각적인 피드백을 받을 수 있었기 때문입니다.
```typescript
const url = 'http://localhost:3000/api/auth/register/local';
const payload = {
    username: 'testuser123',
    password: 'password123'
};

// 100개의 동시 요청 생성
const requests = Array.from({ length: 100 }, () => {
    return axios.post(url, payload).catch((err) => err.response);
});

const responses = await Promise.all(requests);

// 결과 집계
const summary = responses.reduce((acc, res) => {
    const status = res.status || 'Unknown';
    acc[status] = (acc[status] || 0) + 1;
    return acc;
}, {});

console.table(Object.entries(summary).map(([status, count]) => ({
    'HTTP Status': status,
    'Count': count,
    'Result': status === '201' ? 'SUCCESS' : 
              status === '409' ? 'CONFLICT' : 'ERROR'
})));
```

## 예상을 벗어난 결과

테스트를 돌렸습니다. 예상은 이랬습니다.

- ✅ 1개의 요청만 성공 (201)
- ✅ 나머지 99개는 중복 에러 (409)

하지만 실제 결과는 전혀 달랐습니다.
```bash
┌─────────┬─────────────┬───────┬───────────┐
│ (index) │ HTTP Status │ Count │ Result    │
├─────────┼─────────────┼───────┼───────────┤
│ 0       │ '201'       │ 1     │ 'SUCCESS' │
│ 1       │ '500'       │ 99    │ 'ERROR'   │
└─────────┴─────────────┴───────┴───────────┘
```

409가 아닌 **500 에러**가 발생했습니다. 서버 로그를 확인해보니 더 흥미로운 사실을 발견했습니다.
```bash
POST /api/auth/register/local 500 1730.163 ms - 165
{
    severity: 'ERROR',
    code: '23505',
    detail: 'Key (provider, provider_id)=(local, testuser123) already exists.',
    table: 'auth_identities',
    constraint: 'IDX_b4756e087ec67d0abe2f53792a',
}
```

PostgreSQL의 Unique 제약조건 위반 에러였습니다. 다행히 DB 레벨에서 중복 저장은 막았지만, 사용자에게는 "서버가 뭔가 잘못됐다"는 느낌을 주는 500 에러를 반환하고 있었습니다.

## 왜 이런 일이 발생했을까?

문제의 원인은 명확했습니다. 100개의 요청이 "거의 동시에" 중복 체크를 통과했기 때문입니다.
```
시간 →

Request 1:  [중복 체크 ✓] -----> [데이터 저장 ✓]
Request 2:  [중복 체크 ✓] -----> [데이터 저장 ✗] (DB 에러!)
Request 3:  [중복 체크 ✓] -----> [데이터 저장 ✗] (DB 에러!)
...
Request 100: [중복 체크 ✓] -----> [데이터 저장 ✗] (DB 에러!)
```

모든 요청이 중복 체크 시점에는 "아직 데이터가 없다"고 판단했습니다. 그리고 거의 동시에 INSERT를 시도했고, 첫 번째만 성공하고 나머지는 Unique 제약조건에 걸려 실패한 것입니다.

이것이 전형적인 **Race Condition**입니다. Check와 Act 사이의 짧은 시간 간격이 문제를 만들었습니다.

## 해결 방법 검토하기

해결 방법은 크게 두 가지 방향으로 나뉩니다.

1. **예방(Prevention)**: 애초에 Race Condition이 발생하지 않게 만들기
2. **대응(Handling)**: 발생한 에러를 적절히 처리하기

### 예방 방법들

여러 Lock 메커니즘을 검토했습니다.

#### 1) 비관적 락 (Pessimistic Lock)
```typescript
const existingAuth = await this.authRepository.findOne({
  where: { provider, providerId },
  lock: { mode: 'pessimistic_write' } // SELECT ... FOR UPDATE
});
```

**문제점:**
- 읽기만 하는데 쓰기 락을 거는 것은 과도합니다
- 더 큰 문제는, **없는 행에는 락을 걸 수 없다**는 것입니다
- 회원가입은 INSERT 작업인데, 조회 시점에는 행이 존재하지 않습니다

#### 2) 낙관적 락 (Optimistic Lock)
```typescript
@Entity()
class AuthIdentity {
  @VersionColumn()
  version: number;
}
```

**문제점:**
- 버전 컬럼으로 충돌을 감지하지만, 신규 생성(INSERT) 시에는 적합하지 않습니다
- UPDATE 작업에 유용한 방식입니다

#### 3) 분산 락 (Distributed Lock with Redis)
```typescript
const lockKey = `signup:${username}`;
const lock = await redis.set(lockKey, '1', 'NX', 'EX', 5);

if (!lock) {
  throw new AppError('이미 처리 중입니다', 409);
}

try {
  // 회원가입 로직
} finally {
  await redis.del(lockKey);
}
```

**문제점:**
- Redis 의존성이 추가됩니다
- 락 해제 실패 시 복구 로직이 필요합니다
- 회원가입처럼 단순한 기능에는 오버엔지니어링입니다

## 실용적인 선택

결국 **실용성을 선택**했습니다.

몇 가지 사실을 고려했습니다:
- 이미 DB에 Unique 제약조건이 있습니다
- Race Condition은 실제로는 극히 드문 상황입니다
- 복잡한 예방 로직은 유지보수 비용을 높입니다

**"발생한 에러를 잘 처리하는 것"**이 더 합리적이라 판단했습니다.

### 최종 코드
```typescript
async registerLocal(req: AuthLocalReq): Promise<TokenPair> {
  const { username, password } = req;
  
  // 1차 방어: 일반적인 상황에서의 중복 체크
  const existingAuth = await this.authRepository.findByProviderAndProviderId(
    UserAuthProvider.LOCAL,
    username,
  );

  if (existingAuth) {
    throw new AppError(
      AppErrorMessage.USER_ALREADY_EXISTS, 
      409, 
      AppErrorCode.CONFLICT
    );
  }

  const queryRunner = this.dataSource.createQueryRunner();
  await queryRunner.connect();
  await queryRunner.startTransaction();

  try {
    const hashedPassword = await bcrypt.hash(password, this.SALT_ROUNDS);
    const auth = await this.authRepository.create(
      UserAuthProvider.LOCAL,
      username,
      hashedPassword,
    );
    
    const tokenPair = await this.generateAndStoreTokens(auth.id);
    await queryRunner.commitTransaction();

    return {
      authId: auth.id,
      accessToken: tokenPair.accessToken,
      refreshToken: tokenPair.refreshToken,
    };
  } catch (error) {
    await queryRunner.rollbackTransaction();

    // 2차 방어: Race Condition 발생 시 DB 에러를 사용자 친화적인 에러로 변환
    const dbError = error as { code?: string };
    if (dbError.code === '23505') { // PostgreSQL Unique 제약조건 위반
      throw new AppError(
        AppErrorMessage.USER_ALREADY_EXISTS, 
        409, 
        AppErrorCode.CONFLICT
      );
    }
    
    throw error;
  } finally {
    await queryRunner.release();
  }
}
```

**핵심은 2차 방어선입니다.** PostgreSQL의 에러 코드 `23505`(Unique 제약조건 위반)를 catch해서, 사용자에게 적절한 409 Conflict 에러로 변환합니다.

### 개선된 결과

다시 테스트를 돌려봤습니다.
```bash
┌─────────┬─────────────┬───────┬───────────┐
│ (index) │ HTTP Status │ Count │ Result    │
├─────────┼─────────────┼───────┼───────────┤
│ 0       │ '201'       │ 1     │ 'SUCCESS' │
│ 1       │ '409'       │ 99    │ 'CONFLICT'│
└─────────┴─────────────┴───────┴───────────┘
```

완벽합니다! 이제 사용자는 "이미 존재하는 아이디입니다"라는 명확한 메시지를 받을 수 있습니다.

## 마무리

이번 경험을 통해 배운 것들:

### 1. "당연히 잘 되겠지"는 없다
직접 테스트하기 전까지는 확신할 수 없습니다. 특히 동시성 문제는 코드 리뷰만으로는 발견하기 어렵습니다.

### 2. 완벽한 해결책은 없다
모든 해결책에는 Trade-off가 있습니다. 상황에 맞는 선택을 하는 것이 중요합니다. 때로는 "완벽한 예방"보다 "적절한 대응"이 더 나은 선택일 수 있습니다.

### 3. DB 제약조건은 마지막 방어선
애플리케이션 레벨의 검증이 실패하더라도, DB 제약조건은 데이터 무결성을 지켜줍니다. 항상 DB 레벨의 제약조건을 함께 설정하는 것이 좋습니다.

다음 단계로는, 실제 프로덕션 환경에서 Race Condition이 얼마나 발생하는지 모니터링해볼 계획입니다. 만약 빈도가 높다면, 그때 분산 락 도입을 고려해보려 합니다.

## 참고

- [PostgreSQL Error Codes](https://www.postgresql.org/docs/current/errcodes-appendix.html)
- [TypeORM Transaction Documentation](https://typeorm.io/transactions)
- [Node.js Concurrency Model](https://nodejs.org/en/docs/guides/blocking-vs-non-blocking/)