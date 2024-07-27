# 03. 사용자 및 권한

## 3.1 사용자 식별

- MySQL 계정을 언급할 때는 아이디와 호스트를 명시하여야 한다.
  ```
  'svc_id'@'127.0.0.1'
  'svc_id'@'192.168.0.10'
  'svc_id'@'%
  ```
- 범위가 더 좁은 것을 먼저 선택한다.
  - ex) 192.168.0.10에서 svc_id로 접근한다면, 'svc_id'@'192.168.0.10'을 이용한다.

## 3.2 사용자 계정 관리

### 1. 시스템 계정 & 일반 계정

- MySQL 8.0부터 계정은 `SYSTEM_USER` 권한을 가지느냐에 따라 시스템 계정과 일반 계정으로 구분
  - 시스템 계정 : DB 서버 관리자를 위한 계정
  - 일반 계정 : 응용 프로그램이나 개발자를 위한 계정

- 시스템 계정에서만 할 수 있는 일 들
  - 계정 관리 : 계정 생성 및 삭제, 계정의 권한 부여
  - 다른 세션 또는 그 세션에서 실행 중인 쿼리 강제 종료
  - cf) 스토어드 프로그램 생성 시, DEFINER를 타 사용자로 생성

- 내장 계정 들은 내부적으로 목적을 가지고 있으므로 삭제되지 않도록 주의
  - 'mysql.sys'@'localhost' : sys 스키마의 객체들의 DEFINER로 사용하는 계정
  - 'mysql'@'localhost' : MySQL 플러그인이 서버로 접근할 때 사용되는 계정
  - 'mysql.schema'@'localhost' : information_schema의 정의된 뷰의 DEFINER로 사용되는 계정
  - 의도적으로 잠긴 계정을 풀지 않는 한 악의적인 용도로 사용할 수 없으므로 보안을 걱정하지 않아도 된다.

### 2. 계정 생성

- GRANT, CREATE USER
  - MySQL 5.7 버전까지는 GRANT 명령으로 권한의 부여와 동시에 계정 생성이 가능했다.
  - MySQL 8.0 버전부터는 계정의 생성은 CREATE USER, 권한 부여는 GRANT로 부여

```sql
CREATE USER 'user'@'%'
	IDENTIFIED WITH 'mysql_nativee_password' BY 'password'
	REQUIRE NONE
	PASSWORD EXPIRE INTERVAL 30 DAY
	ACCOUNT LOCK
	PASSWORD HISTORY DEFAULT
	PASSWORD REUSE INTERVAL DEFAULT
	PASSWORD REQUIRE CURRENT DEFAULT;
```

- IDENTIFIED WITH : 사용자의 인증 방식과 비밀번호 설정
  - MySQL 서버에서는 다양한 인증 방식을 플러그인 형태로 제공
  - 글로벌 인증 방식 변경 : `SET GLOBAL default_authentication_plugin="mysql_native_password`
  - https://minsql.com/mysql8/C-2-A-authentificationPlugin/ (24.07.27)

- REQUIRE : 암호화된 SSL/TLS 채널을 사용할 지 여부를 설정
  - 별도로 설정하지 않으면 비암호화 채널로 연결
  - cf) Cashing SHA-2 Authentication 인증 방식을 사용하면 암호화된 채널로만 접근 가능

- PASSWORD EXPIRE : 비밀번호의 유효 기간을 설정하는 옵션
  - 개발자, DB 관리자의 비밀번호 유효기간을 주는 것은 보안상 안전하지만, 응용프로그램 접속용 계정에 유효 기간을 설정하는 것은 위험할 수 있으니 주의하자
  - `PASSWORD EXPIRE` : 만듦과 동시에 비밀번호 만료
  - `PASSWORD EXPIRE NEVER` : 계정 비밀번호의 만료 기간 없음
  - `PASSWORD EXPIRE DEFAULT` : 기본 값(default_password_lifetime) 사용
  - `PASSWORD EXPIRE INTERVAL n DAY` : 비밀번호 유효 기간을 오늘부터 n일 뒤로 설정

- PASSWORD HISTORY : 한 번 사용했던 비밀번호를 재사용하지 못하게 설정하는 옵션
  - `PASSWORD HISTORY DEFAULT` : 기본 값(password_history) 사용
  - `PASSWORD HISTORY n` : 이력을 최근 n개까지 저장, 해당 이력에 있는 비밀번호는 재사용 불가
  - 조회 예시 : `SELECT * FROM mysql.password_history`

- PASSWORD REUSE INTERVAL : 한 번 사용했던 비밀번호의 재사용 금지 기간을 설정하는 옵션
  - `PASSWORD REUSE INTERVAL DEFAULT` : 기본값(password_reuse_interval) 사용
  - `PASSWORD REUSE INTERVAL n DAY` : n일자 이후에 비밀번호를 재사용 할 수 있게 설정

- PASSWORD REQUIRE : 새로운 비밀번호로 변경할 때 현재 비밀번호를 필료로 할지 말지를 결정하는 옵션
  - `PASSWORD REQUIRE CURRENT` : 비밀번호를 변경할 때, 현재 비밀번호를 먼저 입력하도록 설정
  - `PASSWORD REQUIRE OPTIONAL` : 비밀번호를 변경할 때, 현재 비밀번호를 먼저 입력하지 않아도 되도록 설정
  - `PASSWORD REQUIRE DEFAULT` : 기본 값(password_require_current) 사용

- ACCOUNT LOCK / UNLOCK : 계정 생성 또는 ALTER USER 명렬을 통해 계정 정보를 변경할 때 계정을 사용하지 못하게 잠글지 여부 결정
  - `ACCOUNT LOCK` : 계정을 사용하지 못하게 잠금
  - `ACCOUNT UNLOCK` : 잠긴 계정을 다시 사용 가능 상태로 잠금 해제

## 3. 비밀번호 관리

### 1. 고수준 비밀번호

- 'validate_password'
  - 비밀번호를 쉽게 유추할 수 있는 단어들이 사용되지 않게 글자의 조합을 강제하거나 금칙어를 설정하는 기능도 있다.
  - MySQL 서버에서 비밀번호에 유효성 체크 규칙을 적용하려면 'validate_password' 컴포넌트를 이용

```sql
-- validate_password 컴포넌트 설치
INSTALL COMPONENT 'file://component_validate_password';

-- 설치된 컴포넌트 확인
SELECT * FROM mysql.component;

-- validate_password 컴포넌트에서 제공하는 시스템 변수 보기
SHOW GLOBAL VALIABLES LIKE 'validate_password%';
```

- 비밀번호 정책
  - `LOW` : 비밀번호 길이만 검증 (validate_password.length)
  - `MEDIUM` : 비밀번호 길이 및 숫자, 대소문자, 특수문자 배합 검증 (validate_password.mixed_case_count, validate_password.number_count, validate_password.special_char_count)
  - `STRONG` : `MEDIUM` 정책 + 금칙어 포함 여부 검증 (validate_password.dictionary_file 파일에 저장)
- 금칙어 파일 등록 방법
```sql
-- 금칙어 등록
SET GLOBAL validate_password.dictionay_file='prohibitive_word.data';
-- 비밀번호 정책 변경
SET GLOBAL validate_password.policy='STRONG';
```

### 2. 이중 비밀번호

- 일반적으로 서버의 계정 정보는 응용 프로그램 서버로부터 공용으로 사용되는 경우가 많다. 이러한 구현 특성으로 인해 DB 서버의 계정 정보는 쉽게 변경하기가 어럽다.
- 그 중에서도 계정의 비밀번호는 서비스가 실행 중인 상태에서 변경이 불가능해 몇 년동안 초기 비밀번호를 사용하는 경우도 많았다. (보안에 취약)

- 세컨더리 비밀번호
  - 비밀번호를 둘 중 하나로 사용해도 계정 인증이 되도록 함
  - 모든 응용 프로그램의 소스코드를 바꾸고 나면, 보안 상 세컨더리 비밀번호를 지우는 것이 좋다.
- 세컨더리 비밀번호 설정
```sql
-- 비밀번호 변경
ALTER USER 'root'@'localhost' IDENTIFIED BY 'old_password';

-- 비밀번호를 변경하면서, 기존 비밀번호를 세컨더리 비밀번호로 설정
ALTER USER 'root'@'localhost' IDENTIFIED BY 'old_password' RETAIN CURRENT PASSWORD;

-- 세컨더리 비밀번호 제거
ALTER USER 'root'@'localhost' DISCARD OLD PASSWORD;
```

## 3.4 권한 (Privilege)

- 정적 권한과 동적 권한
  - MySQL 5.7 까지는 글로벌 권한(DB나 Table 외 권한)과 객체 권한(DB나 Table 권한)으로 분리
  - MySQL 8.0은 정적 권한(DB 소스 코드에 명시)과 동적 권한(시작과 동시에 바꿀 수 있음)으로 분리
    - 위의 5.7까지의 권한들은 정적 권한들로 분류
    - 5.7 버전까지는 `SUPER` 권한이 DB 관리를 위해 꼭 필요한 권한이었지만, 8.0 부터는 공적 권한으로 분산됨
    - 따라서 관리자 별로 필요한 권한만 부여할 수 있게 되었다.
  - 각 권한들의 내용은 Real MySQL 65~67p 참고

- 권한 부여하기
```sql
-- 권한 부여하기
GRANT privilege_list ON db.table TO 'user'@'localhost';

-- 글로벌 권한 부여 (객체마다 부여가 불가능하므로, 전체 권한 부여)
GRANT SUPER ON *.* TO 'user'@'localhost';

-- (모든 or 특정) DB 권한 부여 (DB Table 및 스토어드 프로그램)
GRANT EVENT ON *.* TO 'user'@'localhost';
GRANT EVENT ON employees.* TO 'user'@'localhost';

-- 테이블 권한 부여
GRANT SELECT, INSERT, UPDATE, DELETE ON *.* TO 'user'@'localhost';
GRANT SELECT, INSERT, UPDATE, DELETE ON employees.* TO 'user'@'localhost';
GRANT SELECT, INSERT, UPDATE, DELETE ON employees.department TO 'user'@'localhost';
```

```sql
-- 특정 컬럼 권한 부여 (모두 조회, 특정 column 업데이트) 
GRANT SELECT, UPDATE(dept_name) ON employees.departmetn TO 'user'@'localhost';
```
- 컬럼 단위 권한이 하나라도 설정된다면, 나머지 모든 테이블의 모든 컬럼에 대해서도 권한 체크를 하기 때문에, 전체적인 성능에 영향을 미칠 수 있다.
- 컬럼 단위 접근이 필요하가면, 권한을 허용하고자 하는 컬럼만으로 뷰(VIEW)를 만들어 사용하는 것도 방법이다.

- 각 계정의 부여된 권한이나 역할을 확인하는 법
  - `SHOW GRANTS` 명령어
  - 표 형태로 깔끔하게 보고자 한다면, mysql DB의 권한 관련 테이블을 참조하면 된다.
  - Real MySQL 70p 참고

## 3.5 역할 (Role)

- MySQL 8.0 버전부터 권한을 묶어서 역할(Role)을 사용할 수 있게 되었다.
  - MySQL 서버 내부적으로 역할은 계정과 똑같은 모습을 하고 있다.

```sql
-- 역할 만들기
CRAETE ROLE role_emp_read, role_emp_write;

-- 역할에 권한 부여
GRANT SELECT ON employees.* TO role_emp_read
GRANT INSERT, UPDATE, DELETE ON employees.* TO role_emp_write;

-- 계정 만들기
CREATE USER 'reader'@'127.0.0.1' IDENTIFIED BY 'qwerty'
CREATE USER 'writer'@'127.0.0.1' IDENTIFIED BY 'qwerty'

-- 계정에 역할 부여하기
GRANT role_emp_read TO 'reader'@'127.0.0.1';
GRANT role_emp_read, role_emp_write TO 'writer'@'127.0.0.1';

-- 계정에 역할 활성화하기 (로그아웃하고 로그인하면 다시 활성화 해주어야 한다)
SET ROLE 'role_emp_read';

-- 로그인 시, 자동으로 역할을 활성화하도록 설정 (기본값 OFF)
SET GLOBAL activate_all_roles_in_login=ON;
```

- MySQL 내부적으로 역할과 계정은 동일한 모습을 하고 있다.
  - mysql DB의 user 테이블을 살펴보면 실제 권한과 사용자 계정이 구분없이 저장된 것을 확인할 수 있다.
  - 하나의 계정에 다른 계정의 권한을 병합하기만 하면 되므로, MySQL 서버는 역할과 계정을 구분할 필요 없다.
  - 역할이 위처럼 저장될 때는 host : '%', account_locked : Y 상태로 저장된다.
  - tip) 역할과 계정을 명확히 구분하고자 한다면, DB 관리자가 식별할 수 있는 prefix나 키워드를 추가해 역할을 선택하는 방법을 추천한다.

- `CREATE ROLE` 과 `CREATE USER`을 구분해서 지원하는 이유
  - DB 관리의 직무를 분리할 수 있게 해서, 보안을 강화하는 용도로 사용할 수 있다.
    - ex) `CREATE ROLE`은 가능하지만, `CREATE USER`는 불가능한 사용자
  - `CREATE ROLE`은 account_lock=Y 이므로 로그인 용도로 사용할 수 없다.
- 계정의 기본 역할, 역할 그래프 관계 확인
  - `SHOW GRANTS` 로 볼 수도 있다.
  - mysql.default_roles(계정별 기본 역할), mysql.role_edges(역할에 부여된 역할 관계 그래프) 테이블로 깔끔하게 볼 수 있다.
