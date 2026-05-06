# Registration Page DB 설계안

## 개요

부서 내 다양한 Service를 Project+Revision 단위로 제공 중. 현재는 사람 간 수동 요청 방식이며,
이를 Web Page로 자동화하려고 함. 서비스마다 필요한 파라미터가 달라 유연한 구조가 필요.

**전제 조건**
- Project/Revision은 관리자가 사전 등록
- 요청 즉시 자동 처리 (승인 워크플로우 없음)
- 동일 (Project + Revision + Service) 중복 등록 불허
- 인증은 외부에서 처리 (DB 설계 범위 외)

---

## 핵심 설계 방향: EAV 하이브리드 패턴

서비스마다 파라미터 종류/개수가 다른 문제를 EAV(Entity-Attribute-Value) 패턴으로 해결.
- **메타 테이블**로 "어떤 서비스가 어떤 필드를 필요로 하는지" 정의
- **값 테이블**로 각 등록 건의 실제 입력값 저장
- 새로운 서비스 추가 시 코드 변경 없이 메타 데이터만 추가하면 됨
- **MULTI_REF 패턴**으로 "참조 테이블 다중 선택 + 항목별 추가 입력" 지원

---

## 전체 ERD

```
TB_PROJECT ──< TB_PROJECT_REVISION
     │
     └─────────────────────────────────────────┐
TB_SERVICE ──< TB_SERVICE_PARAM_DEF ──< TB_SERVICE_SUBPARAM_DEF
                      │
                      └──> TB_REGISTRATION <───┘
                                 │
                                 ├──< TB_REGISTRATION_PARAM        (단순 스칼라 값)
                                 ├──< TB_REGISTRATION_LOG          (처리 로그)
                                 └──< TB_REGISTRATION_SUBITEM ──── [TB_BLOCK 등 참조 테이블]
                                           │
                                           └──< TB_REGISTRATION_SUBITEM_PARAM
                                                       │
                                                       └── TB_SERVICE_SUBPARAM_DEF
```

---

## 테이블 설계 (Oracle 12c+ 기준)

### 1. TB_PROJECT — 프로젝트 마스터 (관리자 등록)

```sql
CREATE TABLE TB_PROJECT (
    PROJECT_ID    NUMBER         GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    PROJECT_CODE  VARCHAR2(50)   NOT NULL,   -- 'PROJECT1', 'PROJECT2'
    PROJECT_NAME  VARCHAR2(200)  NOT NULL,
    DESCRIPTION   VARCHAR2(1000),
    IS_ACTIVE     CHAR(1)        DEFAULT 'Y' NOT NULL CHECK (IS_ACTIVE IN ('Y','N')),
    CREATED_AT    TIMESTAMP      DEFAULT SYSTIMESTAMP NOT NULL,
    CREATED_BY    VARCHAR2(100),
    CONSTRAINT UQ_PROJECT_CODE UNIQUE (PROJECT_CODE)
);
```

### 2. TB_PROJECT_REVISION — 프로젝트별 리비전 (관리자 등록)

```sql
CREATE TABLE TB_PROJECT_REVISION (
    REVISION_ID   NUMBER         GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    PROJECT_ID    NUMBER         NOT NULL REFERENCES TB_PROJECT(PROJECT_ID),
    REVISION_NO   NUMBER         NOT NULL,   -- 0, 1, 2...
    REVISION_NAME VARCHAR2(100),             -- 선택적 표시명
    DESCRIPTION   VARCHAR2(1000),
    STATUS        VARCHAR2(20)   DEFAULT 'ACTIVE' NOT NULL
                                 CHECK (STATUS IN ('ACTIVE','DEPRECATED','CLOSED')),
    CREATED_AT    TIMESTAMP      DEFAULT SYSTIMESTAMP NOT NULL,
    CREATED_BY    VARCHAR2(100),
    CONSTRAINT UQ_PROJECT_REVISION UNIQUE (PROJECT_ID, REVISION_NO)
);
```

### 3. TB_SERVICE — 서비스 마스터 (관리자 등록)

```sql
CREATE TABLE TB_SERVICE (
    SERVICE_ID    NUMBER         GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    SERVICE_CODE  VARCHAR2(50)   NOT NULL,   -- 'SERVICE1', 'SERVICE2'
    SERVICE_NAME  VARCHAR2(200)  NOT NULL,
    DESCRIPTION   VARCHAR2(1000),
    IS_ACTIVE     CHAR(1)        DEFAULT 'Y' NOT NULL CHECK (IS_ACTIVE IN ('Y','N')),
    DISPLAY_ORDER NUMBER         DEFAULT 0,
    CREATED_AT    TIMESTAMP      DEFAULT SYSTIMESTAMP NOT NULL,
    CONSTRAINT UQ_SERVICE_CODE UNIQUE (SERVICE_CODE)
);
```

### 4. TB_SERVICE_PARAM_DEF — 서비스별 파라미터 정의 (핵심 메타 테이블)

서비스마다 필요한 입력 필드를 관리자가 여기에 정의해두면, 사용자 등록 화면이 자동으로 구성됨.

```sql
CREATE TABLE TB_SERVICE_PARAM_DEF (
    PARAM_DEF_ID      NUMBER         GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    SERVICE_ID        NUMBER         NOT NULL REFERENCES TB_SERVICE(SERVICE_ID),
    PARAM_KEY         VARCHAR2(100)  NOT NULL,   -- 내부 키 (snake_case 영문)
    PARAM_LABEL       VARCHAR2(200)  NOT NULL,   -- 화면 표시 레이블
    PARAM_TYPE        VARCHAR2(20)   NOT NULL
                      CHECK (PARAM_TYPE IN (
                          'TEXT',       -- 단일 텍스트 입력
                          'NUMBER',     -- 숫자 입력
                          'DATE',       -- 날짜 선택
                          'SELECT',     -- 드롭다운 선택 (OPTIONS_JSON 활용)
                          'BOOLEAN',    -- 체크박스 단일
                          'TEXTAREA',   -- 멀티라인 텍스트
                          'MULTI_REF'   -- 참조 테이블 기반 다중 선택 + 항목별 입력
                      )),
    IS_REQUIRED       CHAR(1)        DEFAULT 'Y' NOT NULL CHECK (IS_REQUIRED IN ('Y','N')),
    OPTIONS_JSON      CLOB,          -- SELECT 타입: [{"value":"v1","label":"옵션1"}, ...]
    VALIDATION_REGEX  VARCHAR2(500), -- 입력값 유효성 검사 정규식
    DEFAULT_VALUE     VARCHAR2(500),
    PLACEHOLDER       VARCHAR2(200), -- 입력 힌트 텍스트
    DISPLAY_ORDER     NUMBER         DEFAULT 0,
    -- MULTI_REF 전용 컬럼
    REF_TABLE_NAME    VARCHAR2(100), -- 참조 테이블명 (ex. 'TB_BLOCK')
    REF_LABEL_COL     VARCHAR2(100), -- 화면에 표시할 컬럼명 (ex. 'BLOCK_NAME')
    REF_VALUE_COL     VARCHAR2(100), -- PK 컬럼명 (ex. 'BLOCK_ID')
    REF_FILTER_SQL    VARCHAR2(1000),-- 필터 조건 (ex. 'IS_ACTIVE=''Y''')
    CONSTRAINT UQ_SERVICE_PARAM UNIQUE (SERVICE_ID, PARAM_KEY)
);
```

### 5. TB_SERVICE_SUBPARAM_DEF — MULTI_REF 항목별 서브 필드 정의

MULTI_REF 타입의 파라미터에서, 선택된 각 항목마다 추가로 입력받을 필드 정의.

```sql
CREATE TABLE TB_SERVICE_SUBPARAM_DEF (
    SUBPARAM_DEF_ID  NUMBER         GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    PARAM_DEF_ID     NUMBER         NOT NULL REFERENCES TB_SERVICE_PARAM_DEF(PARAM_DEF_ID),
    PARAM_KEY        VARCHAR2(100)  NOT NULL,   -- 'assignee_name', 'assignee_email'
    PARAM_LABEL      VARCHAR2(200)  NOT NULL,   -- '담당자명', '담당자 이메일'
    PARAM_TYPE       VARCHAR2(20)   NOT NULL
                     CHECK (PARAM_TYPE IN ('TEXT','NUMBER','DATE','SELECT','BOOLEAN','TEXTAREA')),
    IS_REQUIRED      CHAR(1)        DEFAULT 'Y' NOT NULL CHECK (IS_REQUIRED IN ('Y','N')),
    OPTIONS_JSON     CLOB,
    VALIDATION_REGEX VARCHAR2(500),
    DEFAULT_VALUE    VARCHAR2(500),
    PLACEHOLDER      VARCHAR2(200),
    DISPLAY_ORDER    NUMBER         DEFAULT 0,
    CONSTRAINT UQ_SUBPARAM UNIQUE (PARAM_DEF_ID, PARAM_KEY)
);
```

### 6. TB_REGISTRATION — 등록 요청 (핵심 트랜잭션 테이블)

```sql
CREATE TABLE TB_REGISTRATION (
    REGISTRATION_ID  NUMBER         GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    PROJECT_ID       NUMBER         NOT NULL REFERENCES TB_PROJECT(PROJECT_ID),
    REVISION_ID      NUMBER         NOT NULL REFERENCES TB_PROJECT_REVISION(REVISION_ID),
    SERVICE_ID       NUMBER         NOT NULL REFERENCES TB_SERVICE(SERVICE_ID),
    STATUS           VARCHAR2(20)   DEFAULT 'PENDING' NOT NULL
                     CHECK (STATUS IN ('PENDING','PROCESSING','COMPLETED','FAILED')),
    REQUESTED_BY     VARCHAR2(100)  NOT NULL,   -- 요청자 (인증 시스템에서 주입)
    REQUESTED_AT     TIMESTAMP      DEFAULT SYSTIMESTAMP NOT NULL,
    PROCESSED_AT     TIMESTAMP,
    FAIL_REASON      VARCHAR2(2000),
    CONSTRAINT UQ_REGISTRATION UNIQUE (PROJECT_ID, REVISION_ID, SERVICE_ID)
);
```

### 7. TB_REGISTRATION_PARAM — 단순 파라미터 값 (EAV 값 테이블)

```sql
CREATE TABLE TB_REGISTRATION_PARAM (
    PARAM_VALUE_ID   NUMBER         GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    REGISTRATION_ID  NUMBER         NOT NULL REFERENCES TB_REGISTRATION(REGISTRATION_ID),
    PARAM_DEF_ID     NUMBER         NOT NULL REFERENCES TB_SERVICE_PARAM_DEF(PARAM_DEF_ID),
    PARAM_VALUE      CLOB           NOT NULL,
    CONSTRAINT UQ_REG_PARAM UNIQUE (REGISTRATION_ID, PARAM_DEF_ID)
);
```

### 8. TB_REGISTRATION_SUBITEM — 다중 선택 항목 목록

MULTI_REF 타입에서 사용자가 선택한 항목들 (ex. 선택한 Block 목록).

```sql
CREATE TABLE TB_REGISTRATION_SUBITEM (
    SUBITEM_ID       NUMBER         GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    REGISTRATION_ID  NUMBER         NOT NULL REFERENCES TB_REGISTRATION(REGISTRATION_ID),
    PARAM_DEF_ID     NUMBER         NOT NULL REFERENCES TB_SERVICE_PARAM_DEF(PARAM_DEF_ID),
    REF_ID           NUMBER         NOT NULL,   -- 참조 테이블의 PK (ex. TB_BLOCK.BLOCK_ID)
    DISPLAY_ORDER    NUMBER         DEFAULT 0
);
```

### 9. TB_REGISTRATION_SUBITEM_PARAM — 선택 항목별 입력값

선택된 각 항목(Block)에 대해 입력된 추가 정보 (ex. 담당자명, 이메일).

```sql
CREATE TABLE TB_REGISTRATION_SUBITEM_PARAM (
    SUBPARAM_VALUE_ID  NUMBER        GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    SUBITEM_ID         NUMBER        NOT NULL REFERENCES TB_REGISTRATION_SUBITEM(SUBITEM_ID),
    SUBPARAM_DEF_ID    NUMBER        NOT NULL REFERENCES TB_SERVICE_SUBPARAM_DEF(SUBPARAM_DEF_ID),
    PARAM_VALUE        CLOB          NOT NULL,
    CONSTRAINT UQ_SUBITEM_PARAM UNIQUE (SUBITEM_ID, SUBPARAM_DEF_ID)
);
```

### 10. TB_REGISTRATION_LOG — 자동 처리 로그

```sql
CREATE TABLE TB_REGISTRATION_LOG (
    LOG_ID           NUMBER         GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    REGISTRATION_ID  NUMBER         NOT NULL REFERENCES TB_REGISTRATION(REGISTRATION_ID),
    LOG_LEVEL        VARCHAR2(10)   NOT NULL CHECK (LOG_LEVEL IN ('INFO','WARN','ERROR')),
    MESSAGE          CLOB,
    CREATED_AT       TIMESTAMP      DEFAULT SYSTIMESTAMP NOT NULL
);
```

---

## 인덱스 전략

```sql
CREATE INDEX IDX_REG_PROJECT   ON TB_REGISTRATION(PROJECT_ID, REVISION_ID);
CREATE INDEX IDX_REG_STATUS    ON TB_REGISTRATION(STATUS, REQUESTED_AT);
CREATE INDEX IDX_REG_PARAM     ON TB_REGISTRATION_PARAM(REGISTRATION_ID);
CREATE INDEX IDX_REG_SUBITEM   ON TB_REGISTRATION_SUBITEM(REGISTRATION_ID, PARAM_DEF_ID);
CREATE INDEX IDX_REVISION_PROJ ON TB_PROJECT_REVISION(PROJECT_ID, STATUS);
```

---

## Service1 설정 예시

### 관리자가 등록하는 메타 데이터

**TB_SERVICE_PARAM_DEF**

| PARAM_KEY  | PARAM_LABEL    | PARAM_TYPE | REF_TABLE_NAME | REF_LABEL_COL |
|------------|----------------|------------|----------------|---------------|
| depot_path | Depot 경로     | TEXT       | -              | -             |
| blocks     | Block 선택     | MULTI_REF  | TB_BLOCK       | BLOCK_NAME    |

**TB_SERVICE_SUBPARAM_DEF** (`blocks` 파라미터의 항목별 서브 필드)

| PARAM_KEY      | PARAM_LABEL      | PARAM_TYPE | IS_REQUIRED |
|----------------|------------------|------------|-------------|
| assignee_name  | 담당자명          | TEXT       | Y           |
| assignee_email | 담당자 이메일     | TEXT       | Y           |
| assignee_phone | 담당자 연락처     | TEXT       | N           |

### 사용자 등록 시 저장되는 데이터

```
TB_REGISTRATION
  REGISTRATION_ID=101, PROJECT_ID=1, REVISION_ID=1, SERVICE_ID=1, STATUS='PENDING'

TB_REGISTRATION_PARAM
  REGISTRATION_ID=101, depot_path → '//depot/AA/Project1/SSS/{block}.csv'

TB_REGISTRATION_SUBITEM
  SUBITEM_ID=1, REGISTRATION_ID=101, REF_ID=<BLOCK_A_ID>
  SUBITEM_ID=2, REGISTRATION_ID=101, REF_ID=<BLOCK_C_ID>

TB_REGISTRATION_SUBITEM_PARAM
  SUBITEM_ID=1, assignee_name  → '홍길동'
  SUBITEM_ID=1, assignee_email → 'hong@example.com'
  SUBITEM_ID=2, assignee_name  → '김철수'
  SUBITEM_ID=2, assignee_email → 'kim@example.com'
```

---

## 처리 흐름

```
사용자              Web Page                Backend                  DB
  │                    │                       │                      │
  ├─ Project 선택 ────>│                       │                      │
  ├─ Revision 선택 ───>│                       │                      │
  ├─ Service 선택 ────>│── 파라미터 정의 조회 ─>│── SELECT             │
  │                    │                       │   TB_SERVICE_PARAM_DEF
  │                    │                       │   TB_SERVICE_SUBPARAM_DEF
  │                    │<── 동적 폼 렌더링 ────│                      │
  │                    │   (MULTI_REF → 체크박스 + 항목별 입력 행)    │
  ├─ 폼 입력 & 제출 ──>│── INSERT 요청 ────────>│                     │
  │                    │                       │── TB_REGISTRATION    │
  │                    │                       │── TB_REGISTRATION_PARAM
  │                    │                       │── TB_REGISTRATION_SUBITEM
  │                    │                       │── TB_REGISTRATION_SUBITEM_PARAM
  │                    │                       │                      │
  │                    │                       ├─ 자동 처리 실행      │
  │                    │                       │── UPDATE STATUS      │
  │                    │                       │── INSERT LOG         │
  │<── 완료/실패 표시 ──│<─────────────────────│                      │
```

---

## 향후 고려사항

- **파라미터 정의 변경 시 호환성**: PARAM_DEF 삭제 대신 `IS_ACTIVE CHAR(1)` 컬럼 추가 후 비활성화
- **재처리**: 실패 건의 STATUS를 `FAILED` → `PENDING`으로 변경하는 관리자 기능
- **로그 파티셔닝**: TB_REGISTRATION_LOG는 데이터 증가 시 월별 파티셔닝 고려
- **REF_TABLE_NAME 보안**: 동적 테이블 조회는 백엔드 화이트리스트로 관리 (SQL Injection 방지)
