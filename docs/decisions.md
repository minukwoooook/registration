# Architecture Decision Records

Registration Page 설계 과정에서 내린 주요 결정들을 기록합니다.

---

## ADR-001: Project·Revision 관리 방식 — 관리자 사전 등록

- **날짜**: 2026-05-06
- **상태**: Accepted

### Context
사용자가 서비스를 등록할 때 어떤 Project와 Revision에 등록할지 선택해야 한다.
이 목록을 사용자가 자유롭게 입력하거나, 관리자가 미리 관리하거나, 혼합하는 방식 중 선택이 필요했다.

### Decision
관리자가 `TB_PROJECT`와 `TB_PROJECT_REVISION`에 사전 등록한 목록에서만 선택 가능하도록 한다.

### Consequences
- **긍정**: 잘못된 프로젝트/리비전 등록을 방지. 목록의 일관성 유지.
- **긍정**: 사용자 입력 실수(오탈자 등)로 인한 데이터 오염 없음.
- **부정**: 신규 프로젝트 추가 시 반드시 관리자를 통해야 함.

---

## ADR-002: 서비스 등록 후 처리 방식 — 즉시 자동 처리

- **날짜**: 2026-05-06
- **상태**: Accepted

### Context
사용자가 서비스 등록 요청 후, 담당자가 검토·승인하는 워크플로우가 필요한지 판단이 필요했다.

### Decision
등록 즉시 백엔드가 자동으로 서비스를 프로비저닝한다. 별도의 승인 단계를 두지 않는다.
`TB_REGISTRATION.STATUS`는 `PENDING → PROCESSING → COMPLETED / FAILED`로 자동 전이된다.

### Consequences
- **긍정**: 사용자가 빠르게 서비스를 사용할 수 있음. 담당자의 수동 개입 불필요.
- **부정**: 잘못된 입력이 즉시 반영될 수 있으므로 입력 유효성 검사가 더 중요해짐.
- **부정**: 향후 승인 워크플로우가 필요해지면 STATUS 흐름 및 화면을 변경해야 함.

---

## ADR-003: 동일 조합 중복 등록 불허

- **날짜**: 2026-05-06
- **상태**: Accepted

### Context
동일한 (Project + Revision + Service) 조합에 대해 여러 번 등록이 가능하게 할지, 단 한 번만 허용할지 결정이 필요했다.

### Decision
`TB_REGISTRATION`에 `UNIQUE (PROJECT_ID, REVISION_ID, SERVICE_ID)` 제약을 걸어 중복 등록을 DB 레벨에서 원천 차단한다.
파라미터 수정이 필요한 경우 기존 등록 건을 수정하는 방식으로 처리한다.

### Consequences
- **긍정**: 동일 환경에 동일 서비스가 이중으로 설정되는 운영 오류 방지.
- **긍정**: 데이터 일관성 유지가 쉬움.
- **부정**: 변경 이력(히스토리)이 필요한 경우 별도 이력 테이블을 추가해야 함.

---

## ADR-004: 서비스별 가변 파라미터 처리 — EAV 하이브리드 패턴

- **날짜**: 2026-05-06
- **상태**: Accepted

### Context
각 서비스마다 필요한 입력 필드의 종류와 개수가 다르다. 이를 처리하는 방식으로 다음을 검토했다.

| 방식 | 설명 |
|------|------|
| 서비스별 전용 테이블 | Service1_Params, Service2_Params 등 별도 테이블 생성 |
| 순수 EAV | 모든 값을 (key, value) 형태로 저장 |
| **EAV 하이브리드** | 메타 테이블로 필드 정의 + 값 테이블에 실제값 저장 |
| JSON 컬럼 | 파라미터 전체를 JSON으로 직렬화하여 단일 컬럼 저장 |

### Decision
EAV 하이브리드 패턴을 채택한다.
- `TB_SERVICE_PARAM_DEF`: 관리자가 서비스별 필드를 정의 (메타)
- `TB_REGISTRATION_PARAM`: 사용자가 입력한 실제 값 저장

### Consequences
- **긍정**: 새로운 서비스 추가 시 코드 변경 없이 메타 데이터만 추가하면 됨.
- **긍정**: 필드별 유효성 검사 규칙(정규식, 필수 여부 등)을 DB에서 관리 가능.
- **부정**: 복잡한 JOIN이 필요하여 조회 쿼리가 다소 복잡해짐.
- **부정**: 타입 안전성이 낮음 — 모든 값이 CLOB으로 저장되므로 애플리케이션 레이어에서 타입 변환 필요.

---

## ADR-005: 다중 선택 + 항목별 추가 입력 — MULTI_REF 패턴

- **날짜**: 2026-05-06
- **상태**: Accepted

### Context
Service1의 Block 선택처럼, 참조 테이블에서 여러 항목을 선택하고 선택된 각 항목마다 추가 정보(담당자 등)를 입력받는 요구사항이 있다.
단순 EAV(`TB_REGISTRATION_PARAM`)로는 이 중첩 구조를 표현할 수 없다.

### Decision
`PARAM_TYPE = 'MULTI_REF'`를 도입하고, 하위 두 테이블로 처리한다.
- `TB_REGISTRATION_SUBITEM`: 선택된 항목 목록 (참조 테이블의 PK를 REF_ID로 저장)
- `TB_REGISTRATION_SUBITEM_PARAM`: 항목별 추가 입력값

`TB_SERVICE_PARAM_DEF`에 `REF_TABLE_NAME`, `REF_LABEL_COL`, `REF_VALUE_COL` 컬럼을 추가하여 어느 테이블을 참조할지 메타로 관리한다.

### Consequences
- **긍정**: 다양한 참조 테이블(TB_BLOCK 외에도)을 코드 변경 없이 재사용 가능.
- **긍정**: 항목별 추가 필드도 `TB_SERVICE_SUBPARAM_DEF`로 유연하게 정의 가능.
- **부정**: 테이블 depth가 깊어져 전체 데이터 조회 시 JOIN이 4단계 이상 필요.
- **주의**: `REF_TABLE_NAME`을 동적으로 사용하는 쿼리는 백엔드에서 화이트리스트로 관리해야 SQL Injection을 방지할 수 있음.

---

## ADR-006: 데이터 타입과 UI 렌더링 방식 분리 — PARAM_TYPE / UI_COMPONENT

- **날짜**: 2026-05-08
- **상태**: Accepted

### Context
같은 데이터 타입이라도 화면 렌더링 방식이 다를 수 있다.
예를 들어 `TEXT` 타입이라도 단일 입력창(TEXT_FIELD)과 여러 줄 입력창(TEXTAREA)으로 다르게 표현된다.
`MULTI_REF` 타입이라도 인라인 체크박스(CHECKBOX_GROUP)와 버튼 클릭 모달(BUTTON_MODAL)로 구현이 달라진다.
`PARAM_TYPE` 하나로 두 역할을 모두 처리하면 값이 폭발적으로 늘어나고 의미가 불명확해진다.

### Decision
`TB_SERVICE_PARAM_DEF`와 `TB_SERVICE_SUBPARAM_DEF`에 컬럼을 분리한다.

| 컬럼 | 역할 | 예시 |
|------|------|------|
| `PARAM_TYPE` | 데이터 저장 및 유효성 검사 | `TEXT`, `NUMBER`, `MULTI_REF` |
| `UI_COMPONENT` | 프론트엔드 렌더링 컴포넌트 | `TEXT_FIELD`, `BUTTON_MODAL`, `DROPDOWN` |

`UI_COMPONENT`가 NULL이면 프론트엔드가 `PARAM_TYPE`에 맞는 기본 컴포넌트를 사용한다.

### Consequences
- **긍정**: 프론트엔드가 `UI_COMPONENT`만 보고 렌더링 방식을 결정하므로 로직이 단순해짐.
- **긍정**: 같은 데이터 타입에 대해 다양한 UI 표현을 코드 변경 없이 지원 가능.
- **부정**: `PARAM_TYPE`과 `UI_COMPONENT` 조합의 유효성 검사가 필요 (예: `NUMBER` 타입에 `BUTTON_MODAL`은 불가).
  - DB CHECK 제약만으로는 조합 검증이 어려우므로 애플리케이션 레이어에서 검증 필요.

---

## ADR-007: 담당자 정보 관리 — 단일 테이블 + CONTACT_SCOPE 구분

- **날짜**: 2026-05-08
- **상태**: Accepted

### Context
담당자 정보가 필요한 범위가 두 가지로 나뉜다.

1. **서비스 전체 담당자**: (Project + Revision + Service) 단위의 주담당자. 서비스 이상 발생 시 알림 대상.
2. **필드별 담당자**: 특정 입력 필드에 대한 책임자. `HAS_CONTACT=Y`인 파라미터에 한해 등록 폼에서 입력받음.

(MULTI_REF의 블록별 담당자는 블록 선택 시 함께 입력받는 서브 파라미터이므로 `TB_REGISTRATION_SUBITEM_PARAM`으로 처리.)

이를 별도 테이블로 분리하거나, 각 테이블에 컬럼을 추가하거나, 통합 테이블로 관리하는 방법을 검토했다.

### Decision
`TB_REGISTRATION_CONTACT` 단일 테이블에 `CONTACT_SCOPE` 컬럼으로 범위를 구분한다.

| CONTACT_SCOPE | PARAM_DEF_ID | 의미 |
|---|---|---|
| `SERVICE` | NULL | 서비스 전체 주담당자 |
| `PARAM` | 해당 필드 ID | 특정 파라미터 담당자 |

CHECK 제약으로 `SCOPE='SERVICE'`이면 `PARAM_DEF_ID IS NULL`, `SCOPE='PARAM'`이면 `PARAM_DEF_ID IS NOT NULL`을 강제한다.

### Consequences
- **긍정**: 담당자 조회 시 테이블 하나만 보면 됨. 알림 발송 로직이 단순해짐.
- **긍정**: 주담당자(IS_PRIMARY=Y)와 부담당자(IS_PRIMARY=N) 구분으로 알림 우선순위 관리 가능.
- **부정**: `CONTACT_SCOPE`와 `PARAM_DEF_ID`의 NULL 조합을 항상 함께 확인해야 하는 쿼리 패턴 필요.

---

## ADR-008: 파라미터별 담당자 입력 여부 — HAS_CONTACT 플래그

- **날짜**: 2026-05-08
- **상태**: Accepted

### Context
모든 입력 필드에 담당자가 필요한 것은 아니다. 어떤 필드는 담당자 입력이 필요하고 어떤 필드는 불필요하다.
이를 프론트엔드가 어떻게 판단할지 결정이 필요했다.

### Decision
`TB_SERVICE_PARAM_DEF`에 `HAS_CONTACT CHAR(1) DEFAULT 'N'` 컬럼을 추가한다.
- `HAS_CONTACT='Y'`: 해당 파라미터 입력란 아래에 담당자(이름, 이메일 등) 입력란을 함께 렌더링
- `HAS_CONTACT='N'`: 담당자 입력란 미표시

담당자 정보는 폼 제출 시 `TB_REGISTRATION_CONTACT` (CONTACT_SCOPE='PARAM')에 저장된다.

### Consequences
- **긍정**: 관리자가 메타 데이터만 변경하면 담당자 입력 여부를 동적으로 제어 가능.
- **긍정**: 프론트엔드가 이 플래그 하나로 렌더링 여부를 결정하므로 조건 분기가 단순함.
- **부정**: 담당자 필드 구성(어떤 항목을 입력받을지)이 현재는 고정(이름, 이메일, 연락처)되어 있어, 서비스마다 다른 담당자 필드가 필요한 경우 추가 설계 필요.
