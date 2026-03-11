# Bunny Market의 기술 아키텍쳐

## 1. 시스템 구성
Bunny Market은 다음과 같은 주요 기술 스택으로 구성됩니다:

| 구성 요소          | 기술 스택                          |
|------------------|-----------------------------------|
| 프론트엔드         | Next.js API Routes, React, TypeScript, Tailwind CSS   |
| 프론트엔드 - 상태 관리자 | React Query, Redux          |
| 백엔드            | Next.js API Routes, TypeScript    |
| 데이터베이스       | PostgreSQL (Supabase)             |
| 인증 및 사용자 관리 | Supabase Auth (XBOX OAuth)        |
| 호스팅 및 배포     | Vercel                            |
| 패키지 관리       | npm                               |


## 2. 주요 디렉토리 구조
```
bunny-market/
├── public/                 # 정적 파일 (이미지, 아이콘 등)
├── src/
│   ├── components/         # 재사용 가능한 UI 컴포넌트
│   ├── app/              # Next.js app 라우터
│   │   ├── page.tsx       # 각 페이지의 진입점
│   │   ├── layout.tsx     # 공통 레이아웃 컴포넌트
│   │   ├── A/          # A 페이지 및 관련 파일 (모든 라우터의 기본 토대 디렉토리 구조)
│   │   │  ├── _component/ # A 페이지에서 사용하는 컴포넌트
│   │   │  ├── _hooks/     # A 관련 커스텀 Hooks
│   │   │  ├── _utils/     # A 관련 유틸리티 함수
│   │   │  ├── page.tsx    # A 페이지 컴포넌트
│   │   │  └── ...         # A 관련 추가 파일
│   │   └── ...             # 기타 페이지 디렉토리
│   ├── utils/          # 각 기능별 모듈 (Market, PriceTracker, JobBoard etc.)
│   ├── hooks/          # 커스텀 React Hooks
│   ├── types/          # TypeScript 타입 정의
│   └── services/       # API 호출 및 데이터 처리 로직
├── .env.local          # 환경 변수 파일
├── package.json        # 프로젝트 의존성 및 스크립트
├── tsconfig.json       # TypeScript 설정
├── eslint.json         # ESLint 설정
├── .prettierrc           # Prettier 설정
└── README.md           # 프로젝트 설명 및 문서
```

## 3. 코드 작성 규칙
- 코드는 react+nextjs를 사용하여 작성하되 css는 tailwind를 사용하며 피할 수 없는 경우에만 style을 직접 사용한다.
- 다크모드는 tailwind의 dark: 부분을 사용하되 대부분은 기본 dark: 에 맞춰 특수한경우 외엔 작성하지 않아도 된다.
- 코드는 SoC원칙을 준수해야한다.
- 디자인부분은 shadcn, lucide-react를 적극 이용하여 디자인이 일관되게 코드를 작성한다.
- 코드 작성후 lint검사 tsc의 noEmit을 이용한 타입검사를 한다.
- 타입은 가급적이면 unknow, any를 사용하지 않는다.
- 컴포넌트 파일은 파스칼 케이스로 작성한다. 하지만 라우터 이름은 케밥케이스를 사용한다.
- 주요 함수/컴포넌트에 jsdoc주석을 작성한다.
- 에러처리를 할때 console.error 만 쓰는것이 아니라 ErrModal를 사용하여 사용자에게 알려야 한다. (예외, 429 같은 오류는 FailModal.tsx으로 실패했다고 알림)
- alert, confrim 같은 브라우저 대화창 사용 X, react 컴포넌트(CompleteModal.tsx, ConfirmModal.tsx) 사용
- TypeScript의 interface 보단 type을 사용한다.
- 비동기 처리는 async/await와 try/catch를 사용한다.

## 4. 데이터 모델링

### `profiles` (사용자 정보)
Supabase Auth의 users 테이블과 1:1로 대응하며, 게임 내 신뢰도를 관리합니다.
| 필드명       | 타입     | 설명                         |
|------------|--------|----------------------------|
| id         | uuid (PK, references auth.users)   | 고유 사용자 ID |
| xbox_nickname | string (Unique, NN) | 사용자 닉네임 (XBOX Gamertag) |
| nickname   | string (Unique, NN) | 사용자 닉네임 (커뮤니티 내) |
| trust_score | integer (Default: 0) | 신뢰도 점수 (0~100)          |
| bio        | text    | 사용자 소개글                  |
| avatar_url | string  | 사용자 아바타 이미지 URL         |
| is_banned   | boolean (Default: false) | 사용자 차단 여부               |
| ban_metadata | jsonb   | 차단 관련 추가 정보 (차단 사유, 기간 등) |


### `items` (아이템 정보/위키)
공통 아이템 정보를 관리합니다. 거래 게시글과 경매 시스템에서 참조됩니다.
| 필드명       | 타입     | 설명                         |
|------------|--------|----------------------------|
| id         | bigint (PK)   | 고유 아이템 ID |
| name       | string (Unique, NN) | 아이템명 (예: "다이아몬드", "철곡괭이") |
| category   | enum('crop', 'mineral', 'equipment', 'artifact', 'etc') | 아이템 카테고리 |
| base_price | integer  | 기본 시세 (골드) / 기본시세가 없으면 -1              |
| image_path | string   | 아이템 이미지 경로 |
| description | text     | 아이템 설명 및 활용 팁 (md형식 예정)          |
| stat_definitions | jsonb    | 장비 아이템의 경우, 각 스탯의 정의와 효과를 JSON 형태로 저장 |

### market_posts (거래 게시글)
거래 게시글 정보를 관리합니다. 판매, 구매, 교환 게시글을 모두 포함하며, 교환 게시글의 경우 추가 필드로 교환 아이템 상세 정보를 저장합니다.
| 필드명       | 타입     | 설명                         |
|------------|--------|----------------------------|
| id         | bigint (PK)   | 고유 게시글 ID |
| author_id  | uuid (FK -> profiles.id) | 게시글 작성자 |
| item_id    | bigint (FK -> items.id) | 거래 아이템 |
| post_type  | enum ('SELL', 'BUY', 'EXCHANGE') | 게시글 유형 (판매, 구매, 교환) |
| quantity   | integer (Default: 1)  | 거래 수량                      |
| price      | integer  | 거래 가격 (골드) / (교환 시 null 가능)  |
| exchange_item_details | text | 교환 게시글의 경우, 교환 아이템 상세 정보 (예: "내 다이아몬드 3개와 상대의 철곡괭이 1개 교환") |
| metadata   | jsonb    | 추가 정보 (장비의 경우 등급/강화정보/속성 정보 등) |
| description | text     | 게시글 상세 설명 (md형식 예정) |
| status    | enum('ACTIVE', 'PENDING', 'COMPLETED', 'CANCELLED') | 게시글 상태 (활성, 거래중, 완료, 취소) |
| created_at | timestamp with time zone (Default: now()) | 게시글 생성 시간 |
| updated_at | timestamp with time zone (Default: now()) | 게시글 수정 시간 |

### `auctions` & `auction_bids` (경매 시스템)
경매 시스템을 위한 테이블로, `auctions`는 경매 게시글 정보를, `auction_bids`는 각 경매에 대한 입찰 정보를 관리합니다.
| 필드명       | 타입     | 설명                         |
|------------|--------|----------------------------|
| id         | bigint (PK)   | 고유 경매 ID |
| seller_id  | uuid (FK -> profiles.id) | 판매자 ID |
| item_id    | bigint (FK -> items.id) | 경매 아이템 |
| starting_price | integer  | 시작 가격 (골드) |
| current_price | integer  | 현재 최고 입찰 가격 (골드) |
| end_time   | timestamp with time zone | 경매 종료 시간 |
| status    | enum('ACTIVE', 'COMPLETED', 'CANCELLED') | 경매 상태 |

| 필드명       | 타입     | 설명                         |
|------------|--------|----------------------------|
| id         | bigint (PK)   | 고유 입찰 ID |
| auction_id | bigint (FK -> auctions.id) | 입찰이 속한 경매 |
| bidder_id  | uuid (FK -> profiles.id) | 입찰자 ID |
| bid_amount | integer  | 입찰 금액 (골드) |
| bid_time   | timestamp with time zone (Default: now()) | 입찰 시간 |

### `price_history` (시세 통계)
시세 통계 엔진에서 사용되는 테이블로, 거래 완료 시점의 아이템별 최종 거래 가격을 기록하여 시세 추적에 활용됩니다.
| 필드명       | 타입     | 설명                         |
|------------|--------|----------------------------|
| id         | bigint (PK)   | 고유 거래 ID |
| item_id    | bigint (FK -> items.id) | 거래 아이템 |
| final_price | integer  | 최종 거래 가격 (골드) |
| deal_type  | enum ('MARKET', 'AUCTION') | 거래 유형 (시장 거래, 경매 거래) |
| recorded_at | timestamp with time zone (Default: now()) | 거래 기록 시간 |


### `job_posts` (구인/구직 게시글)
구인/구직 게시글 정보를 관리합니다. 구인과 구직 게시글을 모두 포함하며, 작업 유형과 카테고리를 명시하여 검색 효율성을 높입니다.
| 필드명       | 타입     | 설명                         |
|------------|--------|----------------------------|
| id         | bigint (PK)   | 고유 게시글 ID |
| author_id  | uuid (FK -> profiles.id) | 게시글 작성자 |
| job_type   | enum ('HIRING', 'LOOKING') | 게시글 유형 (구인, 구직) |
| category   | enum('farming', 'mining', 'building', 'etc') | 작업 카테고리 |
| title      | string   | 게시글 제목 |
| description | text     | 게시글 상세 설명 (md형식 예정) |
| pay       | integer  | 급여 (골드) / 협의 가능 시 null |
| is_closed | boolean (Default: false) | 모집 마감 여부 |
| created_at | timestamp with time zone (Default: now()) | 게시글 생성 시간 |
| updated_at | timestamp with time zone (Default: now()) | 게시글 수정 시간 |