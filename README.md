# 💳 신용·카드·대출 통합 데이터베이스 설계

> 카드사 신용평가 시스템(CSS)을 기반으로 한 관계형 데이터베이스 설계 프로젝트

---

## 📌 프로젝트 개요

실제 카드사 CSS(Credit Scoring System) 데이터 구조를 분석하고,  
DB 설계 원칙(정규화, ERD, 물리적 설계)에 맞게 재구조화한 프로젝트입니다.

- **기간**: 2025년
- **데이터**: 합성 데이터 (SYN_xxx) 기반 2,000명 샘플
- **기준년월**: 201807 ~ 202212
- **DBMS**: MySQL 8.0

---

## 🗂️ 프로젝트 구조

```
credit-db-project/
├── DDL/
│   └── credit_db_ddl_v3.sql       # 테이블 생성 DDL
├── ERD/
│   ├── ERD_final_v2.html          # 테이블 구조 시각화
│   └── ERD_Chen.html              # Chen 표기법 ERD
├── data/
│   ├── member_table_utf8.csv
│   ├── credit_info_utf8.csv
│   ├── basic_credit_v2_utf8.csv
│   ├── alt_credit_v2_utf8.csv
│   ├── asset_dataset_utf8.csv
│   ├── score_dataset_utf8.csv
│   ├── loan_dataset_utf8.csv
│   ├── card_dataset_utf8.csv
│   ├── card_overdue_dataset_v2_utf8.csv
│   ├── loan_overdue_dataset_v2_utf8.csv
│   ├── transaction_v2_utf8.csv
│   ├── balance_v2_utf8.csv
│   ├── normal_balance_utf8.csv
│   └── overdue_balance_utf8.csv
└── docs/
    └── ERD_물리설계_v6.xlsx        # 물리적 설계 문서
```

---

## 📐 논리적 설계 (릴레이션)

```
회원(__회원번호__, 생애주기, 성별, VIP등급, 연령, 거주지역, 직장지역)
신용정보(__신용정보번호__, 회원번호(fk))
기본신용정보(__신용정보번호(fk), 기준년월, 유형__, 한도, 이자율, 연체발생여부)
대안신용정보(__신용정보번호(fk), 기준년월__, 건강보험료납부여부, 국민연금납부여부,
             통신비납부이용금액, 보험료납부이용금액, 관리비납부이용금액,
             가스전기료납부이용금액, 통신정기결제건수, 보험정기결제건수)
자산(__회원번호(fk), 기준년월__, 순자산평가금액, 보유주택매매가합계, 총자산평가금액)
Score(__회원번호(fk), 평가일자__, 연체여부, 신용점수)
대출(__대출번호__, 회원번호(fk), 유형, 건수, 금액)
카드연체(__카드번호(fk), 기준년월, 유형__, 연체일수, 연체금액)
대출연체(__대출번호(fk), 기준년월__, 연체일수, 연체금액)
카드(__카드번호__, 회원번호(fk))
거래(__카드번호(fk), 기준년월, 유형__, 이용건수, 이용금액)
잔액(__카드번호(fk), 기준년월__)
정상잔액(__카드번호(fk), 기준년월, 유형__, 금액, 일자)
연체잔액(__카드번호(fk), 기준년월, 유형__, 금액, 일자)
```

---

## 🏗️ DB 구조 특징

### 월별 스냅샷 구조
대부분의 테이블이 `기준년월`을 PK에 포함 — 시계열 데이터 표현 가능

### ISA 구조 (슈퍼타입/서브타입)
- `잔액` → `정상잔액` / `연체잔액`
- `신용정보` → `기본신용정보` / `대안신용정보`

### 대안신용정보 확장
기존 건강보험·국민연금 납부여부 외 통신비·보험료·관리비·가스전기료 카드 납부 데이터 추가

---

## 📊 주요 분석 쿼리

### 연령대별 평균 신용점수
```sql
SELECT m.age_group,
       COUNT(*) AS 회원수,
       ROUND(AVG(s.credit_score), 0) AS 평균신용점수
FROM member m
JOIN score s ON m.member_no = s.member_no
GROUP BY m.age_group
ORDER BY 평균신용점수 DESC;
```

### 대출구간별 신용점수 & 자산
```sql
SELECT 
  CASE
    WHEN l.loan_amt = 0 THEN '무대출'
    WHEN l.loan_amt < 30000 THEN '소액'
    WHEN l.loan_amt < 60000 THEN '중액'
    ELSE '고액'
  END AS 대출구간,
  COUNT(*) AS 회원수,
  ROUND(AVG(s.credit_score), 0) AS 평균신용점수,
  ROUND(AVG(a.total_asset), 0) AS 평균자산
FROM member m
JOIN loan l ON m.member_no = l.member_no
JOIN score s ON m.member_no = s.member_no
JOIN asset a ON m.member_no = a.member_no
GROUP BY 대출구간
ORDER BY 평균신용점수 DESC;
```

### 대안신용정보 납부 패턴과 신용점수
```sql
SELECT 
  (CASE WHEN telecom_amt > 0 THEN 1 ELSE 0 END +
   CASE WHEN insurance_amt > 0 THEN 1 ELSE 0 END +
   CASE WHEN mgmt_fee_amt > 0 THEN 1 ELSE 0 END +
   CASE WHEN utility_amt > 0 THEN 1 ELSE 0 END) AS 납부항목수,
  COUNT(*) AS 회원수,
  ROUND(AVG(s.credit_score), 0) AS 평균신용점수
FROM alt_credit ac
JOIN credit_info ci ON ac.credit_info_no = ci.credit_info_no
JOIN score s ON ci.member_no = s.member_no
GROUP BY 납부항목수
ORDER BY 납부항목수;
```

---

## 🔍 설계 과정에서 발견한 난점

| 난점 | 해결 방법 |
|------|---------|
| 원천 데이터에 카드번호 없음 | AUTO_INCREMENT surrogate key 도입 |
| Wide format 원천 데이터 | Python으로 Long format 변환 (2000행→6000행) |
| PK 중복 오류 (정상잔액) | bal_type을 PK에 추가 |
| 연체 테이블 NULL FK 문제 | 카드연체/대출연체로 테이블 분리 |
| CSV 인코딩 오류 | LOAD DATA LOCAL INFILE + UTF-8 변환 |

---

## 🛠️ 실행 방법

```bash
# 1. MySQL 접속
mysql -u root -p --local-infile=1

# 2. DDL 실행
source credit_db_ddl_v3.sql

# 3. 데이터 적재 (FK 순서 준수)
USE credit_db;
LOAD DATA LOCAL INFILE 'data/member_table_utf8.csv'
INTO TABLE member
CHARACTER SET utf8mb4
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
-- 나머지 테이블도 순서대로 적재
```

---

## 📋 테이블 적재 순서

FK 제약 조건으로 인해 아래 순서 준수 필요:

1. member
2. credit_info
3. basic_credit
4. alt_credit
5. asset
6. score
7. loan
8. card
9. card_overdue
10. loan_overdue
11. transactions
12. balance
13. normal_balance
14. overdue_balance
