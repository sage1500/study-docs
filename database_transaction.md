# 1. Transaction

## 1.1. Connection, Transaction, Session

- Connection: java.sql パッケージで定義されている Java標準インタフェース。
- Transaction: トランザクションを表現するクラス。非Java標準。Spring Framework や、MyBatis 等で独自に定義される。
- Session: データベース非標準。MyBatis で独自に定義される。

## 1.2. MyBatis と Spring Framework(Spring JDBC) の Transaction の連携

org.mybatis:mybatis-springを使う(Spring Boot の場合、mybatis-spring-boot-starter を依存関係に含めるだけでよい) と、
MyBatis の Transaction が Spring Framework の Transaction を使用する実装に変わる。

参照) org.mybatis.spring.transaction.SpringManagedTransaction

MyBatis の Transaction から Spring Framework の Transaction への参照は
DataSouce オブジェクトをキーとして連携する。
そのため、MyBatis と Spring Framework で同じ DataSource を使用しないと MyBatis と Spring Framework の Transaction が連携しないので注意が必要。
通常、DataSource は一つしか利用しないため、あまり気にしなくてよいが、
複数の DataSource を扱う場合には、気にする必要がある。

具体的には、MyBatisの Mapperインタフェースは、DataSource に紐づいているため、
Spring Framework の `@Transactional` アノテーションでは、これと同じ DataSOurce に紐づいている
TransactionManager の Bean名を指定する必要がある。

- MyBatis: Mapperインタフェース と DataSource の連携
    - Mapperインタフェース -> `@MapperScan(sqlSessionFactoryRef = "SqlSessionFactoryのBean名")` -> SqlSessionFactory -> DataSource 
- Spring Framework: `@Transactional` と DataSource の連携
    - `@Transactional(transactionManager = "TransactionManagerのBean名")` -> TransactionManager -> DataSource 




