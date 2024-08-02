# 3장

## 3.1 사용자 식별

- MySQL에서는 계정 권한을 설정하는 방법은 다른 DBMS와 조금 차이가 있음
- 단순 사용자의 아이디 뿐만 아니라 **IP**까지도 확인함
- MySQL 8.0 권한을 묶어서 관리하는 **역할**(Role) 개념도 도입
- 로그인시 `'` 로 식별자를 감싼다. (ex. `'svc_id'@'172.0.0.0'`)
- 모든 외부 IP에서 접속 가능하게 만들고 싶다면 사용자 계정의 호스트 부분을 `%` 대체하면 된다.
- 만약 사용자 계정이 겹친다면 더 좁은 범위의 IP 를 우선 선택한다.
    
    ```java
    'svc_id'@'192.168.0.10' (이 계정의 비밀번호는 123)
    
    'svc_id'@'%'(이 계정의 비밀번호는 abd)
    ```
    
    위의 경우 192.168.0.10 으로 접속한다면 두 경우 모두의 해당된다. 하지만 더 좁은 범위를 우선적으로 선택하기 때문에 위에 있는 계정을 통해 비밀번호를 확인한다.
    
    의도적으로 중첩된 계정을 생성하지는 않겠지만 실수로 이런 상황이 발생할 가능성이 있기 때문에 사용자 계정 생성시 주의해야한다.
    

## 3.2 사용자 계정 관리

### 3.2.1 시스템 계정과 일반 계정

- MySQL 8.0 부터는 계정이 `SYSTEM_USER` 권한을 가지고 있느냐에 따라 ‘시스템 계정(System Account)’ 과 ‘일반 계정(Regular Account)’로 나뉜다.
    - 시스템 계정 : 서버 관리자를 위한 계정, 일반 계정과 같이 사용자를 위한 계정이다.
    - 일반 계정 : 응용 프로그램이나 개발자를 위한 계정
- 시스템 계정은 시스템 계정과 일반 계정을 관리(생성, 삭제, 변경)할 수 있다.
- 시스템 계정이 수행할 수 있는 일
    - 계정 관리
    - 다른 세션 또는 그 세션에서 실행 중인 쿼리를 강제 종료
    - 스토어드 프로그램 생성 시 DEFINER를 타 사용자로 설정 (?)

<aside>
💡 **‘사용자’ 와 ‘계정’ 단어 구분**

사용자 : MySQL 서버를 사용하는 주체(사람 또는 응용 프로그램)

계정 : MySQL 서버에 로그인하기 위한 식별자(로그인 아이디)

</aside>

- MySQL 서버에는 다음과 같이 내장된 계정들이 있다. 내부적으로 각기 다른 목적으로 사용되므로 삭제하지 않도록 주의해야한다.
    - `'mysql.sys'@'localhost'` : MySQL 8.0부터 내장된 sys 스키마의 객체들의 DEFINER로 사용되는 계정
    - `'mysql.session'@'localhost'` : MySQL 플러그인이 서버로 접근할 때 사용되는 계정
    - `'mysql.infoschema'@'localhost'` : information_schema에 정의된 뷰의 DEFINER로 사용되는 계정
- 위 계정들은 처음부터 잠겨 있는 상태이기 때문에 의도적으로 풀지 않는 한 악의적인 용도로 사용할 수 없음

### 3.2.2 계정 생성

- MySQL5.7 버전까지는 GRANT 명령으로 권한 부여와 동시에 계정 생성 가능
- 그러나 8.0 버전부터는 **계정 생성**과 **권한 부여**의 명령어를 구분해서 실행하도록 바뀜
- 계정 생성시 설정할 수 있는 옵션
    - 계정의 인증 방식과 비밀번호
    - 비밀번호 관련 옵션 (유효기간, 이력 개수, 재사용 불가 기간)
    - 기본 역할 (Role)
    - SSL 옵션
    - 계정 잠금 여부

```sql
CREATE USER 'tmpUser'@'%'
    IDENTIFIED WITH 'mysql_native_password' BY 'password'
    REQUIRE NONE
    PASSWORD EXPIRE INTERVAL 30 DAY
    ACCOUNT UNLOCK
    PASSWORD HISTORY DEFAULT
    PASSWORD REUSE INTERVAL DEFAULT
    PASSWORD REQUIRE CURRENT DEFAULT;
```

- `IDENTIFIED WITH`
    - 뒤에는 반드시 인증 방식(인증 플러그인의 이름)을 명시해한다.
    - 기본 인증 방식 사용시 `IDENTIFIED BY 'password'` 형식으로 명시
    - 인증 방식 플러그인은 4가지 방식이 가장 대표적
        - Native Pluggable Authentication : 단순 비밀번호 해시(SHA-1) 값을 저장해두고 일치하는지 확인
        - Caching SHA-2 Pluggable Authentication : Native Pluggable Authentication 와 가장 큰 차이는 SHA-2 알고리즘을 사용한다는 것. 해시 결과값을 캐시해서 이름에 Caching이 포함된 것. 이 방식을 사용하기 위해선 SSL/TSL 또는 RSA 키페어를 반드시 사용해야 하는데, 이를 위해 클라이언트에서 접속 시 SSL 옵션을 활성화해야함
        - PAM Pluggable Authentication : 유닉스나 리눅스 패스워드 또는 LDAP 같은 외부 인증을 사용할 수 있게 해주는 인증 방식, MySQL 엔터프라이즈 에디션에서만 사용 가능
        - LDAP Pluggable Authentication : LDAP을 이용한 외부 인증을 사용할 수 있게 해주는 인증 방식,  MySQL 엔터프라이즈 에디션에서만 사용 가능

MySQL 5.7 버전까지는 Native Authentication이 기본 인증 방식이였음, 8.0 이후부턴 Caching SHA-2 Pluggable Authentication가 기본 인증으로 바뀜. 이는 SSL/TSL 또는 RSA 키페어가 필요하기 때문에 8.0 부터는 다른 방식으로 접속해야함, 그래서 기존 버전과 호환성을 위해 Native Authentication 방식으로 계정을 생성해야 할 수도 있다. 이를 위해 MySQL 설정을 변경하거나 my.cnf 설정 파일에 추가하면 된다.

- `REQUIRE`
    - MySQL 서버에 접속할 때 암호화된 SSL/TLS채널을 사용할지 여부를 선택
    - 별도 설정이 없으면 비암호화 채널로 연결
    - Caching SHA-2 Pluggable Authentication 를 사용하면 암호화된 채널만으로 MySQL 서버에 접속할 수 있게 된다.

- `PASSWORD EXPIRE`
    - 비밀번호 유효 기간 설정 옵션
    - 별도 명시가 없으면 `default_password_lifetime` 시스템 변수 값으로 유효 기간이 설정됨
    - 비밀번호 유효기간 설정이 보안상 안전하지만 응용 프로그램 접속용 계정에 유효 기간을 설정하는 것은 위험할 수 있으니 주의 ?
    - 설정 가능 옵션
        - `PASSWORD EXPIRE` : 계정 생성과 동시에 비밀번호의 만료 처리
        - `PASSWORD EXPIRE NEVER` : 계정 비밀번호의 만료 기간 없음
        - `PASSWORD EXPIRE DEFAULT: default_password_lifetime` : 기본 시스템 변수 값으로 유효기간 설정
        - `PASSWORD EXPIRE INTERVAL n DAY` : 유효기간을 오늘부터 n일자로 설정

- `PASSWORD HISTORY`
    - 한 번 사용했던 비밀번호를 재사용하지 못하게 설정하는 옵션
    - 설정 가능 옵션
        - `PASSWORD HISTORY DEFAULT: password_history` 시스템 변수에 저장된 개수만큼 비밀번호의 이력을 저장, 저장된 이력에 남아있는 비밀번호는 재설정 X
        - `PASSWORD HISTORY n` 비밀번호의 이력을 최근 n까지만 저장
    - 이전 비밀번호를 사용하지 못하게 하려면 이전 비밀번호를 서버가 기억하고 있어야 하는데, 이를 위해 `password_history` 테이블을 사용한다.

- `PASSWORD REUSE INTERVAL`
    - 한 번 사용했던 비밀번호의 재사용 금지 기간을 설정하는 옵션
    - 설정 가능 옵션
        - `PASSWORD REUSE INTERVAL DEFAULT: password_reuse_interval` 변수에 저장된 기간으로 설정
        - `PASSWORD REUSE INTERVAL n DAY` : n일자 이후에 비밀번호를 재사용할 수 있게 설정

- `PASSWORD REQUIRE`
    - 비밀번호가 만료되어 새로운 비밀번호 설정 시, 이전 비밀번호를 필요로 할지 말지를 결정하는 옵션

- `ACCOUNT LOCK / UNLOCK`
    - 계정 생성 / 변경 시 계정을 사용하지 못하게 잠글지 여부를 결정 ?
    

## 3.3 비밀번호 관리

### 3.3.1 고수준 비밀번호

MySQL 에서는 비밀번호 재사용 금지 뿐만 아니라, 쉽게 유추되는 단어를 사용하지 못하게 글자 조합을 강제하거나 금칙어를 설정하는 옵션이 있다. 이를 사용하기 위해선 component_validate_password를 설치해야한다.

<aside>
💡 MySQL 5.7 버전에서는 컴포넌트 형태가 아니라 플러그인 형대로 제공되었기 때문에 명령어의 차이가 있다. 이는 책을 참고하자.

</aside>

```sql
// validate_password 설치
INSTALL COMPONENT 'file://component_validate_password';

// 설치된 컴포넌트 확인
SELECT * FROM mysql.component;

// 컴포넌트의 시스템 변수 확인
SHOW GLOBAL VARIABLES LIKE 'validate_password%';
```

![image](https://github.com/user-attachments/assets/23531eb2-d0ce-4441-9250-1b859286e276)


**[비밀번호 정책]**

- LOW : 비밀번호 길이만 검증
- MEDIUM : 비밀번호 길이 검증 + 숫자와 대소문자 그리고 특수문자와 배합 검증
- STRONG : MEDIUM 검증 + 금칙어가 포함되어 있는지 검증

**[validate_password 시스템 변수]**

- validate_password.length : 검증할 비밀번호 길이
- validate_password.mixed_case_count : 최소 대소문자 포함 길이
- validate_password.number_count : 최소 숫자 포함 길이
- validate_password.special_char_count : 최소 특수문자 포함 길이
- validate_password.dictionary_file : 시스템 변수에 설정된 사전 파일에 명시된 단어를 포함하고 있는지 검증 (1234, qwer 같은 단어를 등록해두면 해당 단어를 포함하고 있으면 비밀번호로 설정할 수 없게 한다)

![image](https://github.com/user-attachments/assets/2ea8bb99-435e-4f18-9ea7-e63c421383cb)


금칙어는 직접 이력해도 되나 인터넷에 검색하면 많이 있기 때문에 다운받아서 사용하는 것을 추천한다.

```sql
SET GLOBAL validate_password.dictionary_file='prohibitive_word.data'
SET GLOBAL validate_password.policy='STRONG';
```

금칙어 파일을 다운받고 위 같은 명령어로 MySQL 서버에 금칙어 파일을 등록하면 된다.

또한 금칙어는 비밀번호 정책이 ‘STRONG’ 이여야 하므로 정책도 바꿔줘야한다.

### 3.3.2 이중 비밀번호

일반적으로 DB는 여러 응용 프로그램에서 동일한 DB 서버를 사용하는 경우가 많았다. 때문에 데이터베이스 서버의 계정 정보는 여러 응용 프로그램에서 공용적으로 사용되는 경우가 많았다. 이러한 특성 때문에 DB 서버 계정의 정보를 쉽게 바꾸기가 어려웠다. 그 중에서 비밀번호 같은 경우에는 서버가 구동중인 경우 변경이 불가능했기 때문에 더더욱 비밀번호를 변경하기 어려웠다. 그래서 처음 설정한 비밀번호를 몇년동안 바꾸지 않고 지속적으로 사용하는 경우가 많았는데 이는 보안상 위험하다.

이를 해결하기 위해 MySQL 8.0 부터는 계정의 비밀번호를 2개의 값을 동시에 사용하는 기능을 제공했다. 이를 MySQL 서버 메뉴얼에서는 ‘이중 비밀번호(Double Password)’라고 소개했다.

<aside>
💡 **주의**

‘이중’ 비밀번호라는 것은 2개 비밀번호 중 하나의 비밀번호만 맞아도 통과하는 것을 말한다. 2개의 비밀번호를 모두 맞춰야 한다는 말이 아니다.

</aside>

이중 비밀번호 기능은 하나의 계정의 2개의 비밀번호를 설정할 수 있는데 프라이머리 비밀번호와 세컨더리 비밀번호로 나뉜다.

- 프라이머리 : 최근에 설정한 비밀번호
- 세컨더리 : 이전에 설정한 비밀번호

이중 비밀번호를 설정하기 위해선 `RETAIN CURRENT PASSWORD` 옵션을 추가하면된다.

```sql
mysal> ALTER USER 'root'@'localhost' IDENTIFIED BY 'old_password';

mysal> ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_password' RETAIN CURRENT PASSWORD;

mysal> ALTER USER 'root'@'localhost' DISCARD OLD PASSWORD; // 세컨더리 비밀번호 삭제
```

## 3.4 권한(Privilege)

MySQL 5.7 이전까지는 권한을 글로벌 권한과 객체 단위 권한으로 구분했다.

- 글로벌 권한 : 데이터베이스나 테이블 이외에 객체에 적용되는 권한
- 객체 단위 권한 : 데이터베이스나 테이블을 제어하는데 필요한 권한

객체 권한은 GRANT 명령어로 권한 부여시 반드시 특정 객체를 명시해야한다. 

반대로 글로벌 권한은 GRANT 명령어에서 특정 객체를 명시하면 안된다.

예외적으로 `ALL` 은 글로벌과 객체 권한 두 가지 용도로 사용할 수 있다. 특정 객체에 ALL 권한이 부여되면 해당 객체에서 적용할 수 있는 모든 객체의 권한을 부여한다. 글로벌에 ALL 권한 부여시, 글로벌 수준에서 부여가능한 모든 권한이 부여된다.

**[정적 권한]**

![image](https://github.com/user-attachments/assets/9282b123-91bd-465c-9b4b-4d473f81b051)


![image](https://github.com/user-attachments/assets/47d0f7f5-47e1-4344-b565-64e4e08af5bf)


**[동적 권한]**

MySQL 8.0 부터는 동적 권한이 추가되었다. 위 표에 있는 권한들은 이전부터 제공되던 정적 권한이라고 한다. 정적 권한은 소스코드에 고정적으로 명시되어 있는 권한을 말한다. 동적 권한은(일부 권한은 MySQL 서버에 명시되어 있기는 하지만) MySQL 서버가 시작하면서 동적으로 생성하는 권한을 말한다. (ex. 컴포넌트 설치 시 그때 부터 등록되는 권한들을 동적 권한이라고 한다.)

![image](https://github.com/user-attachments/assets/4af2ca02-0bda-4bbd-b597-b28701e0dd23)


MySQL 5.7 이전에는 SUPER 라는 권한이 데이터베이스 관리를 위해 꼭 필요한 권한이였는데 이 권한이 8.0 이후로 SUPER 권한이 잘개 쪼개져서 동적 권한으로 분산되었다. 

**[권한 부여]**

```sql
GRANT privilege_list ON db.tavle TO 'user'@'host';
```

- 권한 부여는 위와 같이 GRANT 명령어를 사용한다. 이때 권한의 특성(범위)에 따라 ON 절에 명시되는 오브젝트(DB 나 테이블)이 바뀌어야한다.
- 8.0 버전 이후부턴 존재하지 않는 사용자에 대해 권한 명령어 사용지 에러가 발생한다.
- `GRANT OPTION` 권한은 다른 권한과 달리 GRANT 절 마지막에 ‘WITH GRANT OPTION’ 을 붙여야한다.
- privilege_list 에는 `,` 를 사용해서 명령어를 여러개 명시할 수 있다.

**[글로벌 권한 부여]**

```sql
GRANT SUPER ON *.* TO 'user'@'localhost';
```

글로벌 권한은 특정 오브젝트에 부여할 수 없기 때문에 항상 ON 뒤에는 `*.*` 가 온다.

`*.*` 는 모든 오브젝트를 포함해 모든 MySQL 서버 전체를 말한다.

**[DB 권한 부여]**

```sql
GRANT EVENT ON *.* TO 'user'@'localhost';
GRANT EVENT ON employees.* TO 'user'@'localhost';`
```

DB 권한은 특정 DB에만 부여하거나 모든 DB에 대해 부여할 수 있기 때문에 `employees.*` 를 사용하거나 `*.*` 를 사용한다.

DB 권한 같은 경우 테이블에 대해 부여할 수 없기 때문에 `emplyees.department` 와 같이 명시해줄 순 없다.

**[테이블 권한 부여]**

```sql
GRANT SELECT, INSERT, DELETE, UPDATE ON *.* TO 'user'@'localhost';
GRANT SELECT, INSERT, DELETE, UPDATE ON employees.* TO 'user'@'localhost';
GRANT SELECT, INSERT, DELETE, UPDATE ON employees.department TO 'user'@'localhost';
```

테이블 권한은 위와 같이 특정 테이블 부터 모든 DB에 부여가 가능하다.

**[칼럼 권한 부여]**

테이블에 특정 칼럼에 대해서만도 권한을 부여할 수 있는데 GRANT 명령어 문법이 조금 달라진다. 또한 칼럼에 부여할 수 있는 권한은 DELETE 를 제외한 SELECT, INSERT, UPDATE 를 부여할 수 있다.

```sql
GRANT SELECT, INSERT, UPDATE(dept_name) ON employees.department TO 'user'@'localhost';
```

SELECT, INSERT는 모든 칼럼에 대해 권한을 부여하지만 UPDATE 는 dept_name 칼럼에 대해서만 수행할 수 있게 권한을 부여하려면 위와 같이 명령어를 작성하면 된다.

사실 칼럼 단위 권한 부여는 잘 사용하지 않는다. 칼럼 단위에 권한을 부여하는 순간 모든 테이블의 모든 칼럼에 대해서 권한을 확인하는 작업이 추가되므로 전체적인 성능에 문제를 끼칠 수 있다.

각 계정에 대해 권한을 확인하기 위해 SHOW GRANT 명령어를 사용해도 되지만 표 형태로 깔끔하게 보려면 mysql DB 관련 테이블을 참조하자.

![image](https://github.com/user-attachments/assets/d9dbd535-1631-49b0-9df4-0780a61bcb4d)


## 3.5 역할(Role)

8.0 버전 부터는 권한을 묶어서 역할(Role)이라는 기능을 제공한다.

실제 서버에서 역할은 생성하는 것은 계정을 생성하는 것과 똑같은 모습을 하고 있다.

**[역할 생성]**

```sql
CREATE ROLE 
		role_emp_read,
		role_emp_write;
```

`CREATE ROLE` 작업은 빈 껍데기 역할을 생성하는 명령어이다. 여기에 GRANT 명령어로 권한을 부여해주면 된다.

**[역할 권한 부여]**

```sql
GRANT SELECT ON employees.* TO role_emp_read;
GRANT INSERT, DELETE, UPDATE ON employees.* TO role_emp_write;
```

기본적으로 역할은 그 자체로 사용할 수 없고 계정을 생성해서 역할을 부여해야 사용할 수 있다.

```sql
CREATE USER reader@'127.0.0.1' IDENTIFIED BY '1234';
CREATE USER writer@'127.0.0.1' IDENTIFIED BY '1234';
```

계정을 생성하고

**[역할 부여]**

```sql
GRANT role_emp_read TO reader@'127.0.0.1';
GRANT role_emp_write, role_emp_read TO writer@'127.0.0.1';
```

역할을 부여한다.

SHOW GRANTS 를 하면 권한이 부여된 것을 확인할 수 있다.

~~⁉️ 확인할 수 있다는데 나는 다르게 나온다..~~ → 권한 부여시 employees.*  라고 했는데 내가 사용하는 DB 가 employees가 아니여서 권한이 부여가 안됐었다.

```sql
SELECT * FROM finance_db.company;
```

그런데 실제로 조회 작업을 해보면 권한이 없다는 에러가 발생한다.

**[역할 확인]**

```sql
SELECT current_role();
```

실제로 권한을 확인해봐도 NONE 이라고 나온다. 이는 역할이 아직 활성화 되어있지 않기 때문이다. 따라서 역할을 활성화 해줘야한다.

**[계정 역할 활성화]**

```sql
// 역할 활성화
SET role_emp_read;

SELECT * FROM finance_db.company;
```

역할을 활성화한 다음에 다시 쿼리를 실행하면 잘 보인다.

이러한 역할은 계정을 로그아웃 했다가 다시 로그인하면 역할이 활성화되지 않는 상태로 초기화되어버린다.

매번 역할을 활성하는 것이 불편해보이는데 이는 MySQL 서버에서 역할을 자동으로 활성화하는 옵션이 꺼져있기 때문이다. 이러한 옵션은 `activate_all_roles_on_login` 시스템 변수로 설정할 수 있다. 해당 값을 ON 으로 설정하게 되면 매번 로그인시마다 역할을 활성화 해주지 않아도 된다.

**[항상 계정 역할 활성화]**

```sql
SET GLOBAL activate_all_roles_on_login=ON;
```

**[역할과 계정의 차이점]**

이제 역할의 비밀에 대해서 더 파해쳐보자. MySQL 서버 내부적으로 **역할**은 **계정**과 동일한 객체로 취급된다. 단지 하나의 계정에 다른 권한을 가지고 있는 계정을 병합해서 제어가 가능해졌을 뿐이다. 실제로 mysql user 테이블을 살펴보면 권한이 같이 관리되는 것을 볼 수 있다.

```sql
SELECT user, host, account_locked FROM mysql.user;
```

<img width="606" alt="image" src="https://github.com/user-attachments/assets/fb737e80-5c27-4847-a313-9f66f694a31a">


역할과 계정은 account_locked 값만 다를 뿐 아무런 차이가 없다. 역할과 계정을 구분하는 칼럼이 따로 있는 것도 아니다. 그렇다면 어떻게 역할과 계정을 구분할 수 있을까, 하는 의문이 생길 수 있는데, 하나의 계정에 다른 계정의 권한을 병합하면 되기 때문에 역할과 계정을 따로 구분할 필요가 없다.⁉️ (납득이 안됨)

일반적으로 CREATE USER 명령어는 계정을 생성할 때 `user@'127.0.0.1'` 와 같이 계정 이름 뒤에 호스트 부분을 함께 명시한다. 하지만 역할 생성시에는 `CREATE ROLE writer` 와 같이 뒤에 호스트를 따로 명시하지 않았다. 이러한 차이가 계정과 역할을 구분한다고 생각할 수 있다. 하지만 뒤에 호스트를 따로 명시하지 않으면 자동으로 `%` 가 호스트로 들어간다. 실제로 이전에 user 테이블을 확인했을 때 역할의 경우 호스트 칼럼에 `%` 가 들어간 것을 볼 수 있다. 따라서 `reader` 라고 계정 이름만 명시해도 `reader@'%'` 와 동일한 계정을 생성한다.

```sql
CREATE ROLE 
		role_emp_read,
		role_emp_write;

CREATE ROLE
		role_emp_reader@'%',
    role_emp_writer@'%';
```
<img width="575" alt="image" src="https://github.com/user-attachments/assets/ba700d58-ed9d-483d-9ffa-7f753cecdaca">

<aside>
💡 계정과 역할은 MySQL 서버 내부적으로 아무런 차이가 없다. 실제 관리자나 사용자 입장에서 확인했을 때도 역할과 계정을 구분하기가 어렵다. 그래서 책에 예제에서는 `role_` 이라는 Prefix 를 사용하여 계정과 역할을 구분해주었다. 물론 역할로 사용되다가 계정으로 사용될 수도 있지만 역할과 계정을 구분하고자 한다면 Prefix 나 키워드를 추가하여 구분하는 방식을 권장한다.

</aside>

**[역할 호스트, 계정 호스트가 다르다면?]**

```sql
// 계정, 역할 생성
CREATE ROLE role_emp_local_read@'localhost';
CREATE USER reader@'127.0.0.1';

// 역할에 권한 부여
GRANT SELECT ON finance_db.* TO role_emp_local_read@'localhost';

// 계정에 역할 부여
GRANT role_emp_local_read@'localhost' TO reader@'127.0.0.1';
```

역할의 호스트와 계정의 호스트가 다른 경우 어떤 일이 벌어지는지 확인하는 예제이다. 위 쿼리를 날려보면 아무런 문제 없이 reader에게 role_emp_local_read 권한을 부여할 수 있다. 즉, 역할의 호스트는 아무런 영향이 없다. 만약 역할을 계정처럼 사용한다면 그때는 호스트 부분이 중요해진다.

**[CREATE USER, CREATE ROLE 명령어가 구분되는 이유]**

역할과 계정이 내부적으로 동일한 객체라고 했는데 왜 굳이 CREATE USER 과 CREATE ROLE 명령어를 구분했을까? 이는 데이터베이스 관리 직무를 분리하여 보안성을 높일 수 있기 때문이다. CREATE USER 권한은 없지만 CREATE ROLE 명령어만 실행할 수 있는 역할을 만들 수 있다. 이렇게 생성된 역할은 계정과 동일한 객체로 만들어지지만, `account_locked` 값이 ‘Y’ 이기 때문에 로그인 용도로는 사용할 수 없다.

계정의 기본 역할 또는 역할에 부여된 역할 그래프 관계를 SHOW GRANTS 로 확인할 수 있지만, 표 형태로 깔끔하게 보고 싶다면 mysql DB 관련 테이블을 참고하자.
<img width="876" alt="image" src="https://github.com/user-attachments/assets/1d1e3a05-1954-4abc-962b-d33cf4ec672c">
```sql
SELECT * FROM mysql.default_roles;
SELECT * FROM mysql.role_edges;
```
