Database
---

ネタ
- ER図作成
    - Eclipse
        - AmaterasERD
            - ※Peiades All In One Eclipseに含まれている
        - ERMaster
            - ※追加インストールが必要

# AmaterasERD
おそらく ERMaster のインスパイア元。

# ERMaster

http://ermaster.sourceforge.net/index_ja.html


## 外部キー制約
DDL生成時にオプションとして外部キーの生成するコードをDDLから除外することができる。
そのため、性能が気になるなどの事情で外部キー制約をつけたくない場合でも、
ER図上は、外部キー制約ありで設計するとよい。


# MyBatis Generator

https://mybatis.org/generator/index.html

maven用のプラグインがあるので、そちらを利用する。

## Maven
コード生成のコマンド

```
$ mvn mybatis-generator:generate
```

## schema.sql
H2 でのUT用に schema.sql が用意している場合は、
`mybatis.generator.sqlScript` プロパティに、このファイルを指定することで、
MyBatisGenerator にスキーマを構築させることができる。
ただし、`mybatis.generator.sqlScript` で使用するDDL に `--` を使ったコメントがあるとダメっぽい。
そのため、ERMaster で DDLを生成する際には、「CREATE文にコメントを付ける」「CREATE文のカラムにコメントを付ける」のチェックを外した状態で schema.sql を作成する必要がある。
このチェックがあるご、ERMaster は `--` を使ったコメントをDDL中に出力する。

## pom.xml

```xml
	<properties>
		<!-- ... -->
		<mybatis.generator.configurationFile>generatorConfig.xml</mybatis.generator.configurationFile>
		<mybatis.generator.jdbcDriver>org.h2.Driver</mybatis.generator.jdbcDriver>
		<mybatis.generator.jdbcURL>jdbc:h2:mem:temp;DB_CLOSE_DELAY=-1</mybatis.generator.jdbcURL>
		<mybatis.generator.jdbcUserId></mybatis.generator.jdbcUserId>
		<mybatis.generator.jdbcPassword></mybatis.generator.jdbcPassword>
		<mybatis.generator.overwrite>true</mybatis.generator.overwrite>
		<mybatis.generator.sqlScript>classpath:schema.sql</mybatis.generator.sqlScript>
		<!-- ... -->
	</properties>
```
- memo
    - 上記には記載していないが、`mybatis.generator.includeAllDependencies` プロパティを `true` に設定すると、
        pom.xml の dependency を plugin の dependency として含めることができる。plugin の dependency に JDBCドライバの dependency を書きたくない場合に使えなくもない。
        ```xml
        <mybatis.generator.includeAllDependencies>true</mybatis.generator.includeAllDependencies>
        ```
    - plugin の `<configiration>` で指定するパラメータと
        `<properties>` で指定する `mybatis.generator.xxx` のプロパティの両方が定義されていた場合、
        `<configiration>` の方が優先されるようだ。

```xml
	<build>
		<!-- ... -->
		<plugins>
    		<!-- ... -->
			<plugin>
				<groupId>org.mybatis.generator</groupId>
				<artifactId>mybatis-generator-maven-plugin</artifactId>
				<version>1.4.1</version>
				<dependencies>
                    <!-- JDBCドライバ -->
					<dependency>
						<groupId>com.h2database</groupId>
						<artifactId>h2</artifactId>
						<version>${h2.version}</version>
					</dependency>
                    <!-- MBG Plugin -->
                    <dependency>
                        <groupId>${project.groupId}</groupId>
                        <artifactId>demo-mbg-plugins</artifactId>
                        <version>${project.version}</version>
                    </dependency>
				</dependencies>
			</plugin>
    		<!-- ... -->
		</plugins>
	</build>
```
- point
    - plugin の dependency には、JDBCドライバが必要。
        もし、独自の MyBatisGeneratorPluginを利用する場合は、独自の MyBatisGeneratorPlugin も plugin の dependency に追記する必要あり。
    - spring-boot-starter-parent を親pomにしている場合は、`h2.version` プロパティが定義されているため、
        これを version に設定でる。
        なお、dependencyManagement で h2 のバージョンが設定されているが、この設定は、plugin の dependency には影響しないため、ここのでの versionの設定が必要になる。

## generatorConfig.xml
```xml
<!DOCTYPE generatorConfiguration PUBLIC
 "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
 "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>
	<context id="simple" targetRuntime="MyBatis3">

		<plugin type="org.mybatis.generator.plugins.SerializablePlugin" />
		<plugin type="org.mybatis.generator.plugins.ToStringPlugin" />
		<plugin type="org.mybatis.generator.plugins.EqualsHashCodePlugin" />
		<plugin type="org.mybatis.generator.plugins.FluentBuilderMethodsPlugin" />
		<plugin type="org.mybatis.generator.plugins.MapperAnnotationPlugin" />
		<!-- <plugin type="org.mybatis.generator.plugins.RowBoundsPlugin" /> -->
		<plugin type="com.example.demo.mbg.plugins.OffsetLimitPlugin" />

		<jdbcConnection
			driverClass="${mybatis.generator.jdbcDriver}"
			connectionURL="${mybatis.generator.jdbcURL}"
			userId="${mybatis.generator.jdbcUserId}"
			password="${mybatis.generator.jdbcPassword}" />

		<javaModelGenerator
			targetPackage="com.example.demo.domain.entity"
			targetProject="src/main/java" />

		<sqlMapGenerator
			targetPackage="com.example.demo.domain.simple.repository"
			targetProject="src/main/resources" />

		<javaClientGenerator type="XMLMAPPER"
			targetPackage="com.example.demo.domain.simple.repository"
			targetProject="src/main/java" />

		<table tableName="t_order" domainObjectName="Order" mapperName="OrderRepository">
			<generatedKey column="id" sqlStatement="JDBC"/>
		</table>
		<table tableName="t_order_item" domainObjectName="OrderItem" mapperName="OrderItemRepository" />
		<table tableName="t_item" domainObjectName="Item" mapperName="ItemRepository" />
	</context>
</generatorConfiguration>
```
- point
    - plugin は適宜設定する。
    - RowBoundsPlugin は性能がでない、および、PostgreSQL の場合は、OutOfMemoryError の例外がスローされる可能性があるため、`offset` と `limit` を使ったページネーション用の SQL を出力するように Plugin を自作するとよい。
        - 参考) https://jpdebug.com/p/1968350
            - なお、プラグイン作成時に使用する MyBatisGenerator のバージョンは、実際に使用する MyBatisGenerator のバージョンを合わせること。参考ページの MyBatisGenerator のバージョンは少し古いため、ソースコードの若干の修正が必要になる可能性あり。
    - `targetProject` を `src/main/java`、`src/main/resources` にしているため、`mvn mybatis-generator:generate` コマンドを投入する際には、カレントディレクトリが pom.xml と同じディレクトリにあることを前提としている。
- memo
    - generatorConfig.xml の targetProject を MAVEN にすると、`target\generated-sources\mybatis-generator` にソースが出力される。そのままビルドして jar を Mavenリポジトリにデプロイする場合は使える。
        生成したコースコードを git にコミットする場合には使えない。


# PostgreSQL と H2 Database の両方で使用できる SQL にする

- ID値生成
    - DDL編
        - 参考) https://vividcode.hatenablog.com/entry/sql/sequence-postgres-and-h2
        - ERMaster でシーケンスを作成する。※シーケンス命名即の例： `<テーブル名>_<カラム名>_seq`
        - ERMaster でカラムのデフォルト値の部分に `nextval('<作成したシーケンス名>')` を設定する。
    - generatorConfig.xml
        - `<table>` の子要素の　`<generatedKey>` を設定する。
            ```xml
            <table tableName="t_order" domainObjectName="Order" mapperName="OrderRepository">
                <generatedKey column="id" sqlStatement="JDBC"/>
            </table>
            ```

# H2 Database

## Spring Boot + H2 Console

H2 Console は SQL Developper 相当のツール。

Spring Boot で Spring MVC + H2 + DevTools の環境で開発している場合、
デフォルトで H2 Console が有効になっている（はず）。

　参考) https://www.lifestyle12345.com/2019/04/h2-console-spring-boot.html

そのため、In memory DB で動かしている場合でも
DBのデータの内容確認や情報更新等が可能。

Spring Boot の場合、`spring.h2.console.xxx` プロパティで各種設定変更可能。
基本デフォルトでOK。

H2 Console はデフォルトでは、 `/h2-console` にアクセスすることで表示できる。
以下を入力して、Connect ボタンを押下する。

- JDBC URL : `spring.datasource.url` プロパティと同じ値
- User Name : `spring.datasource.username` プロパティと同じ値。設定しなかった場合は sa
- Password : `spring.datasource.password`

参考) https://it-jog.com/java/springdatajpa/h2console
