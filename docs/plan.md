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

## 핵심 설계 방향

| 문제 | 해결 방식 |
|------|-----------|
| 서비스마다 파라미터 종류/개수가 다름 | **EAV 하이브리드** — 메타 테이블로 필드 정의, 값 테이블에 실제값 저장 |
| 다중 선택 + 항목별 추가 입력 필요 | **MULTI_REF 패턴** — 서브아이템 테이블 분리 |
| 파라미터마다 UI 표현 방식이 다름 | **PARAM_TYPE / UI_COMPONENT 분리** — 데이터 타입과 렌더링 방식을 별개 컬럼으로 관리 |
| 서비스·필드별 담당자 관리 필요 | **TB_REGISTRATION_CONTACT** — 범위(SERVICE/PARAM)를 구분하는 단일 테이블 |

---

## 전체 ERD

```
TB_PROJECT ──< TB_PROJECT_REVISION
     │
     └──────────────────────────────────────────────────┐
TB_SERVICE ──< TB_SERVICE_PARAM_DEF ──< TB_SERVICE_SUBPARAM_DEF
                       │
                       └──> TB_REGISTRATION <───────────┘
                                  │
                                  ├──< TB_REGISTRATION_PARAM       (단순 스칼라 값)
                                  ├──< TB_REGISTRATION_CONTACT     (서비스·필드 담당자)
                                  ├──< TB_REGISTRATION_LOG         (처리 로그)
                                  └──< TB_REGISTRATION_SUBITEM ─── [TB_BLOCK 등 참조 테이블]
                                            │
                                            └──< TB_REGISTRATION_SUBITEM_PARAM
                                                          │
                                                          └── TB_SERVICE_SUBPARAM_DEF
```

---

## 설계 핵심 개념: PARAM_TYPE vs UI_COMPONENT

두 개념을 분리하는 것이 핵심이다.

| 구분 | 역할 | 예시 |
|------|------|------|
| `PARAM_TYPE` | **데이터 저장/검증** 방식 | `TEXT`, `NUMBER`, `MULTI_REF` |
| `UI_COMPONENT` | **화면 렌더링** 방식 | `TEXT_FIELD`, `BUTTON_MODAL`, `DROPDOWN` |

같은 `PARAM_TYPE=TEXT`라도 `UI_COMPONENT`에 따라 단일 입력창(TEXT_FIELD) 또는 여러 줄 입력창(TEXTAREA)으로 렌더링이 달라진다.
같은 `PARAM_TYPE=MULTI_REF`라도 인라인 체크박스(CHECKBOX_GROUP) 또는 버튼 클릭 모달(BUTTON_MODAL)로 구현이 달라진다.

**유효한 PARAM_TYPE / UI_COMPONENT 조합**

| PARAM_TYPE | UI_COMPONENT 선택지 |
|------------|---------------------|
| TEXT | `TEXT_FIELD` (기본), `TEXTAREA` |
| NUMBER | `NUMBER_FIELD` |
| DATE | `DATE_PICKER` |
| SELECT | `DROPDOWN` (기본), `RADIO_GROUP` |
| BOOLEAN | `CHECKBOX_SINGLE`, `TOGGLE` |
| MULTI_REF | `BUTTON_MODAL` (버튼 → 모달에서 항목 선택), `CHECKBOX_GROUP` (인라인) |

---

## 테이블 설계 (Oracle 12c+ 기준)

### 1. TB_PROJECT — 프로젝트 마스터 (관리자 등록)

```sql
CREATE TABLE TB_PROJECT (
    PROJECT_ID    NUMBER         GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    PROJECT_CODE  VARCHAR2(50)   NOT NULL,
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
    REVISION_NO   NUMBER         NOT NULL,
    REVISION_NAME VARCHAR2(100),
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
    SERVICE_CODE  VARCHAR2(50)   NOT NULL,
    SERVICE_NAME  VARCHAR2(200)  NOT NULL,
    DESCRIPTION   VARCHAR2(1000),
    IS_ACTIVE     CHAR(1)        DEFAULT 'Y' NOT NULL CHECK (IS_ACTIVE IN ('Y','N')),
    DISPLAY_ORDER NUMBER         DEFAULT 0,
    CREATED_AT    TIMESTAMP      DEFAULT SYSTIMESTAMP NOT NULL,
    CONSTRAINT UQ_SERVICE_CODE UNIQUE (SERVICE_CODE)
);
```

### 4. TB_SERVICE_PARAM_DEF — 서비스별 파라미터 정의 (핵심 메타 테이블)

관리자가 서비스별 입력 필드를 정의. `PARAM_TYPE`(데이터 타입)과 `UI_COMPONENT`(렌더링 방식)를 분리 관리.

```sql
CREATE TABLE TB_SERVICE_PARAM_DEF (
    PARAM_DEF_ID      NUMBER         GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    SERVICE_ID        NUMBER         NOT NULL REFERENCES TB_SERVICE(SERVICE_ID),
    PARAM_KEY         VARCHAR2(100)  NOT NULL,
    PARAM_LABEL       VARCHAR2(200)  NOT NULL,
    -- 데이터 저장·검증 타입
    PARAM_TYPE        VARCHAR2(20)   NOT NULL
                      CHECK (PARAM_TYPE IN (
                          'TEXT', 'NUMBER', 'DATE', 'SELECT', 'BOOLEAN', 'TEXTAREA', 'MULTI_REF'
                      )),
    -- 화면 렌더링 컴포넌트 (NULL이면 PARAM_TYPE 기본값 사용)
    UI_COMPONENT      VARCHAR2(30)
                      CHECK (UI_COMPONENT IN (
                          'TEXT_FIELD',    -- 단일 텍스트 입력
                          'TEXTAREA',      -- 멀티라인 텍스트
                          'NUMBER_FIELD',  -- 숫자 입력
                          'DATE_PICKER',   -- 날짜 선택
                          'DROPDOWN',      -- 드롭다운 선택
                          'RADIO_GROUP',   -- 라디오 버튼 그룹
                          'CHECKBOX_SINGLE', -- 단일 체크박스
                          'TOGGLE',        -- 토글 스위치
                          'CHECKBOX_GROUP',-- 인라인 다중 체크박스
                          'BUTTON_MODAL'   -- 버튼 클릭 → 모달에서 항목 선택
                      )),
    IS_REQUIRED       CHAR(1)        DEFAULT 'Y' NOT NULL CHECK (IS_REQUIRED IN ('Y','N')),
    HAS_CONTACT       CHAR(1)        DEFAULT 'N' NOT NULL CHECK (HAS_CONTACT IN ('Y','N')),
                      -- Y이면 이 필드에 대한 담당자 입력란이 등록 폼에 함께 표시됨
    OPTIONS_JSON      CLOB,          -- SELECT 타입: [{"value":"v1","label":"옵션1"}, ...]
    VALIDATION_REGEX  VARCHAR2(500),
    DEFAULT_VALUE     VARCHAR2(500),
    PLACEHOLDER       VARCHAR2(200),
    DISPLAY_ORDER     NUMBER         DEFAULT 0,
    -- MULTI_REF 전용
    REF_TABLE_NAME    VARCHAR2(100), -- 참조 테이블명 (ex. 'TB_BLOCK')
    REF_LABEL_COL     VARCHAR2(100), -- 표시 컬럼 (ex. 'BLOCK_NAME')
    REF_VALUE_COL     VARCHAR2(100), -- PK 컬럼 (ex. 'BLOCK_ID')
    REF_FILTER_SQL    VARCHAR2(1000),-- 필터 조건 (ex. 'IS_ACTIVE=''Y''')
    CONSTRAINT UQ_SERVICE_PARAM UNIQUE (SERVICE_ID, PARAM_KEY)
);
```

### 5. TB_SERVICE_SUBPARAM_DEF — MULTI_REF 항목별 서브 필드 정의

```sql
CREATE TABLE TB_SERVICE_SUBPARAM_DEF (
    SUBPARAM_DEF_ID  NUMBER         GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    PARAM_DEF_ID     NUMBER         NOT NULL REFERENCES TB_SERVICE_PARAM_DEF(PARAM_DEF_ID),
    PARAM_KEY        VARCHAR2(100)  NOT NULL,
    PARAM_LABEL      VARCHAR2(200)  NOT NULL,
    PARAM_TYPE       VARCHAR2(20)   NOT NULL
                     CHECK (PARAM_TYPE IN ('TEXT','NUMBER','DATE','SELECT','BOOLEAN','TEXTAREA')),
    UI_COMPONENT     VARCHAR2(30)
                     CHECK (UI_COMPONENT IN (
                         'TEXT_FIELD','TEXTAREA','NUMBER_FIELD','DATE_PICKER',
                         'DROPDOWN','RADIO_GROUP','CHECKBOX_SINGLE','TOGGLE'
                     )),
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
    REQUESTED_BY     VARCHAR2(100)  NOT NULL,
    REQUESTED_AT     TIMESTAMP      DEFAULT SYSTIMESTAMP NOT NULL,
    PROCESSED_AT     TIMESTAMP,
    FAIL_REASON      VARCHAR2(2000),
    CONSTRAINT UQ_REGISTRATION UNIQUE (PROJECT_ID, REVISION_ID, SERVICE_ID)
);
```

### 7. TB_REGISTRATION_CONTACT — 담당자 정보 (서비스·필드 통합)

서비스 전체 담당자(문제 발생 시 알림 대상)와 특정 입력 필드의 담당자를 하나의 테이블로 관리.

```sql
CREATE TABLE TB_REGISTRATION_CONTACT (
    CONTACT_ID       NUMBER         GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    REGISTRATION_ID  NUMBER         NOT NULL REFERENCES TB_REGISTRATION(REGISTRATION_ID),
    -- 담당자 범위 구분
    CONTACT_SCOPE    VARCHAR2(10)   NOT NULL CHECK (CONTACT_SCOPE IN ('SERVICE', 'PARAM')),
    PARAM_DEF_ID     NUMBER         REFERENCES TB_SERVICE_PARAM_DEF(PARAM_DEF_ID),
    -- CONTACT_SCOPE='SERVICE' → PARAM_DEF_ID NULL  (서비스 전체 담당자)
    -- CONTACT_SCOPE='PARAM'   → PARAM_DEF_ID NOT NULL (특정 필드 담당자)
    CONTACT_NAME     VARCHAR2(200)  NOT NULL,
    CONTACT_EMAIL    VARCHAR2(200),
    CONTACT_PHONE    VARCHAR2(50),
    IS_PRIMARY       CHAR(1)        DEFAULT 'Y' NOT NULL CHECK (IS_PRIMARY IN ('Y','N')),
    -- Y: 주담당자 (알림 1순위), N: 부담당자
    DISPLAY_ORDER    NUMBER         DEFAULT 0,
    CONSTRAINT CHK_PARAM_SCOPE CHECK (
        (CONTACT_SCOPE = 'SERVICE' AND PARAM_DEF_ID IS NULL) OR
        (CONTACT_SCOPE = 'PARAM'   AND PARAM_DEF_ID IS NOT NULL)
    )
);
```

### 8. TB_REGISTRATION_PARAM — 단순 파라미터 값

```sql
CREATE TABLE TB_REGISTRATION_PARAM (
    PARAM_VALUE_ID   NUMBER         GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    REGISTRATION_ID  NUMBER         NOT NULL REFERENCES TB_REGISTRATION(REGISTRATION_ID),
    PARAM_DEF_ID     NUMBER         NOT NULL REFERENCES TB_SERVICE_PARAM_DEF(PARAM_DEF_ID),
    PARAM_VALUE      CLOB           NOT NULL,
    CONSTRAINT UQ_REG_PARAM UNIQUE (REGISTRATION_ID, PARAM_DEF_ID)
);
```

### 9. TB_REGISTRATION_SUBITEM — 다중 선택 항목 목록

```sql
CREATE TABLE TB_REGISTRATION_SUBITEM (
    SUBITEM_ID       NUMBER         GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    REGISTRATION_ID  NUMBER         NOT NULL REFERENCES TB_REGISTRATION(REGISTRATION_ID),
    PARAM_DEF_ID     NUMBER         NOT NULL REFERENCES TB_SERVICE_PARAM_DEF(PARAM_DEF_ID),
    REF_ID           NUMBER         NOT NULL,
    DISPLAY_ORDER    NUMBER         DEFAULT 0
);
```

### 10. TB_REGISTRATION_SUBITEM_PARAM — 선택 항목별 입력값

```sql
CREATE TABLE TB_REGISTRATION_SUBITEM_PARAM (
    SUBPARAM_VALUE_ID  NUMBER        GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    SUBITEM_ID         NUMBER        NOT NULL REFERENCES TB_REGISTRATION_SUBITEM(SUBITEM_ID),
    SUBPARAM_DEF_ID    NUMBER        NOT NULL REFERENCES TB_SERVICE_SUBPARAM_DEF(SUBPARAM_DEF_ID),
    PARAM_VALUE        CLOB          NOT NULL,
    CONSTRAINT UQ_SUBITEM_PARAM UNIQUE (SUBITEM_ID, SUBPARAM_DEF_ID)
);
```

### 11. TB_REGISTRATION_LOG — 자동 처리 로그

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
CREATE INDEX IDX_REG_PROJECT    ON TB_REGISTRATION(PROJECT_ID, REVISION_ID);
CREATE INDEX IDX_REG_STATUS     ON TB_REGISTRATION(STATUS, REQUESTED_AT);
CREATE INDEX IDX_REG_PARAM      ON TB_REGISTRATION_PARAM(REGISTRATION_ID);
CREATE INDEX IDX_REG_SUBITEM    ON TB_REGISTRATION_SUBITEM(REGISTRATION_ID, PARAM_DEF_ID);
CREATE INDEX IDX_REG_CONTACT    ON TB_REGISTRATION_CONTACT(REGISTRATION_ID, CONTACT_SCOPE);
CREATE INDEX IDX_REVISION_PROJ  ON TB_PROJECT_REVISION(PROJECT_ID, STATUS);
```

---

## Service1 설정 예시

### 관리자가 등록하는 메타 데이터

**TB_SERVICE_PARAM_DEF**

| PARAM_KEY  | PARAM_LABEL | PARAM_TYPE | UI_COMPONENT | HAS_CONTACT | REF_TABLE_NAME |
|------------|-------------|------------|--------------|-------------|----------------|
| depot_path | Depot 경로  | TEXT       | TEXT_FIELD   | N           | -              |
| base_dir   | 기본 경로   | TEXT       | TEXTAREA     | Y           | -              |
| blocks     | Block 선택  | MULTI_REF  | BUTTON_MODAL | N           | TB_BLOCK       |

- `blocks` 파라미터는 버튼 클릭 → 모달에서 Block 목록을 보여주고 선택
- `base_dir` 파라미터는 `HAS_CONTACT=Y` → 이 필드 전용 담당자 입력란이 폼에 노출

**TB_SERVICE_SUBPARAM_DEF** (`blocks` 파라미터의 블록별 서브 필드)

| PARAM_KEY      | PARAM_LABEL    | PARAM_TYPE | UI_COMPONENT |
|----------------|----------------|------------|--------------|
| assignee_name  | 담당자명        | TEXT       | TEXT_FIELD   |
| assignee_email | 담당자 이메일   | TEXT       | TEXT_FIELD   |
| assignee_phone | 담당자 연락처   | TEXT       | TEXT_FIELD   |

### 사용자 등록 시 저장되는 데이터

```
TB_REGISTRATION
  REGISTRATION_ID=101, PROJECT_ID=1, REVISION_ID=1, SERVICE_ID=1

TB_REGISTRATION_CONTACT
  [서비스 주담당자]
    CONTACT_SCOPE='SERVICE', PARAM_DEF_ID=NULL
    CONTACT_NAME='이팀장', CONTACT_EMAIL='lee@company.com', IS_PRIMARY='Y'
  [base_dir 필드 담당자]
    CONTACT_SCOPE='PARAM', PARAM_DEF_ID=<base_dir의 PARAM_DEF_ID>
    CONTACT_NAME='김담당', CONTACT_EMAIL='kim@company.com', IS_PRIMARY='Y'

TB_REGISTRATION_PARAM
  depot_path → '//depot/AA/Project1/SSS/{block}.csv'
  base_dir   → '/home/service1/base'

TB_REGISTRATION_SUBITEM (blocks 파라미터로 선택된 블록들)
  SUBITEM_ID=1, REF_ID=<BLOCK_A_ID>
  SUBITEM_ID=2, REF_ID=<BLOCK_C_ID>

TB_REGISTRATION_SUBITEM_PARAM
  SUBITEM_ID=1, assignee_name  → '홍길동'
  SUBITEM_ID=1, assignee_email → 'hong@company.com'
  SUBITEM_ID=2, assignee_name  → '김철수'
  SUBITEM_ID=2, assignee_email → 'kim@company.com'
```

### 사용자 화면 렌더링 흐름 (Service1 선택 시)

```
┌─────────────────────────────────────────────┐
│  Service1 등록                               │
├─────────────────────────────────────────────┤
│  Depot 경로 *                                │
│  [ TEXT_FIELD                             ]  │
│                                              │
│  기본 경로 *                                  │
│  [ TEXTAREA                               ]  │
│  담당자: [ 이름 ] [ 이메일 ]   ← HAS_CONTACT=Y │
│                                              │
│  Block 선택 *                                │
│  [ Block 선택하기 ▶ ]  ← BUTTON_MODAL        │
│    └─ 모달 오픈 후 선택된 블록 목록 표시:      │
│       ☑ BLOCK_A  담당자: [이름] [이메일]      │
│       ☑ BLOCK_C  담당자: [이름] [이메일]      │
│                                              │
│  서비스 주담당자 *                             │
│  [ 이름 ] [ 이메일 ] [ 연락처 ]               │
└─────────────────────────────────────────────┘
```

---

## 담당자 범위 정리

| 담당자 종류 | 저장 위치 | CONTACT_SCOPE | 용도 |
|------------|-----------|---------------|------|
| 서비스 주담당자 | TB_REGISTRATION_CONTACT | `SERVICE` | 서비스 장애 시 알림 대상 |
| 특정 필드 담당자 | TB_REGISTRATION_CONTACT | `PARAM` | 해당 필드 값 관련 문의 대상 |
| 블록별 담당자 | TB_REGISTRATION_SUBITEM_PARAM | (서브 파라미터로 저장) | 각 블록의 책임자 |

---

## 처리 흐름

```
사용자              Web Page                Backend                  DB
  │                    │                       │                      │
  ├─ Project 선택 ────>│                       │                      │
  ├─ Revision 선택 ───>│                       │                      │
  ├─ Service 선택 ────>│── 파라미터 정의 조회 ─>│── SELECT             │
  │                    │                       │   TB_SERVICE_PARAM_DEF (UI_COMPONENT 포함)
  │                    │                       │   TB_SERVICE_SUBPARAM_DEF
  │                    │<── 동적 폼 렌더링 ────│                      │
  │                    │   (UI_COMPONENT 기반으로 각 필드 렌더링)      │
  │                    │   (HAS_CONTACT=Y인 필드에 담당자 입력란 추가) │
  ├─ Block 선택 버튼 ──>│── TB_BLOCK 목록 조회 >│── SELECT TB_BLOCK    │
  │<── 모달에 목록 표시 │<─────────────────────│                      │
  ├─ 폼 입력 & 제출 ──>│── INSERT 요청 ────────>│                     │
  │                    │                       │── TB_REGISTRATION    │
  │                    │                       │── TB_REGISTRATION_CONTACT (SERVICE + PARAM)
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
- **UI_COMPONENT 유효성**: PARAM_TYPE과 UI_COMPONENT 조합이 유효한지 백엔드 또는 CHECK 제약으로 검증
