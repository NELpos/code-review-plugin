# Express 코드 리뷰 가이드

## 목차
- 개요
- 1. 미들웨어 패턴 (Middleware Patterns)
- 2. 라우트 구조화 (Route Organization)
- 3. 에러 처리 (Error Handling)
- 4. 보안 미들웨어 (Security Middleware)
- 5. 요청/응답 패턴 (Request/Response Patterns)
- 6. 인증/인가 (Authentication & Authorization)
- 7. 성능 최적화 (Performance Optimization)
- 8. 통합 체크리스트


## 개요

**Express 코드 리뷰 가이드**는 Express 4.x 프로젝트에서 Tidy First 원칙과 Modern Software Engineering 원칙을 적용한 체계적인 코드 리뷰를 위한 문서입니다. 미들웨어 설계, 라우트 구조, 에러 처리, 보안, 요청/응답 패턴, 인증/인가, 성능 최적화까지 7가지 핵심 영역을 다룹니다.

**7가지 핵심 검토 영역**:
1. **Middleware Patterns** - 미들웨어 순서, async 래핑, next() 흐름 제어
2. **Route Organization** - 라우터 분리, 레이어 구조, 요청 검증
3. **Error Handling** - 중앙 에러 핸들러, 커스텀 에러 클래스, 비동기 에러 전파
4. **Security Middleware** - helmet, CORS, rate limiting, 인젝션 방지
5. **Request/Response Patterns** - 상태 코드, DTO, 일관된 응답 형식
6. **Authentication & Authorization** - JWT 검증, RBAC, 정보 유출 방지
7. **Performance Optimization** - 커넥션 풀, N+1 쿼리, 동기 블로킹 제거

> Express 5.x에서는 async 핸들러의 에러가 자동으로 next()에 전파됩니다. 본 가이드는 Express 4.x 기준이며, 5.x 관련 차이점은 해당 섹션에서 별도로 안내합니다.

---

## 1. 미들웨어 패턴 (Middleware Patterns)

### 검토 항목

1. 미들웨어 등록 순서가 올바른가? (보안 → 파싱 → 로깅 → 인증 → 라우트 → 에러)
2. async 핸들러에 에러 래핑이 되어 있는가?
3. `next()` 호출이 누락되어 요청이 중단되는가?
4. 미들웨어가 응답 후에도 `next()`를 호출하는가? (이중 응답 위험)
5. 서드파티 미들웨어가 최신 버전인가?

### 개선 패턴

#### 예시 1: async 에러 래핑

**Before** (나쁜 예):
```js
// routes/users.js
// async 핸들러에서 예외 발생 시 unhandled rejection으로 서버 크래시 위험
const express = require('express');
const router = express.Router();
const User = require('../models/User');

router.get('/users', async (req, res) => {
  // try/catch 없이 async 함수 사용
  // DB 연결 실패, 타임아웃 등 예외 발생 시 Express가 잡지 못함
  const users = await User.find({ active: true });
  const enriched = await Promise.all(
    users.map(async (user) => {
      const profile = await fetchProfile(user.id);
      return { ...user.toObject(), profile };
    })
  );
  res.json(enriched);
});

router.post('/users', async (req, res) => {
  // 여기도 마찬가지로 래핑 없음
  const user = new User(req.body);
  await user.save();
  res.status(201).json(user);
});

router.delete('/users/:id', async (req, res) => {
  // 존재하지 않는 사용자 삭제 시 에러가 전파되지 않음
  await User.findByIdAndDelete(req.params.id);
  res.status(204).end();
});

module.exports = router;
```

**문제점**:
- Express 4.x는 async 함수에서 throw된 에러를 자동으로 잡지 못함
- 에러 발생 시 `unhandledRejection`으로 프로세스가 종료될 수 있음
- 매 핸들러마다 try/catch를 추가하면 코드 중복이 심해짐
- 클라이언트는 응답을 영원히 기다리게 됨 (타임아웃까지)

**After** (좋은 예):
```js
// utils/asyncHandler.js
// async 핸들러를 감싸서 에러를 자동으로 next()에 전달
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

module.exports = asyncHandler;
```

```js
// routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const asyncHandler = require('../utils/asyncHandler');

// asyncHandler로 감싸면 에러가 자동으로 중앙 에러 핸들러로 전파됨
router.get('/users', asyncHandler(async (req, res) => {
  const users = await User.find({ active: true });
  const enriched = await Promise.all(
    users.map(async (user) => {
      const profile = await fetchProfile(user.id);
      return { ...user.toObject(), profile };
    })
  );
  res.json(enriched);
}));

router.post('/users', asyncHandler(async (req, res) => {
  const user = new User(req.body);
  await user.save();
  res.status(201).json(user);
}));

router.delete('/users/:id', asyncHandler(async (req, res) => {
  await User.findByIdAndDelete(req.params.id);
  res.status(204).end();
}));

module.exports = router;
```

**개선 효과**:
- 모든 async 에러가 중앙 에러 핸들러로 안전하게 전파됨
- try/catch 중복 제거로 핸들러 코드가 깔끔해짐
- Express 5.x에서는 이 래핑이 불필요 (내장 async 지원) — 마이그레이션 시 제거 가능
- `express-async-errors` 패키지를 사용하면 기존 코드 수정 없이 동일 효과 달성 가능

#### 예시 2: 미들웨어 순서 정리

**Before** (나쁜 예):
```js
// app.js
// 미들웨어 순서가 뒤죽박죽 — 보안 헤더 적용 전에 라우트가 처리됨
const express = require('express');
const morgan = require('morgan');
const helmet = require('helmet');
const cors = require('cors');
const bodyParser = require('body-parser');

const app = express();

// 라우트가 보안 미들웨어보다 먼저 등록됨!
const userRouter = require('./routes/users');
const orderRouter = require('./routes/orders');
app.use('/api/users', userRouter);
app.use('/api/orders', orderRouter);

// 보안 헤더가 라우트 뒤에 적용 — 이미 응답이 나간 뒤라 무의미
app.use(helmet());
app.use(cors());

// 바디 파싱도 라우트 뒤에 — req.body가 undefined
app.use(bodyParser.json());

// 로깅도 라우트 뒤에 — 요청 로그가 남지 않음
app.use(morgan('combined'));

// 에러 핸들러 없음 — Express 기본 HTML 에러 페이지가 노출됨
```

**문제점**:
- 보안 헤더(helmet)가 라우트 뒤에 등록되어 응답에 적용되지 않음
- `bodyParser`가 라우트 뒤에 있어 `req.body`가 항상 `undefined`
- 로깅 미들웨어가 라우트 뒤에 있어 요청이 기록되지 않음
- 에러 핸들러가 없어 스택 트레이스가 클라이언트에 노출될 수 있음

**After** (좋은 예):
```js
// app.js
// 올바른 미들웨어 순서: 보안 → 파싱 → 로깅 → 인증 → 라우트 → 에러
const express = require('express');
const morgan = require('morgan');
const helmet = require('helmet');
const cors = require('cors');

const app = express();

// 1단계: 보안 미들웨어 (가장 먼저)
app.use(helmet());
app.use(cors({
  origin: ['https://example.com', 'https://admin.example.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  credentials: true,
}));

// 2단계: 요청 파싱
app.use(express.json({ limit: '10kb' }));
app.use(express.urlencoded({ extended: true, limit: '10kb' }));

// 3단계: 로깅
app.use(morgan('combined'));

// 4단계: 인증 미들웨어 (필요한 라우트에 적용)
const { authenticate } = require('./middleware/auth');

// 5단계: 라우트
const userRouter = require('./routes/users');
const orderRouter = require('./routes/orders');
const publicRouter = require('./routes/public');

app.use('/api/public', publicRouter);
app.use('/api/users', authenticate, userRouter);
app.use('/api/orders', authenticate, orderRouter);

// 6단계: 404 처리
app.use((req, res, next) => {
  res.status(404).json({ success: false, error: { message: '요청한 리소스를 찾을 수 없습니다' } });
});

// 7단계: 중앙 에러 핸들러 (가장 마지막)
const { errorHandler } = require('./middleware/errorHandler');
app.use(errorHandler);

module.exports = app;
```

**개선 효과**:
- 보안 헤더가 모든 응답에 적용되어 XSS, 클릭재킹 등 방어
- 요청 바디가 라우트 도달 전에 파싱되어 정상 처리
- 모든 요청이 로깅되어 디버깅 및 모니터링 가능
- 에러 핸들러가 일관된 JSON 형식으로 에러를 반환

#### 예시 3: next() 누락/이중 호출 방지

**Before** (나쁜 예):
```js
// middleware/auth.js
// next() 누락 및 이중 호출 문제가 있는 인증 미들웨어
const jwt = require('jsonwebtoken');

function authenticate(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];

  if (!token) {
    // 에러 응답을 보냈지만 return이 없어 아래 코드가 계속 실행됨
    res.status(401).json({ error: '토큰이 필요합니다' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    // 응답을 보낸 뒤에도 next()가 호출됨 — 이중 응답 위험
    next();
  } catch (err) {
    // 에러 시 next()를 호출하지 않아 요청이 중단됨 (타임아웃)
    console.log('토큰 검증 실패:', err.message);
  }
}

function requireAdmin(req, res, next) {
  if (req.user.role === 'admin') {
    next();
  }
  // admin이 아닌 경우 next()도 응답도 없음 — 요청 중단
}

module.exports = { authenticate, requireAdmin };
```

**문제점**:
- `res.json()` 후 `return`이 없어 함수가 계속 실행되어 이중 응답 발생
- `catch` 블록에서 `next()`를 호출하지 않아 요청이 무한 대기
- `requireAdmin`에서 admin이 아닌 경우 요청이 영원히 대기
- "Cannot set headers after they are sent" 에러 발생 위험

**After** (좋은 예):
```js
// middleware/auth.js
// 명확한 분기와 early return으로 흐름 제어
const jwt = require('jsonwebtoken');

function authenticate(req, res, next) {
  const authHeader = req.headers.authorization;
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    // return으로 함수 즉시 종료 — 아래 코드 실행 방지
    return res.status(401).json({
      success: false,
      error: { message: '인증 토큰이 필요합니다' },
    });
  }

  const token = authHeader.split(' ')[1];

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET, {
      algorithms: ['HS256'],
    });
    req.user = decoded;
    // 성공 시에만 next() 호출
    return next();
  } catch (err) {
    // 에러를 중앙 핸들러로 전파하거나 적절한 응답 반환
    return res.status(401).json({
      success: false,
      error: { message: '유효하지 않은 토큰입니다' },
    });
  }
}

function requireAdmin(req, res, next) {
  if (req.user && req.user.role === 'admin') {
    return next();
  }
  // 명시적으로 403 응답 — 요청이 중단되지 않음
  return res.status(403).json({
    success: false,
    error: { message: '관리자 권한이 필요합니다' },
  });
}

module.exports = { authenticate, requireAdmin };
```

**개선 효과**:
- 모든 분기에서 `return`을 사용하여 이중 응답 위험 제거
- 에러 시에도 클라이언트에 적절한 응답이 반환됨
- `next()` 호출이 명확한 조건에서만 실행됨
- 요청이 무한 대기하는 상황이 발생하지 않음

---

## 2. 라우트 구조화 (Route Organization)

### 검토 항목

1. 모든 라우트가 app.js/index.js 하나에 정의되어 있는가?
2. 라우트 핸들러에 비즈니스 로직이 직접 포함되어 있는가?
3. 요청 파라미터/바디 검증이 누락되어 있는가?
4. RESTful 규약을 따르는가? (명사형 URL, 적절한 HTTP 메서드)
5. 라우트 그룹별 공통 미들웨어를 적용하고 있는가?

### 개선 패턴

#### 예시 1: 라우터 모듈 분리

**Before** (나쁜 예):
```js
// app.js
// 모든 CRUD 라우트가 한 파일에 나열 — 수백 줄, 유지보수 불가
const express = require('express');
const app = express();
const db = require('./db');

app.use(express.json());

// 사용자 라우트
app.get('/api/users', async (req, res) => {
  const users = await db.query('SELECT * FROM users');
  res.json(users);
});
app.get('/api/users/:id', async (req, res) => {
  const user = await db.query('SELECT * FROM users WHERE id = ?', [req.params.id]);
  res.json(user);
});
app.post('/api/users', async (req, res) => {
  await db.query('INSERT INTO users SET ?', req.body);
  res.status(201).json({ message: '생성됨' });
});
app.put('/api/users/:id', async (req, res) => {
  await db.query('UPDATE users SET ? WHERE id = ?', [req.body, req.params.id]);
  res.json({ message: '수정됨' });
});
app.delete('/api/users/:id', async (req, res) => {
  await db.query('DELETE FROM users WHERE id = ?', [req.params.id]);
  res.status(204).end();
});

// 주문 라우트 (동일한 패턴이 반복됨)
app.get('/api/orders', async (req, res) => { /* ... */ });
app.get('/api/orders/:id', async (req, res) => { /* ... */ });
app.post('/api/orders', async (req, res) => { /* ... */ });
app.put('/api/orders/:id', async (req, res) => { /* ... */ });
app.delete('/api/orders/:id', async (req, res) => { /* ... */ });

// 상품 라우트, 카테고리 라우트, 리뷰 라우트...
// 이 파일이 500줄 이상으로 계속 증가
app.listen(3000);
```

**문제점**:
- 단일 파일에 모든 라우트가 모여 가독성 저하
- 리소스별 공통 미들웨어 적용이 어려움
- 여러 개발자가 동시 작업 시 충돌 빈번
- 라우트 그룹별 테스트가 불가능

**After** (좋은 예):
```js
// routes/users.js
const express = require('express');
const router = express.Router();
const userController = require('../controllers/userController');
const { authenticate } = require('../middleware/auth');
const { validate } = require('../middleware/validate');
const { createUserSchema, updateUserSchema } = require('../validators/userSchema');

// 라우트 정의만 담당 — 비즈니스 로직은 컨트롤러에 위임
router.get('/', userController.getAll);
router.get('/:id', userController.getById);
router.post('/', validate(createUserSchema), userController.create);
router.put('/:id', validate(updateUserSchema), userController.update);
router.delete('/:id', userController.remove);

module.exports = router;
```

```js
// routes/index.js
const express = require('express');
const router = express.Router();
const { authenticate } = require('../middleware/auth');

const userRouter = require('./users');
const orderRouter = require('./orders');
const publicRouter = require('./public');

// 리소스별 라우터를 그룹화하고 공통 미들웨어 적용
router.use('/public', publicRouter);
router.use('/users', authenticate, userRouter);
router.use('/orders', authenticate, orderRouter);

module.exports = router;
```

```js
// app.js
const express = require('express');
const apiRouter = require('./routes');

const app = express();
app.use(express.json());
app.use('/api', apiRouter);

module.exports = app;
```

**개선 효과**:
- 리소스별 파일 분리로 가독성 및 유지보수성 향상
- 라우트 그룹에 공통 미들웨어(인증 등)를 한 번에 적용 가능
- 병렬 개발 시 파일 충돌 최소화
- 라우터 단위 테스트가 용이

#### 예시 2: 핸들러에서 비즈니스 로직 분리

**Before** (나쁜 예):
```js
// routes/orders.js
// 라우트 핸들러 안에 DB 쿼리, 검증, 변환, 외부 API 호출이 모두 섞여 있음
router.post('/orders', async (req, res) => {
  // 1. 수동 검증
  if (!req.body.items || req.body.items.length === 0) {
    return res.status(400).json({ error: '상품을 선택해주세요' });
  }
  if (!req.body.shippingAddress) {
    return res.status(400).json({ error: '배송지를 입력해주세요' });
  }

  // 2. DB에서 상품 조회 및 가격 계산
  let totalAmount = 0;
  for (const item of req.body.items) {
    const product = await db.query(
      'SELECT * FROM products WHERE id = ?', [item.productId]
    );
    if (!product) {
      return res.status(404).json({ error: `상품 ${item.productId}를 찾을 수 없습니다` });
    }
    if (product.stock < item.quantity) {
      return res.status(400).json({ error: `${product.name} 재고 부족` });
    }
    totalAmount += product.price * item.quantity;
  }

  // 3. 할인 계산
  const user = await db.query('SELECT * FROM users WHERE id = ?', [req.user.id]);
  if (user.membershipLevel === 'gold') {
    totalAmount *= 0.9;
  } else if (user.membershipLevel === 'platinum') {
    totalAmount *= 0.85;
  }

  // 4. 주문 생성
  const order = await db.query('INSERT INTO orders SET ?', {
    userId: req.user.id,
    totalAmount,
    status: 'pending',
    shippingAddress: req.body.shippingAddress,
  });

  // 5. 재고 차감
  for (const item of req.body.items) {
    await db.query(
      'UPDATE products SET stock = stock - ? WHERE id = ?',
      [item.quantity, item.productId]
    );
  }

  // 6. 외부 알림 서비스 호출
  await fetch('https://notification.example.com/send', {
    method: 'POST',
    body: JSON.stringify({ userId: req.user.id, orderId: order.insertId }),
  });

  res.status(201).json({ orderId: order.insertId, totalAmount });
});
```

**문제점**:
- 핸들러가 100줄 이상으로 비대 — 검증, DB, 비즈니스 규칙, 외부 호출이 혼재
- 단위 테스트 불가능 (HTTP 요청 없이 비즈니스 로직만 테스트할 수 없음)
- 할인 정책 변경 시 라우트 파일을 수정해야 함
- 동일한 로직을 다른 라우트에서 재사용할 수 없음

**After** (좋은 예):
```js
// controllers/orderController.js
// 컨트롤러는 요청/응답 변환만 담당
const orderService = require('../services/orderService');
const asyncHandler = require('../utils/asyncHandler');

exports.create = asyncHandler(async (req, res) => {
  const order = await orderService.createOrder({
    userId: req.user.id,
    items: req.body.items,
    shippingAddress: req.body.shippingAddress,
  });

  res.status(201).json({
    success: true,
    data: { orderId: order.id, totalAmount: order.totalAmount },
  });
});
```

```js
// services/orderService.js
// 비즈니스 로직만 담당 — HTTP 컨텍스트와 무관
const orderRepository = require('../repositories/orderRepository');
const productRepository = require('../repositories/productRepository');
const userRepository = require('../repositories/userRepository');
const notificationService = require('./notificationService');
const { AppError } = require('../utils/AppError');

async function createOrder({ userId, items, shippingAddress }) {
  // 상품 조회 및 재고 확인
  const products = await productRepository.findByIds(
    items.map((i) => i.productId)
  );

  for (const item of items) {
    const product = products.find((p) => p.id === item.productId);
    if (!product) {
      throw new AppError(`상품 ${item.productId}를 찾을 수 없습니다`, 404);
    }
    if (product.stock < item.quantity) {
      throw new AppError(`${product.name} 재고가 부족합니다`, 400);
    }
  }

  // 금액 계산 (할인 정책 분리)
  const subtotal = calculateSubtotal(items, products);
  const user = await userRepository.findById(userId);
  const totalAmount = applyDiscount(subtotal, user.membershipLevel);

  // 주문 생성 + 재고 차감 (트랜잭션)
  const order = await orderRepository.createWithStockUpdate({
    userId,
    totalAmount,
    shippingAddress,
    items,
  });

  // 비동기 알림 (실패해도 주문에 영향 없음)
  notificationService.sendOrderConfirmation(userId, order.id).catch(
    (err) => console.error('알림 전송 실패:', err)
  );

  return order;
}

function calculateSubtotal(items, products) {
  return items.reduce((sum, item) => {
    const product = products.find((p) => p.id === item.productId);
    return sum + product.price * item.quantity;
  }, 0);
}

function applyDiscount(amount, membershipLevel) {
  const discountRates = { gold: 0.9, platinum: 0.85 };
  return amount * (discountRates[membershipLevel] || 1);
}

module.exports = { createOrder };
```

**개선 효과**:
- 컨트롤러는 요청/응답 변환만, 서비스는 비즈니스 로직만 담당 (단일 책임)
- 서비스 계층은 HTTP 컨텍스트 없이 단위 테스트 가능
- 할인 정책 변경 시 `applyDiscount` 함수만 수정
- 알림 실패가 주문 프로세스에 영향을 주지 않음

#### 예시 3: 요청 검증 미들웨어

**Before** (나쁜 예):
```js
// routes/users.js
// 핸들러 내부에서 수동 if/else 검증 — 중복, 누락 위험
router.post('/users', async (req, res) => {
  // 이름 검증
  if (!req.body.name) {
    return res.status(400).json({ error: '이름은 필수입니다' });
  }
  if (req.body.name.length < 2) {
    return res.status(400).json({ error: '이름은 2자 이상이어야 합니다' });
  }
  if (req.body.name.length > 50) {
    return res.status(400).json({ error: '이름은 50자 이하여야 합니다' });
  }

  // 이메일 검증
  if (!req.body.email) {
    return res.status(400).json({ error: '이메일은 필수입니다' });
  }
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!emailRegex.test(req.body.email)) {
    return res.status(400).json({ error: '유효한 이메일 형식이 아닙니다' });
  }

  // 비밀번호 검증
  if (!req.body.password) {
    return res.status(400).json({ error: '비밀번호는 필수입니다' });
  }
  if (req.body.password.length < 8) {
    return res.status(400).json({ error: '비밀번호는 8자 이상이어야 합니다' });
  }

  // 나이 검증
  if (req.body.age && (req.body.age < 0 || req.body.age > 150)) {
    return res.status(400).json({ error: '유효한 나이를 입력해주세요' });
  }

  // 검증을 통과한 뒤에야 비즈니스 로직 시작...
  const user = await createUser(req.body);
  res.status(201).json(user);
});
```

**문제점**:
- 검증 코드가 핸들러의 절반 이상을 차지하여 비즈니스 로직이 묻힘
- 에러 메시지 형식이 일관되지 않을 수 있음
- 동일한 검증 규칙을 다른 라우트에서 재사용할 수 없음
- 검증 규칙 누락 시 데이터 무결성 문제 발생

**After** (좋은 예):
```js
// validators/userSchema.js
// joi를 활용한 선언적 스키마 기반 검증
const Joi = require('joi');

const createUserSchema = Joi.object({
  name: Joi.string().min(2).max(50).required()
    .messages({
      'string.min': '이름은 2자 이상이어야 합니다',
      'string.max': '이름은 50자 이하여야 합니다',
      'any.required': '이름은 필수입니다',
    }),
  email: Joi.string().email().required()
    .messages({
      'string.email': '유효한 이메일 형식이 아닙니다',
      'any.required': '이메일은 필수입니다',
    }),
  password: Joi.string().min(8).max(128).required()
    .messages({
      'string.min': '비밀번호는 8자 이상이어야 합니다',
      'any.required': '비밀번호는 필수입니다',
    }),
  age: Joi.number().integer().min(0).max(150).optional()
    .messages({
      'number.min': '유효한 나이를 입력해주세요',
      'number.max': '유효한 나이를 입력해주세요',
    }),
});

const updateUserSchema = Joi.object({
  name: Joi.string().min(2).max(50).optional(),
  email: Joi.string().email().optional(),
  age: Joi.number().integer().min(0).max(150).optional(),
}).min(1).messages({
  'object.min': '수정할 필드를 하나 이상 포함해야 합니다',
});

module.exports = { createUserSchema, updateUserSchema };
```

```js
// middleware/validate.js
// 범용 검증 미들웨어 팩토리
function validate(schema) {
  return (req, res, next) => {
    const { error, value } = schema.validate(req.body, {
      abortEarly: false,    // 모든 에러를 한 번에 반환
      stripUnknown: true,   // 스키마에 없는 필드 제거
    });

    if (error) {
      const messages = error.details.map((d) => d.message);
      return res.status(400).json({
        success: false,
        error: { code: 'VALIDATION_ERROR', messages },
      });
    }

    // 검증 및 정제된 값으로 교체
    req.body = value;
    return next();
  };
}

module.exports = { validate };
```

```js
// routes/users.js
// 라우트 정의가 간결하고 검증 규칙 재사용 가능
const { validate } = require('../middleware/validate');
const { createUserSchema, updateUserSchema } = require('../validators/userSchema');

router.post('/users', validate(createUserSchema), userController.create);
router.put('/users/:id', validate(updateUserSchema), userController.update);
```

**개선 효과**:
- 검증 로직이 스키마로 선언적으로 정의되어 한눈에 파악 가능
- 여러 라우트에서 동일한 스키마를 재사용 가능
- `stripUnknown`으로 예상치 못한 필드가 자동 제거되어 보안 강화
- 에러 메시지 형식이 일관되고 여러 에러를 한 번에 반환

---

## 3. 에러 처리 (Error Handling)

### 검토 항목

1. 매 핸들러마다 try/catch가 중복되는가?
2. 에러 응답에 스택 트레이스가 포함되는가? (정보 노출)
3. 중앙 에러 핸들러(4개 파라미터 미들웨어)를 사용하는가?
4. 커스텀 에러 클래스(AppError 등)를 정의하고 있는가?
5. 비동기 에러가 적절히 전파되는가?

### 개선 패턴

#### 예시 1: 중앙 에러 핸들러

**Before** (나쁜 예):
```js
// routes/users.js
// 매 핸들러마다 try/catch + 에러 응답 중복
router.get('/users', async (req, res) => {
  try {
    const users = await User.find();
    res.json(users);
  } catch (err) {
    console.error('사용자 조회 실패:', err);
    res.status(500).json({ error: err.message, stack: err.stack });
  }
});

router.get('/users/:id', async (req, res) => {
  try {
    const user = await User.findById(req.params.id);
    if (!user) {
      return res.status(404).json({ error: '사용자를 찾을 수 없습니다' });
    }
    res.json(user);
  } catch (err) {
    console.error('사용자 상세 조회 실패:', err);
    res.status(500).json({ error: err.message, stack: err.stack });
  }
});

router.post('/users', async (req, res) => {
  try {
    const user = new User(req.body);
    await user.save();
    res.status(201).json(user);
  } catch (err) {
    // Mongoose 에러 타입별 분기도 매번 반복
    if (err.code === 11000) {
      return res.status(409).json({ error: '이미 존재하는 이메일입니다' });
    }
    if (err.name === 'ValidationError') {
      return res.status(400).json({ error: err.message });
    }
    console.error('사용자 생성 실패:', err);
    res.status(500).json({ error: err.message, stack: err.stack });
  }
});
```

**문제점**:
- try/catch + 에러 응답 코드가 모든 핸들러에 중복됨
- `err.stack`이 클라이언트에 노출되어 내부 구조 유출
- 에러 응답 형식이 핸들러마다 미묘하게 다름
- Mongoose/DB 에러 분기 로직이 여러 곳에 산재

**After** (좋은 예):
```js
// utils/AppError.js
// 커스텀 에러 클래스 — HTTP 상태 코드와 에러 코드를 포함
class AppError extends Error {
  constructor(message, statusCode, errorCode) {
    super(message);
    this.statusCode = statusCode;
    this.errorCode = errorCode || 'INTERNAL_ERROR';
    this.isOperational = true; // 예상된 에러인지 구분

    Error.captureStackTrace(this, this.constructor);
  }

  static badRequest(message, errorCode) {
    return new AppError(message, 400, errorCode || 'BAD_REQUEST');
  }

  static notFound(message) {
    return new AppError(message || '리소스를 찾을 수 없습니다', 404, 'NOT_FOUND');
  }

  static conflict(message) {
    return new AppError(message, 409, 'CONFLICT');
  }

  static unauthorized(message) {
    return new AppError(message || '인증이 필요합니다', 401, 'UNAUTHORIZED');
  }
}

module.exports = { AppError };
```

```js
// middleware/errorHandler.js
// 중앙 에러 핸들러 — 4개 파라미터 미들웨어
const { AppError } = require('../utils/AppError');

function errorHandler(err, req, res, next) {
  // Mongoose 중복 키 에러 변환
  if (err.code === 11000) {
    err = AppError.conflict('이미 존재하는 데이터입니다');
  }
  // Mongoose 유효성 검증 에러 변환
  if (err.name === 'ValidationError') {
    const messages = Object.values(err.errors).map((e) => e.message);
    err = AppError.badRequest(messages.join(', '), 'VALIDATION_ERROR');
  }
  // JSON 파싱 에러
  if (err.type === 'entity.parse.failed') {
    err = AppError.badRequest('잘못된 JSON 형식입니다');
  }

  const statusCode = err.statusCode || 500;
  const errorCode = err.errorCode || 'INTERNAL_ERROR';

  // 운영 환경에서는 스택 트레이스 숨김
  const response = {
    success: false,
    error: {
      code: errorCode,
      message: err.isOperational ? err.message : '서버 내부 오류가 발생했습니다',
    },
  };

  // 개발 환경에서만 스택 트레이스 포함
  if (process.env.NODE_ENV === 'development') {
    response.error.stack = err.stack;
  }

  // 예상하지 못한 에러만 로깅 (운영 에러는 별도 모니터링)
  if (!err.isOperational) {
    console.error('예기치 않은 에러 발생:', err);
  }

  res.status(statusCode).json(response);
}

module.exports = { errorHandler };
```

```js
// routes/users.js (개선 후)
// 핸들러가 간결해지고 에러 처리는 중앙에 위임
const asyncHandler = require('../utils/asyncHandler');
const { AppError } = require('../utils/AppError');

router.get('/users', asyncHandler(async (req, res) => {
  const users = await User.find();
  res.json({ success: true, data: users });
}));

router.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) throw AppError.notFound('사용자를 찾을 수 없습니다');
  res.json({ success: true, data: user });
}));

router.post('/users', asyncHandler(async (req, res) => {
  const user = new User(req.body);
  await user.save();
  res.status(201).json({ success: true, data: user });
}));
```

**개선 효과**:
- 핸들러에서 try/catch가 완전히 제거되어 비즈니스 로직에 집중
- 모든 에러가 일관된 형식 `{ success, error: { code, message } }`으로 응답
- 스택 트레이스가 운영 환경에서 노출되지 않음
- DB 에러 → AppError 변환이 한 곳에서 관리됨

#### 예시 2: 스택 트레이스 노출 방지

**Before** (나쁜 예):
```js
// app.js
// 에러 핸들러가 환경 구분 없이 모든 정보를 노출
app.use((err, req, res, next) => {
  // 운영 환경에서도 내부 구조가 그대로 노출됨
  res.status(500).json({
    error: err.message,
    stack: err.stack,
    // 내부 파일 경로, 라이브러리 버전 등이 공격자에게 유출
    // Error: Cannot read property 'name' of undefined
    //     at /app/src/services/userService.js:42:15
    //     at processTicksAndRejections (node:internal/process/task_queues:96:5)
  });
});
```

**문제점**:
- 스택 트레이스에 파일 경로, 라인 번호, 사용 중인 라이브러리 정보 노출
- 공격자가 서버 구조를 파악하여 표적 공격 가능
- 에러 메시지에 SQL 쿼리나 DB 테이블명이 포함될 수 있음
- 사용자에게 불필요한 기술적 정보가 전달됨

**After** (좋은 예):
```js
// middleware/errorHandler.js
// 환경별 에러 응답 분리
function errorHandler(err, req, res, next) {
  const statusCode = err.statusCode || 500;

  // 운영 환경: 최소한의 정보만 응답
  if (process.env.NODE_ENV === 'production') {
    // 500 에러는 generic 메시지로 대체 — 내부 구조 노출 차단
    const message = statusCode >= 500
      ? '서버 내부 오류가 발생했습니다. 잠시 후 다시 시도해주세요.'
      : err.message;

    return res.status(statusCode).json({
      success: false,
      error: {
        code: err.errorCode || 'INTERNAL_ERROR',
        message,
      },
    });
  }

  // 개발 환경: 디버깅을 위한 상세 정보 포함
  return res.status(statusCode).json({
    success: false,
    error: {
      code: err.errorCode || 'INTERNAL_ERROR',
      message: err.message,
      stack: err.stack,
      details: err.details || null,
    },
  });
}

module.exports = { errorHandler };
```

**개선 효과**:
- 운영 환경에서 서버 내부 구조가 외부에 노출되지 않음
- 개발 환경에서는 충분한 디버깅 정보를 확인 가능
- 5xx 에러는 generic 메시지로 대체하여 사용자에게 불안감 제거
- 보안 감사 시 정보 유출 항목 통과

#### 예시 3: 비동기 에러 전파

**Before** (나쁜 예):
```js
// services/emailService.js
// Promise 에러를 삼키거나 console.log만 하고 무시
async function sendWelcomeEmail(userId) {
  try {
    const user = await User.findById(userId);
    await mailer.send({
      to: user.email,
      subject: '가입을 환영합니다',
      html: renderTemplate('welcome', { name: user.name }),
    });
  } catch (err) {
    // 에러를 삼킴 — 호출자는 성공인 줄 앎
    console.log('이메일 전송 실패:', err);
  }
}

// routes/users.js
router.post('/users', async (req, res) => {
  const user = new User(req.body);
  await user.save();

  // 에러가 전파되지 않아 항상 성공 응답
  await sendWelcomeEmail(user.id);

  res.status(201).json(user);

  // 또는 이런 경우: Promise를 무시하고 fire-and-forget
  sendWelcomeEmail(user.id);
  // unhandled rejection 경고 발생 가능
});
```

**문제점**:
- `console.log`만 하고 에러를 삼키면 문제를 감지할 수 없음
- 호출자가 실패를 인지하지 못하고 후속 로직을 계속 진행
- fire-and-forget Promise는 unhandled rejection 경고 발생
- 운영 환경에서 이메일 전송 실패를 모니터링할 방법이 없음

**After** (좋은 예):
```js
// services/emailService.js
// 에러를 적절히 전파하고, 호출자가 중요도를 결정하도록 함
const { AppError } = require('../utils/AppError');
const logger = require('../utils/logger');

async function sendWelcomeEmail(userId) {
  const user = await User.findById(userId);
  if (!user) {
    throw AppError.notFound(`사용자 ${userId}를 찾을 수 없습니다`);
  }

  try {
    await mailer.send({
      to: user.email,
      subject: '가입을 환영합니다',
      html: renderTemplate('welcome', { name: user.name }),
    });
    logger.info(`환영 이메일 전송 완료: ${user.email}`);
  } catch (err) {
    // 에러를 로깅하고 다시 throw — 호출자가 처리 방법 결정
    logger.error('이메일 전송 실패', { userId, error: err.message });
    throw new AppError('이메일 전송에 실패했습니다', 502, 'EMAIL_SEND_FAILED');
  }
}

module.exports = { sendWelcomeEmail };
```

```js
// controllers/userController.js
// 비핵심 작업은 실패해도 전체 요청에 영향 없도록 분리
const asyncHandler = require('../utils/asyncHandler');
const logger = require('../utils/logger');

exports.create = asyncHandler(async (req, res) => {
  const user = new User(req.body);
  await user.save();

  // 이메일은 비핵심 — 실패해도 회원가입은 성공으로 처리
  sendWelcomeEmail(user.id).catch((err) => {
    logger.warn('환영 이메일 전송 실패 (비핵심)', {
      userId: user.id,
      error: err.message,
    });
    // 재시도 큐에 추가하는 등 후속 처리 가능
  });

  res.status(201).json({ success: true, data: user });
});
```

**개선 효과**:
- 에러가 삼켜지지 않고 명시적으로 전파됨
- 호출자가 에러의 중요도(핵심/비핵심)를 결정하여 적절히 처리
- 구조화된 로깅으로 운영 환경에서 문제 추적 가능
- fire-and-forget에도 `.catch()`를 명시하여 unhandled rejection 방지

---

## 4. 보안 미들웨어 (Security Middleware)

### 검토 항목

1. helmet을 사용하고 있는가?
2. CORS가 `*`로 설정되어 있는가?
3. Rate limiting이 적용되어 있는가?
4. SQL/NoSQL 인젝션에 취약한 문자열 조합 쿼리가 있는가?
5. 요청 바디 크기 제한이 설정되어 있는가?
6. CSRF 보호가 필요한 곳에 적용되어 있는가?

### 개선 패턴

#### 예시 1: helmet + CORS 적절 설정

**Before** (나쁜 예):
```js
// app.js
// 보안 헤더 미적용, CORS 완전 개방
const express = require('express');
const cors = require('cors');

const app = express();

// 모든 출처에서 모든 메서드로 접근 허용 — 보안 위협
app.use(cors());
// helmet 미사용 — X-Powered-By, CSP 등 보안 헤더 없음

// X-Powered-By 헤더가 "Express"를 노출하여 프레임워크 특정 공격 가능
// Content-Security-Policy 미설정으로 XSS 공격에 취약
// X-Frame-Options 미설정으로 클릭재킹 공격 가능
// Strict-Transport-Security 미설정으로 HTTPS 다운그레이드 가능

app.get('/api/data', (req, res) => {
  // 응답 헤더에 보안 관련 설정 없음
  res.json({ secret: 'sensitive-data' });
});

app.listen(3000);
```

**문제점**:
- `X-Powered-By: Express` 헤더가 노출되어 프레임워크 특정 취약점 타겟팅 가능
- CSP(Content Security Policy) 미설정으로 XSS 공격에 취약
- CORS `*`로 모든 도메인에서 API 호출 가능 — CSRF 및 데이터 유출 위험
- 클릭재킹, MIME 스니핑 등 기본적인 웹 공격에 무방비

**After** (좋은 예):
```js
// app.js
// 보안 헤더 적용 및 CORS 화이트리스트 설정
const express = require('express');
const helmet = require('helmet');
const cors = require('cors');

const app = express();

// helmet — 11개 이상의 보안 헤더를 한 번에 설정
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", 'data:', 'https:'],
    },
  },
  // HSTS — HTTPS 강제 (서브도메인 포함)
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true,
  },
}));

// CORS — 허용된 출처만 명시적으로 설정
const allowedOrigins = [
  'https://example.com',
  'https://admin.example.com',
];

if (process.env.NODE_ENV === 'development') {
  allowedOrigins.push('http://localhost:3000');
}

app.use(cors({
  origin: (origin, callback) => {
    // origin이 없는 경우 (서버 간 요청, curl 등) 허용 여부 결정
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('CORS 정책에 의해 차단되었습니다'));
    }
  },
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
  maxAge: 86400, // preflight 캐시 24시간
}));

app.listen(3000);
```

**개선 효과**:
- `X-Powered-By` 헤더 자동 제거로 프레임워크 정보 숨김
- CSP 설정으로 XSS 공격 벡터 차단
- CORS 화이트리스트로 허가된 도메인만 API 접근 가능
- HSTS로 HTTPS 다운그레이드 공격 방지

#### 예시 2: Rate limiting 적용

**Before** (나쁜 예):
```js
// app.js
// rate limiting 없음 — 브루트포스, DDoS 공격에 무방비
const express = require('express');
const app = express();

// 로그인 API에 제한 없음 — 비밀번호 무차별 대입 가능
app.post('/api/auth/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });
  if (!user || !await bcrypt.compare(password, user.password)) {
    return res.status(401).json({ error: '인증 실패' });
  }
  const token = jwt.sign({ id: user.id }, secret);
  res.json({ token });
});

// 회원가입 API에도 제한 없음 — 스팸 계정 대량 생성 가능
app.post('/api/auth/register', async (req, res) => {
  const user = new User(req.body);
  await user.save();
  res.status(201).json(user);
});

// 비용이 높은 검색 API에도 제한 없음 — DB 과부하 유발 가능
app.get('/api/search', async (req, res) => {
  const results = await db.query(
    'SELECT * FROM products WHERE name LIKE ?',
    [`%${req.query.q}%`]
  );
  res.json(results);
});
```

**문제점**:
- 로그인 엔드포인트에 제한이 없어 브루트포스 공격에 취약
- 회원가입에 제한이 없어 봇에 의한 스팸 계정 생성 가능
- 비용이 높은 API(검색, 리포트 등)에 제한이 없어 서비스 장애 유발 가능
- DDoS 공격 시 서버 리소스가 빠르게 고갈됨

**After** (좋은 예):
```js
// middleware/rateLimiter.js
// 용도별 rate limiter 설정
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const { createClient } = require('redis');

const redisClient = createClient({ url: process.env.REDIS_URL });
redisClient.connect();

// 전역 제한 — 모든 API에 적용
const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15분
  max: 100,                    // IP당 100회
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({ sendCommand: (...args) => redisClient.sendCommand(args) }),
  message: {
    success: false,
    error: { code: 'RATE_LIMITED', message: '요청이 너무 많습니다. 잠시 후 다시 시도해주세요.' },
  },
});

// 인증 엔드포인트 — 더 엄격한 제한
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15분
  max: 5,                     // IP당 5회 (브루트포스 방지)
  store: new RedisStore({ sendCommand: (...args) => redisClient.sendCommand(args) }),
  message: {
    success: false,
    error: { code: 'AUTH_RATE_LIMITED', message: '로그인 시도가 너무 많습니다. 15분 후 다시 시도해주세요.' },
  },
});

// 비용이 높은 엔드포인트 — 별도 제한
const heavyLimiter = rateLimit({
  windowMs: 60 * 1000,        // 1분
  max: 10,                     // IP당 10회
  store: new RedisStore({ sendCommand: (...args) => redisClient.sendCommand(args) }),
  message: {
    success: false,
    error: { code: 'RATE_LIMITED', message: '검색 요청이 너무 많습니다. 잠시 후 다시 시도해주세요.' },
  },
});

module.exports = { globalLimiter, authLimiter, heavyLimiter };
```

```js
// app.js
const { globalLimiter, authLimiter, heavyLimiter } = require('./middleware/rateLimiter');

// 전역 rate limiting 적용
app.use(globalLimiter);

// 엔드포인트별 추가 제한
app.use('/api/auth/login', authLimiter);
app.use('/api/auth/register', authLimiter);
app.use('/api/search', heavyLimiter);
```

**개선 효과**:
- 브루트포스 공격 시 15분간 5회로 제한하여 실질적 방어
- Redis 기반 store로 멀티 인스턴스 환경에서도 일관된 제한 적용
- 엔드포인트별 차등 제한으로 정상 사용자에게는 영향 최소화
- `Retry-After` 표준 헤더를 자동으로 포함하여 클라이언트가 대기 시간 인지 가능

#### 예시 3: SQL 인젝션 방지

**Before** (나쁜 예):
```js
// routes/users.js
// 문자열 조합으로 SQL 쿼리 생성 — 인젝션 공격에 취약
router.get('/users/:id', async (req, res) => {
  // 공격자가 id에 "1 OR 1=1" 입력 시 전체 사용자 데이터 유출
  const query = `SELECT * FROM users WHERE id = ${req.params.id}`;
  const users = await db.query(query);
  res.json(users);
});

router.get('/users', async (req, res) => {
  // 검색어에 SQL 코드 삽입 가능
  const query = `SELECT * FROM users WHERE name LIKE '%${req.query.search}%'`;
  const users = await db.query(query);
  res.json(users);
});

router.post('/users', async (req, res) => {
  // req.body를 직접 문자열로 삽입 — DROP TABLE 등 치명적 공격 가능
  const { name, email } = req.body;
  const query = `INSERT INTO users (name, email) VALUES ('${name}', '${email}')`;
  await db.query(query);
  res.status(201).json({ message: '생성됨' });
});

// NoSQL 인젝션 예 (MongoDB)
router.post('/login', async (req, res) => {
  // { "email": {"$gt": ""}, "password": {"$gt": ""} } 로 인증 우회 가능
  const user = await User.findOne({
    email: req.body.email,
    password: req.body.password,
  });
  res.json(user);
});
```

**문제점**:
- 문자열 조합 쿼리는 SQL 인젝션의 가장 흔한 원인
- 공격자가 `1; DROP TABLE users--` 같은 입력으로 테이블 삭제 가능
- NoSQL에서도 연산자 주입(`$gt`, `$ne` 등)으로 인증 우회 가능
- 모든 사용자 데이터 유출, 데이터 변조, 서비스 중단 위험

**After** (좋은 예):
```js
// routes/users.js
// 파라미터 바인딩과 입력 검증으로 인젝션 방지
router.get('/users/:id', async (req, res) => {
  // 파라미터 바인딩 — 값이 자동으로 이스케이프됨
  const users = await db.query(
    'SELECT id, name, email FROM users WHERE id = ?',
    [req.params.id]
  );
  res.json(users);
});

router.get('/users', async (req, res) => {
  // 파라미터 바인딩 + LIKE 절에서도 안전
  const search = req.query.search || '';
  const users = await db.query(
    'SELECT id, name, email FROM users WHERE name LIKE ?',
    [`%${search}%`]
  );
  res.json(users);
});

router.post('/users', async (req, res) => {
  // 파라미터 바인딩으로 INSERT
  const { name, email } = req.body;
  await db.query(
    'INSERT INTO users (name, email) VALUES (?, ?)',
    [name, email]
  );
  res.status(201).json({ message: '생성됨' });
});

// NoSQL 인젝션 방지 (MongoDB)
router.post('/login', async (req, res) => {
  // 입력값을 문자열로 강제 변환하여 연산자 주입 차단
  const email = String(req.body.email);
  const password = String(req.body.password);

  const user = await User.findOne({ email });
  if (!user || !await bcrypt.compare(password, user.hashedPassword)) {
    return res.status(401).json({
      success: false,
      error: { message: '이메일 또는 비밀번호가 올바르지 않습니다' },
    });
  }
  res.json({ success: true, data: user });
});
```

**개선 효과**:
- 파라미터 바인딩으로 SQL 인젝션 원천 차단 (DB 드라이버가 자동 이스케이프)
- `String()` 강제 변환으로 NoSQL 연산자 주입 방지
- SELECT에서 필요한 컬럼만 조회하여 불필요한 데이터 노출 방지
- 비밀번호를 bcrypt로 비교하여 평문 비교 위험 제거

---

## 5. 요청/응답 패턴 (Request/Response Patterns)

### 검토 항목

1. 모든 응답이 200으로 반환되는가? (적절한 상태 코드 사용)
2. DB 엔티티가 필터링 없이 그대로 응답되는가? (민감 필드 노출)
3. `res.json()` 또는 `res.send()` 후 함수가 계속 실행되는가? (return 누락)
4. 응답 형식이 일관된가? (success/error envelope)
5. 적절한 HTTP 상태 코드를 사용하는가? (201, 204, 404, 409 등)

### 개선 패턴

#### 예시 1: 일관된 응답 형식

**Before** (나쁜 예):
```js
// routes/users.js
// 응답 형식이 핸들러마다 제각각
router.get('/users', async (req, res) => {
  const users = await User.find();
  // 배열을 직접 반환
  res.json(users);
});

router.get('/users/:id', async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) {
    // 에러 형식 1: { error: string }
    return res.status(404).json({ error: '사용자를 찾을 수 없습니다' });
  }
  // 객체를 직접 반환
  res.json(user);
});

router.post('/users', async (req, res) => {
  const user = new User(req.body);
  await user.save();
  // 또 다른 형식: { result: object, message: string }
  res.status(201).json({ result: user, message: '사용자가 생성되었습니다' });
});

router.delete('/users/:id', async (req, res) => {
  await User.findByIdAndDelete(req.params.id);
  // 상태 코드만 반환 (바디 없음)
  res.status(200).end();
});

// routes/orders.js
router.get('/orders', async (req, res) => {
  const orders = await Order.find();
  // 또 다른 형식: { data: array, total: number }
  res.json({ data: orders, total: orders.length });
});
```

**문제점**:
- 클라이언트가 응답 구조를 예측할 수 없어 파싱 로직이 복잡해짐
- 성공/실패 여부를 HTTP 상태 코드로만 판단해야 하여 에러 처리 어려움
- 페이지네이션, 메타데이터 추가 시 기존 응답 형식과 충돌
- API 문서 작성이 어려움

**After** (좋은 예):
```js
// utils/response.js
// 표준 응답 헬퍼 함수
function sendSuccess(res, data, statusCode = 200, meta = null) {
  const response = { success: true, data };
  if (meta) {
    response.meta = meta;
  }
  return res.status(statusCode).json(response);
}

function sendError(res, message, statusCode = 500, errorCode = 'INTERNAL_ERROR') {
  return res.status(statusCode).json({
    success: false,
    error: { code: errorCode, message },
  });
}

function sendPaginated(res, data, { page, limit, total }) {
  return res.json({
    success: true,
    data,
    meta: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit),
    },
  });
}

module.exports = { sendSuccess, sendError, sendPaginated };
```

```js
// routes/users.js
// 모든 응답이 일관된 형식을 따름
const { sendSuccess, sendPaginated } = require('../utils/response');
const { AppError } = require('../utils/AppError');

router.get('/users', asyncHandler(async (req, res) => {
  const page = parseInt(req.query.page, 10) || 1;
  const limit = parseInt(req.query.limit, 10) || 20;
  const { users, total } = await userService.findAll({ page, limit });
  sendPaginated(res, users, { page, limit, total });
}));

router.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await userService.findById(req.params.id);
  if (!user) throw AppError.notFound('사용자를 찾을 수 없습니다');
  sendSuccess(res, user);
}));

router.post('/users', asyncHandler(async (req, res) => {
  const user = await userService.create(req.body);
  sendSuccess(res, user, 201);
}));

router.delete('/users/:id', asyncHandler(async (req, res) => {
  await userService.remove(req.params.id);
  sendSuccess(res, null, 204);
}));
```

**개선 효과**:
- 모든 응답이 `{ success, data, meta? }` 또는 `{ success, error }` 형식으로 통일
- 클라이언트가 `response.success`로 성공/실패를 즉시 판단 가능
- 페이지네이션 메타데이터가 표준 위치에 포함됨
- 새로운 엔드포인트 추가 시에도 동일한 응답 형식 보장

#### 예시 2: DB 엔티티 필터링 (DTO 패턴)

**Before** (나쁜 예):
```js
// routes/users.js
// DB 엔티티를 그대로 클라이언트에 반환 — 민감 정보 노출
router.get('/users/:id', async (req, res) => {
  const user = await User.findById(req.params.id);
  // user 객체에 포함된 모든 필드가 노출됨:
  // {
  //   _id: "...",
  //   name: "홍길동",
  //   email: "hong@example.com",
  //   password: "$2b$10$...",         // 해시된 비밀번호
  //   salt: "abc123...",              // 솔트 값
  //   resetPasswordToken: "...",      // 비밀번호 재설정 토큰
  //   loginAttempts: 3,               // 내부 관리용 필드
  //   internalNotes: "VIP 고객",      // 관리자 메모
  //   __v: 0                          // Mongoose 버전 키
  // }
  res.json(user);
});

router.get('/users', async (req, res) => {
  const users = await User.find();
  // 전체 목록에서도 모든 필드가 노출됨
  res.json(users);
});
```

**문제점**:
- 해시된 비밀번호와 솔트가 노출되어 오프라인 크래킹 가능
- 비밀번호 재설정 토큰이 노출되면 계정 탈취 가능
- 내부 관리용 필드(로그인 시도 횟수, 관리자 메모)가 사용자에게 노출
- API 응답 크기가 불필요하게 큼

**After** (좋은 예):
```js
// dto/userDto.js
// 응답용 DTO — 필요한 필드만 명시적으로 선택
function toUserResponse(user) {
  return {
    id: user._id || user.id,
    name: user.name,
    email: user.email,
    profileImage: user.profileImage || null,
    membershipLevel: user.membershipLevel,
    createdAt: user.createdAt,
  };
}

function toUserListItem(user) {
  return {
    id: user._id || user.id,
    name: user.name,
    profileImage: user.profileImage || null,
  };
}

function toUserProfile(user, includePrivate = false) {
  const base = toUserResponse(user);
  if (includePrivate) {
    // 본인 프로필 조회 시에만 포함
    return {
      ...base,
      phone: user.phone,
      address: user.address,
      notificationSettings: user.notificationSettings,
    };
  }
  return base;
}

module.exports = { toUserResponse, toUserListItem, toUserProfile };
```

```js
// controllers/userController.js
const { toUserResponse, toUserListItem, toUserProfile } = require('../dto/userDto');
const { sendSuccess, sendPaginated } = require('../utils/response');

exports.getAll = asyncHandler(async (req, res) => {
  const { users, total } = await userService.findAll(req.query);
  // 목록은 최소한의 필드만 포함
  sendPaginated(res, users.map(toUserListItem), {
    page: req.query.page,
    limit: req.query.limit,
    total,
  });
});

exports.getById = asyncHandler(async (req, res) => {
  const user = await userService.findById(req.params.id);
  if (!user) throw AppError.notFound('사용자를 찾을 수 없습니다');

  // 본인 조회 시 추가 정보 포함
  const isOwner = req.user && req.user.id === user.id.toString();
  sendSuccess(res, toUserProfile(user, isOwner));
});
```

**개선 효과**:
- 민감 필드(password, salt, token 등)가 응답에 절대 포함되지 않음
- 목록/상세 조회 시 적절한 수준의 정보만 노출
- 본인 조회 시에만 개인정보(전화번호, 주소)를 포함하는 접근 제어
- API 응답 크기가 줄어 네트워크 효율성 향상

---

## 6. 인증/인가 (Authentication & Authorization)

### 검토 항목

1. JWT 검증 시 알고리즘이 명시되어 있는가? (`algorithms: ['HS256']`)
2. 비밀번호가 평문 저장되거나 단순 해시(MD5/SHA)를 사용하는가?
3. 인가(authorization) 체크가 각 라우트 핸들러에 인라인으로 되어 있는가?
4. 에러 메시지가 "사용자 없음" vs "비밀번호 틀림"을 구분하는가? (정보 유출)
5. 토큰 만료/갱신 전략이 있는가?

### 개선 패턴

#### 예시 1: JWT 알고리즘 고정

**Before** (나쁜 예):
```js
// middleware/auth.js
// JWT 알고리즘을 명시하지 않아 algorithm confusion 공격 가능
const jwt = require('jsonwebtoken');

function authenticate(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) {
    return res.status(401).json({ error: '토큰이 필요합니다' });
  }

  try {
    // 알고리즘 미지정 — 공격자가 none 알고리즘 또는
    // RS256 공개키를 HS256 비밀키로 사용하는 공격 가능
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (err) {
    return res.status(401).json({ error: '유효하지 않은 토큰' });
  }
}

// 토큰 발급 시에도 만료 시간 미설정
function generateToken(user) {
  // 토큰이 영원히 유효 — 탈취 시 피해가 지속됨
  return jwt.sign(
    { id: user.id, email: user.email, role: user.role },
    process.env.JWT_SECRET
  );
}

module.exports = { authenticate, generateToken };
```

**문제점**:
- 알고리즘 미지정 시 `none` 알고리즘으로 서명 없는 토큰 수락 가능
- RS256/HS256 혼동 공격(algorithm confusion)으로 토큰 위조 가능
- 만료 시간이 없어 토큰 탈취 시 영구적으로 악용 가능
- 사용자 역할 변경 시에도 기존 토큰이 계속 유효

**After** (좋은 예):
```js
// config/jwt.js
// JWT 설정을 중앙에서 관리
const JWT_CONFIG = {
  secret: process.env.JWT_SECRET,
  accessTokenExpiry: '15m',       // 액세스 토큰: 15분
  refreshTokenExpiry: '7d',       // 리프레시 토큰: 7일
  algorithms: ['HS256'],          // 허용 알고리즘 명시
  issuer: 'my-api',              // 발급자 검증
};

module.exports = JWT_CONFIG;
```

```js
// middleware/auth.js
// 알고리즘 고정 및 만료 검증이 적용된 인증 미들웨어
const jwt = require('jsonwebtoken');
const JWT_CONFIG = require('../config/jwt');

function authenticate(req, res, next) {
  const authHeader = req.headers.authorization;
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({
      success: false,
      error: { code: 'NO_TOKEN', message: '인증 토큰이 필요합니다' },
    });
  }

  const token = authHeader.split(' ')[1];

  try {
    // 알고리즘, 발급자를 명시적으로 검증
    const decoded = jwt.verify(token, JWT_CONFIG.secret, {
      algorithms: JWT_CONFIG.algorithms,
      issuer: JWT_CONFIG.issuer,
    });
    req.user = decoded;
    return next();
  } catch (err) {
    // 에러 종류에 따라 다른 응답 (클라이언트가 갱신 여부 판단)
    if (err.name === 'TokenExpiredError') {
      return res.status(401).json({
        success: false,
        error: { code: 'TOKEN_EXPIRED', message: '토큰이 만료되었습니다' },
      });
    }
    return res.status(401).json({
      success: false,
      error: { code: 'INVALID_TOKEN', message: '유효하지 않은 토큰입니다' },
    });
  }
}

module.exports = { authenticate };
```

```js
// services/tokenService.js
// 액세스/리프레시 토큰 분리 발급
const jwt = require('jsonwebtoken');
const JWT_CONFIG = require('../config/jwt');

function generateAccessToken(user) {
  return jwt.sign(
    { id: user.id, role: user.role },
    JWT_CONFIG.secret,
    {
      algorithm: 'HS256',
      expiresIn: JWT_CONFIG.accessTokenExpiry,
      issuer: JWT_CONFIG.issuer,
    }
  );
}

function generateRefreshToken(user) {
  return jwt.sign(
    { id: user.id, tokenVersion: user.tokenVersion },
    JWT_CONFIG.secret,
    {
      algorithm: 'HS256',
      expiresIn: JWT_CONFIG.refreshTokenExpiry,
      issuer: JWT_CONFIG.issuer,
    }
  );
}

module.exports = { generateAccessToken, generateRefreshToken };
```

**개선 효과**:
- `algorithms: ['HS256']` 명시로 algorithm confusion 공격 원천 차단
- 액세스 토큰 15분 만료로 탈취 시 피해 범위 최소화
- 리프레시 토큰의 `tokenVersion`으로 강제 로그아웃 구현 가능
- `issuer` 검증으로 다른 서비스의 토큰을 수락하지 않음

#### 예시 2: RBAC 미들웨어

**Before** (나쁜 예):
```js
// routes/admin.js
// 각 핸들러에서 역할 검사를 인라인으로 반복
router.get('/admin/users', authenticate, async (req, res) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: '관리자만 접근 가능합니다' });
  }
  const users = await User.find();
  res.json(users);
});

router.delete('/admin/users/:id', authenticate, async (req, res) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: '관리자만 접근 가능합니다' });
  }
  await User.findByIdAndDelete(req.params.id);
  res.status(204).end();
});

router.get('/admin/reports', authenticate, async (req, res) => {
  if (req.user.role !== 'admin' && req.user.role !== 'manager') {
    return res.status(403).json({ error: '권한이 없습니다' });
  }
  const reports = await Report.find();
  res.json(reports);
});

router.put('/admin/settings', authenticate, async (req, res) => {
  // 역할 검사 누락! 일반 사용자도 설정 변경 가능
  await Settings.update(req.body);
  res.json({ message: '설정이 변경되었습니다' });
});
```

**문제점**:
- 동일한 역할 검사 코드가 모든 핸들러에 중복
- 역할 검사 누락 시 권한 상승 취약점 발생 (settings 라우트)
- 역할이 추가될 때 모든 핸들러를 수정해야 함
- 에러 메시지 형식이 핸들러마다 다름

**After** (좋은 예):
```js
// middleware/authorize.js
// 역할 기반 접근 제어(RBAC) 미들웨어 팩토리
function authorize(...allowedRoles) {
  return (req, res, next) => {
    // authenticate 미들웨어가 선행되어야 함
    if (!req.user) {
      return res.status(401).json({
        success: false,
        error: { code: 'UNAUTHORIZED', message: '인증이 필요합니다' },
      });
    }

    if (!allowedRoles.includes(req.user.role)) {
      return res.status(403).json({
        success: false,
        error: {
          code: 'FORBIDDEN',
          message: '이 작업을 수행할 권한이 없습니다',
        },
      });
    }

    return next();
  };
}

// 리소스 소유자 검증 미들웨어
function authorizeOwnerOrRole(resourceUserIdFn, ...allowedRoles) {
  return (req, res, next) => {
    const resourceUserId = resourceUserIdFn(req);

    // 리소스 소유자이거나 허용된 역할이면 통과
    if (
      req.user.id === resourceUserId ||
      allowedRoles.includes(req.user.role)
    ) {
      return next();
    }

    return res.status(403).json({
      success: false,
      error: { code: 'FORBIDDEN', message: '이 작업을 수행할 권한이 없습니다' },
    });
  };
}

module.exports = { authorize, authorizeOwnerOrRole };
```

```js
// routes/admin.js
// 미들웨어 체이닝으로 깔끔한 권한 관리
const { authenticate } = require('../middleware/auth');
const { authorize, authorizeOwnerOrRole } = require('../middleware/authorize');

// 관리자 전용
router.get('/admin/users', authenticate, authorize('admin'), adminController.getUsers);
router.delete('/admin/users/:id', authenticate, authorize('admin'), adminController.deleteUser);

// 관리자 + 매니저 접근 가능
router.get('/admin/reports', authenticate, authorize('admin', 'manager'), adminController.getReports);

// 관리자 전용 — 누락 없이 권한 적용
router.put('/admin/settings', authenticate, authorize('admin'), adminController.updateSettings);

// 본인 또는 관리자만 프로필 수정 가능
router.put('/users/:id/profile',
  authenticate,
  authorizeOwnerOrRole((req) => req.params.id, 'admin'),
  userController.updateProfile
);
```

**개선 효과**:
- 역할 검사 로직이 미들웨어로 추출되어 중복 제거
- 새로운 라우트 추가 시 `authorize('admin')`만 추가하면 되어 누락 방지
- 역할 추가/변경 시 미들웨어만 수정 — 영향 범위 최소화
- `authorizeOwnerOrRole`로 "본인 또는 관리자" 패턴을 재사용 가능하게 구현

#### 예시 3: 인증 에러 메시지 통일

**Before** (나쁜 예):
```js
// controllers/authController.js
// 에러 메시지가 사용자 존재 여부를 드러냄
exports.login = async (req, res) => {
  const { email, password } = req.body;

  // 사용자 조회
  const user = await User.findOne({ email });
  if (!user) {
    // "사용자를 찾을 수 없습니다" → 해당 이메일이 미등록임을 노출
    return res.status(401).json({ error: '사용자를 찾을 수 없습니다' });
  }

  // 비밀번호 확인
  const isMatch = await bcrypt.compare(password, user.password);
  if (!isMatch) {
    // "비밀번호가 일치하지 않습니다" → 해당 이메일이 등록되어 있음을 확인
    return res.status(401).json({ error: '비밀번호가 일치하지 않습니다' });
  }

  const token = generateToken(user);
  res.json({ token });
};

exports.register = async (req, res) => {
  const existing = await User.findOne({ email: req.body.email });
  if (existing) {
    // "이미 등록된 이메일입니다" → 이메일 존재 여부 확인 가능 (이메일 열거)
    return res.status(409).json({ error: '이미 등록된 이메일입니다' });
  }

  const user = new User(req.body);
  await user.save();
  res.status(201).json(user);
};
```

**문제점**:
- "사용자 없음" vs "비밀번호 틀림" 메시지로 이메일 존재 여부 확인 가능 (이메일 열거 공격)
- 공격자가 유효한 이메일 목록을 수집한 뒤 브루트포스 공격에 활용
- 회원가입에서도 이메일 존재 여부가 노출됨
- 응답 시간 차이로도 존재 여부를 추론할 수 있음 (타이밍 공격)

**After** (좋은 예):
```js
// controllers/authController.js
// 인증 에러 메시지를 통일하여 정보 유출 방지
const bcrypt = require('bcrypt');
const { generateAccessToken, generateRefreshToken } = require('../services/tokenService');
const { sendSuccess, sendError } = require('../utils/response');

// 통일된 인증 실패 메시지
const AUTH_FAILED_MESSAGE = '이메일 또는 비밀번호가 올바르지 않습니다';

exports.login = asyncHandler(async (req, res) => {
  const { email, password } = req.body;

  const user = await User.findOne({ email }).select('+password');

  // 사용자가 없어도 bcrypt.compare 실행 — 타이밍 공격 방지
  // 더미 해시와 비교하여 응답 시간을 일정하게 유지
  const dummyHash = '$2b$10$dummyhashfortimingattempt000000000000000000';
  const isMatch = await bcrypt.compare(
    password,
    user ? user.password : dummyHash
  );

  if (!user || !isMatch) {
    // 사용자 없음과 비밀번호 틀림을 동일한 메시지로 응답
    return sendError(res, AUTH_FAILED_MESSAGE, 401, 'AUTH_FAILED');
  }

  // 로그인 성공
  const accessToken = generateAccessToken(user);
  const refreshToken = generateRefreshToken(user);

  sendSuccess(res, {
    accessToken,
    refreshToken,
    user: { id: user.id, name: user.name, email: user.email },
  });
});

exports.register = asyncHandler(async (req, res) => {
  const { email, name, password } = req.body;

  // 이메일 중복 여부와 관계없이 동일한 응답
  const existing = await User.findOne({ email });
  if (existing) {
    // 기존 사용자가 있어도 "성공"처럼 응답 (이메일 열거 방지)
    // 실제로는 등록 확인 이메일을 보내는 방식으로 처리
    return sendSuccess(res, {
      message: '등록 확인 이메일을 발송했습니다. 이메일을 확인해주세요.',
    }, 200);
  }

  const hashedPassword = await bcrypt.hash(password, 12);
  const user = new User({ email, name, password: hashedPassword });
  await user.save();

  // 실제 등록 후에도 동일한 메시지
  sendSuccess(res, {
    message: '등록 확인 이메일을 발송했습니다. 이메일을 확인해주세요.',
  }, 200);
});
```

**개선 효과**:
- 로그인 실패 시 "이메일 또는 비밀번호가 올바르지 않습니다" 통일 메시지로 이메일 열거 방지
- 더미 해시 비교로 사용자 존재 여부에 따른 응답 시간 차이 제거 (타이밍 공격 방지)
- 회원가입에서도 이메일 중복 여부를 알려주지 않아 이메일 수집 차단
- bcrypt cost factor 12로 충분한 해싱 강도 확보

---

## 7. 성능 최적화 (Performance Optimization)

### 검토 항목

1. 요청마다 새 DB 연결을 생성하는가? (커넥션 풀 사용)
2. N+1 쿼리 패턴이 있는가?
3. 동기 블로킹 코드(fs.readFileSync 등)가 요청 경로에 있는가?
4. compression 미들웨어가 적용되어 있는가?
5. 자주 조회되는 데이터에 캐싱 전략이 있는가?

### 개선 패턴

#### 예시 1: 커넥션 풀 사용

**Before** (나쁜 예):
```js
// db.js
// 매 요청마다 새 연결을 생성하고 닫음 — 연결 고갈 및 성능 저하
const mysql = require('mysql2/promise');

async function query(sql, params) {
  // 매번 새 연결 생성 — TCP 핸드셰이크, 인증 과정 반복
  const connection = await mysql.createConnection({
    host: process.env.DB_HOST,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
  });

  try {
    const [rows] = await connection.execute(sql, params);
    return rows;
  } finally {
    // 연결 닫기를 까먹으면 연결이 계속 쌓임
    await connection.end();
  }
}

module.exports = { query };
```

```js
// routes/users.js
// 한 요청에서 여러 쿼리 실행 시 매번 새 연결이 열림
router.get('/users/:id/dashboard', async (req, res) => {
  // 연결 1: 사용자 조회
  const user = await db.query('SELECT * FROM users WHERE id = ?', [req.params.id]);
  // 연결 2: 주문 조회
  const orders = await db.query('SELECT * FROM orders WHERE user_id = ?', [req.params.id]);
  // 연결 3: 알림 조회
  const notifications = await db.query('SELECT * FROM notifications WHERE user_id = ?', [req.params.id]);

  // 하나의 요청에 3개의 TCP 연결이 생성되고 파괴됨
  res.json({ user: user[0], orders, notifications });
});
```

**문제점**:
- 매 쿼리마다 TCP 연결 → 인증 → 쿼리 → 연결 종료 과정이 반복됨 (수십 ms 오버헤드)
- 동시 요청이 많으면 DB 서버의 최대 연결 수를 초과하여 장애 발생
- 연결을 닫지 않으면 연결 누수로 리소스 고갈
- 트랜잭션 처리가 어려움 (쿼리마다 다른 연결)

**After** (좋은 예):
```js
// db.js
// 커넥션 풀로 연결을 재사용 — 성능 향상 및 안정성 확보
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  waitForConnections: true,     // 풀이 가득 차면 대기
  connectionLimit: 20,           // 최대 동시 연결 수
  queueLimit: 0,                 // 대기열 무제한
  idleTimeout: 60000,            // 유휴 연결 60초 후 정리
  enableKeepAlive: true,         // TCP Keep-Alive 활성화
  keepAliveInitialDelay: 30000,  // 30초마다 Keep-Alive 패킷
});

// 단순 쿼리용 헬퍼
async function query(sql, params) {
  const [rows] = await pool.execute(sql, params);
  return rows;
}

// 트랜잭션용 헬퍼
async function transaction(callback) {
  const connection = await pool.getConnection();
  try {
    await connection.beginTransaction();
    const result = await callback(connection);
    await connection.commit();
    return result;
  } catch (err) {
    await connection.rollback();
    throw err;
  } finally {
    // 풀에 연결을 반환 (닫지 않음)
    connection.release();
  }
}

// 헬스체크용
async function healthCheck() {
  try {
    await pool.query('SELECT 1');
    return true;
  } catch {
    return false;
  }
}

module.exports = { query, transaction, healthCheck, pool };
```

```js
// routes/users.js
// 커넥션 풀에서 연결을 빌려 사용 — 오버헤드 최소화
router.get('/users/:id/dashboard', asyncHandler(async (req, res) => {
  // 풀에서 연결을 가져와 모든 쿼리에 재사용
  const [user, orders, notifications] = await Promise.all([
    db.query('SELECT id, name, email FROM users WHERE id = ?', [req.params.id]),
    db.query('SELECT * FROM orders WHERE user_id = ? ORDER BY created_at DESC LIMIT 10', [req.params.id]),
    db.query('SELECT * FROM notifications WHERE user_id = ? AND is_read = false', [req.params.id]),
  ]);

  sendSuccess(res, { user: user[0], orders, notifications });
}));
```

**개선 효과**:
- 연결 생성/종료 오버헤드 제거 — 쿼리 응답 시간 수십 ms 단축
- 최대 연결 수 제한으로 DB 서버 과부하 방지
- `Promise.all`로 독립적 쿼리를 병렬 실행하여 총 응답 시간 단축
- 트랜잭션 헬퍼로 안전한 다중 쿼리 처리 가능

#### 예시 2: N+1 쿼리 해결

**Before** (나쁜 예):
```js
// services/orderService.js
// N+1 쿼리 — 사용자 수만큼 추가 쿼리 발생
async function getOrdersWithUsers() {
  // 1번 쿼리: 전체 주문 조회
  const orders = await db.query('SELECT * FROM orders ORDER BY created_at DESC LIMIT 50');

  // N번 쿼리: 각 주문의 사용자 정보를 개별 조회
  const result = [];
  for (const order of orders) {
    // 주문이 50개면 사용자 조회 쿼리가 50번 실행됨
    const user = await db.query(
      'SELECT id, name, email FROM users WHERE id = ?',
      [order.user_id]
    );
    // 각 주문의 상품 정보도 개별 조회
    const items = await db.query(
      'SELECT * FROM order_items WHERE order_id = ?',
      [order.id]
    );
    result.push({
      ...order,
      user: user[0],
      items,
    });
  }

  // 총 쿼리 수: 1 + 50 + 50 = 101개 (주문 50개 기준)
  return result;
}
```

**문제점**:
- 주문 N개에 대해 1(주문) + N(사용자) + N(상품) = 2N+1 쿼리 실행
- 데이터가 증가할수록 선형적으로 쿼리 수가 증가
- DB 서버에 과도한 부하를 주고 응답 시간이 급격히 증가
- 네트워크 왕복 시간(RTT)이 쿼리 수만큼 누적됨

**After** (좋은 예):
```js
// services/orderService.js
// JOIN 또는 WHERE IN으로 일괄 조회 — 쿼리 수를 고정
async function getOrdersWithUsers() {
  // 방법 1: JOIN으로 한 번에 조회
  const orders = await db.query(`
    SELECT
      o.id AS order_id,
      o.total_amount,
      o.status,
      o.created_at,
      u.id AS user_id,
      u.name AS user_name,
      u.email AS user_email
    FROM orders o
    INNER JOIN users u ON o.user_id = u.id
    ORDER BY o.created_at DESC
    LIMIT 50
  `);

  // 주문 ID 목록 추출
  const orderIds = orders.map((o) => o.order_id);

  if (orderIds.length === 0) return [];

  // 방법 2: WHERE IN으로 관련 상품 일괄 조회 (2번째 쿼리)
  const items = await db.query(
    `SELECT * FROM order_items WHERE order_id IN (${orderIds.map(() => '?').join(',')})`,
    orderIds
  );

  // 메모리에서 그룹핑 — DB 쿼리 없이 JavaScript로 조합
  const itemsByOrderId = items.reduce((acc, item) => {
    if (!acc[item.order_id]) acc[item.order_id] = [];
    acc[item.order_id].push(item);
    return acc;
  }, {});

  // 최종 결과 조합
  const result = orders.map((order) => ({
    id: order.order_id,
    totalAmount: order.total_amount,
    status: order.status,
    createdAt: order.created_at,
    user: {
      id: order.user_id,
      name: order.user_name,
      email: order.user_email,
    },
    items: itemsByOrderId[order.order_id] || [],
  }));

  // 총 쿼리 수: 2개 (데이터 양과 무관하게 고정)
  return result;
}
```

**개선 효과**:
- 쿼리 수가 101개 → 2개로 감소 (데이터 양과 무관하게 고정)
- JOIN으로 사용자 정보를 한 번에 가져와 네트워크 왕복 횟수 최소화
- WHERE IN으로 상품 정보를 일괄 조회 후 메모리에서 조합
- 대규모 데이터에서도 일정한 응답 시간 유지

#### 예시 3: 동기 블로킹 제거

**Before** (나쁜 예):
```js
// routes/files.js
// 동기 블로킹 코드가 요청 경로에 존재 — 전체 서버 블로킹
const fs = require('fs');
const path = require('path');
const crypto = require('crypto');

router.get('/files/:name', (req, res) => {
  const filePath = path.join(__dirname, '../uploads', req.params.name);

  // readFileSync — 이벤트 루프가 완전히 블로킹됨
  // 파일이 100MB면 다른 모든 요청이 수 초간 대기
  const data = fs.readFileSync(filePath);

  // 파일 해시 계산도 동기적으로 수행
  const hash = crypto.createHash('sha256').update(data).digest('hex');

  // JSON 설정 파일도 동기적으로 읽기
  const config = JSON.parse(
    fs.readFileSync(path.join(__dirname, '../config/settings.json'), 'utf-8')
  );

  res.json({
    file: data.toString('base64'),
    hash,
    maxFileSize: config.maxFileSize,
  });
});

// 디렉토리 목록도 동기적으로 읽기
router.get('/files', (req, res) => {
  // readdirSync — 파일이 많으면 이벤트 루프 블로킹
  const files = fs.readdirSync(path.join(__dirname, '../uploads'));
  const stats = files.map((file) => {
    // 각 파일의 stat도 동기적으로 조회
    const stat = fs.statSync(path.join(__dirname, '../uploads', file));
    return { name: file, size: stat.size, modified: stat.mtime };
  });
  res.json(stats);
});
```

**문제점**:
- `readFileSync`로 이벤트 루프가 블로킹되어 다른 모든 요청이 대기
- 큰 파일 읽기 시 수 초간 서버 전체가 응답 불가
- `readdirSync` + `statSync` 조합은 파일 수에 비례하여 블로킹 시간 증가
- Node.js의 비동기 장점을 완전히 상실

**After** (좋은 예):
```js
// routes/files.js
// 비동기 API와 스트리밍으로 이벤트 루프 블로킹 방지
const fs = require('fs');
const fsp = require('fs/promises');
const path = require('path');
const crypto = require('crypto');
const { pipeline } = require('stream/promises');

// 설정 파일은 앱 시작 시 한 번만 동기적으로 읽기 (요청 경로 밖)
const config = JSON.parse(
  fs.readFileSync(path.join(__dirname, '../config/settings.json'), 'utf-8')
);

router.get('/files/:name', asyncHandler(async (req, res) => {
  const filePath = path.join(__dirname, '../uploads', req.params.name);

  // 파일 존재 여부 확인 (비동기)
  try {
    await fsp.access(filePath);
  } catch {
    throw AppError.notFound('파일을 찾을 수 없습니다');
  }

  const stat = await fsp.stat(filePath);

  // 큰 파일은 스트리밍으로 전송 — 메모리에 전체 로드하지 않음
  if (stat.size > 10 * 1024 * 1024) { // 10MB 이상
    res.setHeader('Content-Type', 'application/octet-stream');
    res.setHeader('Content-Length', stat.size);
    const readStream = fs.createReadStream(filePath);
    return pipeline(readStream, res);
  }

  // 작은 파일은 비동기로 읽기
  const data = await fsp.readFile(filePath);

  // 해시 계산도 스트리밍으로 (CPU 블로킹 최소화)
  const hash = await new Promise((resolve, reject) => {
    const hashStream = crypto.createHash('sha256');
    const fileStream = fs.createReadStream(filePath);
    fileStream.on('data', (chunk) => hashStream.update(chunk));
    fileStream.on('end', () => resolve(hashStream.digest('hex')));
    fileStream.on('error', reject);
  });

  sendSuccess(res, {
    file: data.toString('base64'),
    hash,
    size: stat.size,
    maxFileSize: config.maxFileSize,
  });
}));

router.get('/files', asyncHandler(async (req, res) => {
  const uploadDir = path.join(__dirname, '../uploads');

  // readdir 비동기 버전 사용
  const files = await fsp.readdir(uploadDir);

  // 모든 stat을 병렬로 조회
  const stats = await Promise.all(
    files.map(async (file) => {
      const stat = await fsp.stat(path.join(uploadDir, file));
      return { name: file, size: stat.size, modified: stat.mtime };
    })
  );

  sendSuccess(res, stats);
}));
```

**개선 효과**:
- 비동기 `fs/promises` API로 이벤트 루프 블로킹 완전 제거
- 큰 파일은 스트리밍으로 전송하여 메모리 사용량 최소화
- 설정 파일은 앱 시작 시 한 번만 읽어 요청 경로에서 I/O 제거
- `Promise.all`로 stat 조회를 병렬화하여 응답 시간 단축

---

## 8. 통합 체크리스트

### 미들웨어 패턴
- [ ] 미들웨어 순서가 보안 → 파싱 → 로깅 → 인증 → 라우트 → 에러 순서인가?
- [ ] 모든 async 핸들러에 에러 래핑(asyncHandler)이 적용되어 있는가?
- [ ] 모든 분기에서 `next()` 또는 응답이 호출되는가?
- [ ] `res.json()` / `res.send()` 후 `return`으로 함수가 종료되는가?
- [ ] 서드파티 미들웨어가 최신 보안 패치를 포함하는가?

### 라우트 구조화
- [ ] 라우트가 리소스별로 별도 파일로 분리되어 있는가?
- [ ] 핸들러에 비즈니스 로직이 직접 포함되지 않는가? (controller → service 분리)
- [ ] 요청 바디/파라미터에 스키마 기반 검증이 적용되어 있는가?
- [ ] RESTful 규약(명사형 URL, 적절한 HTTP 메서드)을 따르는가?
- [ ] 라우트 그룹에 공통 미들웨어가 적용되어 있는가?

### 에러 처리
- [ ] 중앙 에러 핸들러(`(err, req, res, next)`)가 존재하는가?
- [ ] 커스텀 에러 클래스(AppError)가 정의되어 있는가?
- [ ] 운영 환경에서 스택 트레이스가 응답에 포함되지 않는가?
- [ ] DB/외부 서비스 에러가 적절한 HTTP 에러로 변환되는가?
- [ ] 비동기 에러가 삼켜지지 않고 전파되는가?

### 보안 미들웨어
- [ ] helmet이 적용되어 있는가?
- [ ] CORS origin이 화이트리스트로 제한되어 있는가? (`*` 아닌지)
- [ ] Rate limiting이 전역 및 인증 엔드포인트에 적용되어 있는가?
- [ ] SQL/NoSQL 쿼리에 파라미터 바인딩을 사용하는가?
- [ ] 요청 바디 크기 제한(`express.json({ limit })`)이 설정되어 있는가?
- [ ] CSRF 보호가 필요한 곳에 적용되어 있는가?

### 요청/응답 패턴
- [ ] 모든 응답이 일관된 형식(`{ success, data }` / `{ success, error }`)인가?
- [ ] 적절한 HTTP 상태 코드(201, 204, 400, 404, 409 등)를 사용하는가?
- [ ] DB 엔티티가 DTO로 변환되어 민감 필드가 제거되었는가?
- [ ] `res.json()` 후 `return`이 있는가? (함수 계속 실행 방지)
- [ ] 페이지네이션 응답에 메타데이터(page, total 등)가 포함되어 있는가?

### 인증/인가
- [ ] JWT 검증 시 `algorithms: ['HS256']` 등 알고리즘이 명시되어 있는가?
- [ ] 비밀번호가 bcrypt(cost 12+)로 해싱되어 저장되는가?
- [ ] 역할 기반 접근 제어(RBAC)가 미들웨어로 구현되어 있는가?
- [ ] 인증 실패 메시지가 사용자 존재 여부를 드러내지 않는가?
- [ ] 토큰 만료/갱신 전략(액세스 + 리프레시)이 구현되어 있는가?

### 성능 최적화
- [ ] DB 커넥션 풀을 사용하고 있는가? (매 요청 새 연결 X)
- [ ] N+1 쿼리 패턴이 JOIN 또는 WHERE IN으로 해결되어 있는가?
- [ ] 요청 경로에 동기 블로킹 코드(readFileSync 등)가 없는가?
- [ ] compression 미들웨어가 적용되어 있는가?
- [ ] 자주 조회되는 데이터에 캐싱(Redis 등)이 적용되어 있는가?
- [ ] 독립적인 비동기 작업이 `Promise.all`로 병렬 실행되는가?

---

### Before/After 제공 가이드

모든 리뷰 제안은 다음 형식으로 제공합니다:

```markdown
**Before** (문제점):
[현재 코드]

**문제점**:
- 구체적인 문제 설명 (성능, 보안, 유지보수성 등)

**After** (개선안):
[개선된 코드]

**개선 효과**:
- 구체적인 수치 또는 정성적 개선 효과
```

---

**이 가이드를 활용하여 Express 4.x 프로젝트의 체계적인 코드 리뷰를 수행하세요!**
