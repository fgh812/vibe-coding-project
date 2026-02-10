# Database Schema

PostgreSQL 기반 BCD 농구 서비스 스키마

## ERD
```
BR_MST_TEAM
    ├── BR_MST_PLAYER (1:N)
    │   ├── BR_MST_PLAYER_POSITION (1:N)
    │   └── BR_MST_PLAYER_UNIFORM (1:N)
    └── BR_MST_TEAM_UNIFORM (1:N)
            └── BR_MST_PLAYER_UNIFORM (N:1)

BR_CFG_GROUP
    └── BR_CFG_CODE (1:N)
```

---

## 마스터 테이블

### BR_MST_TEAM (팀)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| TEAM_ID | UUID PK | 팀 ID |
| TEAM_NM | VARCHAR(100) | 팀명 |
| SHORT_NM | VARCHAR(10) | 약칭 (전광판용) |
| LOGO_URL | TEXT | 로고 URL |
| DEL_YN | BOOLEAN | 삭제 여부 |
| REG_DT | TIMESTAMPTZ | 등록일시 |
| UPD_DT | TIMESTAMPTZ | 수정일시 |

### BR_MST_PLAYER (선수)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| PLAYER_ID | UUID PK | 선수 ID |
| TEAM_ID | UUID FK | 팀 ID |
| PLAYER_NM | VARCHAR(50) | 이름 |
| BACK_NO | INTEGER | 등번호 |
| PHOTO_URL | TEXT | 프로필 사진 |
| PHONE_NO | VARCHAR(20) | 휴대폰 |
| BIRTH_DT | DATE | 생년월일 |
| ROLE_CD | VARCHAR(20) | 역할 (G:PLR_ROLE) |
| STAT_CD | VARCHAR(20) | 상태 (G:PLR_STAT) |
| DEL_YN | BOOLEAN | 삭제 여부 |
| REG_DT | TIMESTAMPTZ | 등록일시 |
| UPD_DT | TIMESTAMPTZ | 수정일시 |

**인덱스:**
- `UK_PLAYER_BACK_NO`: (TEAM_ID, BACK_NO) WHERE DEL_YN=false
- `IDX_PLAYER_PHONE_NO`: (PHONE_NO) WHERE DEL_YN=false

### BR_MST_TEAM_UNIFORM (팀 유니폼)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| UNIFORM_ID | UUID PK | 유니폼 ID |
| TEAM_ID | UUID FK | 팀 ID |
| UNIFORM_TYPE_CD | VARCHAR(20) | 유형 (G:TEAM_UNIF) |
| UNIFORM_NM | VARCHAR(100) | 명칭 |
| COLOR_HEX | CHAR(7) | 색상 (#FFFFFF) |
| USE_YN | BOOLEAN | 사용 여부 |
| REG_DT | TIMESTAMPTZ | 등록일시 |
| UPD_DT | TIMESTAMPTZ | 수정일시 |

**제약조건:** UK (TEAM_ID, UNIFORM_TYPE_CD)

### BR_MST_PLAYER_POSITION (선수 포지션)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| PLAYER_POS_ID | UUID PK | ID |
| PLAYER_ID | UUID FK | 선수 ID |
| POS_CD | VARCHAR(10) | 포지션 (G:PLR_POS) |
| IS_PRIMARY | BOOLEAN | 주포지션 여부 |
| REG_DT | TIMESTAMPTZ | 등록일시 |

**제약조건:** UK (PLAYER_ID, POS_CD)

### BR_MST_PLAYER_UNIFORM (선수 유니폼 사이즈)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| PLAYER_UNIFORM_ID | UUID PK | ID |
| PLAYER_ID | UUID FK | 선수 ID |
| UNIFORM_ID | UUID FK | 유니폼 ID |
| TOP_SIZE_CD | VARCHAR(10) | 상의 (G:PLR_SIZE_TOP) |
| BTM_SIZE_CD | VARCHAR(10) | 하의 (G:PLR_SIZE_BTM) |
| REG_DT | TIMESTAMPTZ | 등록일시 |
| UPD_DT | TIMESTAMPTZ | 수정일시 |

**제약조건:** UK (PLAYER_ID, UNIFORM_ID)

---

## 설정 테이블

### BR_CFG_GROUP (공통코드 그룹)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| GRP_CD | VARCHAR(20) PK | 그룹코드 |
| GRP_NM | VARCHAR(100) | 그룹명 |
| CATEGORY | VARCHAR(20) | 분류 (PLAYER/GAME/TEAM/SYSTEM) |
| IS_SYSTEM | BOOLEAN | 시스템 필수 여부 |
| USE_YN | BOOLEAN | 사용 여부 |
| REG_DT | TIMESTAMPTZ | 등록일시 |
| UPD_DT | TIMESTAMPTZ | 수정일시 |

### BR_CFG_CODE (공통코드 상세)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| GRP_CD | VARCHAR(20) PK,FK | 그룹코드 |
| DTL_CD | VARCHAR(20) PK | 상세코드 |
| DTL_NM | VARCHAR(100) | 코드명 |
| DISP_ORD | INTEGER | 정렬순서 |
| VAL_1~4 | VARCHAR(100) | 여분 속성값 |
| USE_YN | BOOLEAN | 사용 여부 |
| REG_DT | TIMESTAMPTZ | 등록일시 |
| UPD_DT | TIMESTAMPTZ | 수정일시 |

---

## 코드 그룹 정의

| GRP_CD | 용도 | 사용 테이블 |
|--------|------|------------|
| PLR_ROLE | 선수 역할 | BR_MST_PLAYER.ROLE_CD |
| PLR_STAT | 선수 상태 | BR_MST_PLAYER.STAT_CD |
| PLR_POS | 포지션 | BR_MST_PLAYER_POSITION.POS_CD |
| TEAM_UNIF | 유니폼 유형 | BR_MST_TEAM_UNIFORM.UNIFORM_TYPE_CD |
| PLR_SIZE_TOP | 상의 사이즈 | BR_MST_PLAYER_UNIFORM.TOP_SIZE_CD |
| PLR_SIZE_BTM | 하의 사이즈 | BR_MST_PLAYER_UNIFORM.BTM_SIZE_CD |

---

## 공통 규칙

- **PK**: 모든 마스터 테이블은 UUID 사용
- **삭제**: 물리 삭제 대신 DEL_YN 플래그 사용
- **시간**: TIMESTAMPTZ (타임존 포함)
- **코드 참조**: `G:그룹코드` 형식으로 표기