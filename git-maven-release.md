Git & Maven Relase
---

Git でソース管理している状態での
Maven Release Plugin を使用したリリース方法を考える。


## GitFlow でのリリース方法（案）

手順概略

1. developブランチから releaseブランチを作成する。
2. releaseブランチで mvn release:prepare を実行する。
	1. Releaseバージョンにした状態で git に push され、タグも付与される
	2. バージョンを上げたSnapshotバージョンに戻した状態で git に push される
3. (必要であれば)上記に引き続き mvn release:perform を実行する。
	1. タグからソース取り出し、ビルドし、Mavenリポジトリに deploy する。
4. タグから mainブランチにマージ（プルリクエスト）する。
5. releaseブランチから developブランチにマージ（プルリクエスト）する。
6. (必要であれば)mainブランチにタグを付与する
7. 故障改修時は、releaseブランチに対して更新する。
	1. ※実際には workブランチ作成→修正→releaseブランチにマージという流れ
8. 故障改修後に再リリースは、上記2 からやり直す。


## Maven Release Plugin

本家: https://maven.apache.org/maven-release/maven-release-plugin/index.html


### リモートリポジトリ

`mvn release:prepare`コマンドはタグを push しようとするため、リモートリポジトリが必要。
なお、リモートリポジトリは、github や gitlab の他、`git init --bare` でローカルPC上に作成したリポジトリでも可。


### pom.xml

必須：
- scm/developerConnection要素に gitリポジトリを設定
	- `git init --bare` でローカルPC上に作成したリポジトリを指定してもよい。
- build/plugins に maven-release-plugin を定義する。

オプション：
- scm/tag要素はあらかじめ設定しておいた方がよい。
	- Maven Repository Plugin が追記するため、あらかじめ書いていないと出力位置や、インデントが期待したものと異なる可能性がある。


設定イメージ（一部抜粋）：
```xml
<scm>
	<connection>scm:git:file://C:/git-repos/test/</connection>
	<developerConnection>scm:git:file://C:/git-repos/test/</developerConnection>
	<tag>HEAD</tag>
	<url>http://dummy/</url>
</scm>

<build>
	<plugins>
		<!-- ...その他プラグイン... -->
		
		<!-- release -->
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-release-plugin</artifactId>
			<version>2.5.3</version>
			<configuration>
				<tagNameFormat>v@{project.version}</tagNameFormat>
			</configuration>
		</plugin>
	</plugins>
</build>
```


### .gitignore

`.gitignore` に Maven Release Plugins が出力する作業ファイルを追加する。  
※gitリポジトリへの push が必要な運用する場合は追加しない。

```
### Maven Release Plugins ###
pom.xml.releaseBackup
pom.xml.backup
pom.xml.tag
pom.xml.next
pom.xml.branch
release.properties
```


### コマンドメモ

#### mvn release:clean

mvn release:prepareで更新した内容を基に戻す。  

※gitリポジトリ上は元に戻さない。
※pom.xml は pom.xml.releaseBackup から戻すため、これがないと元に戻せない。


#### mvn release:parepare

リリース時の Git上のソース管理という意味では、このコマンドが主役。

1. pom.xml を SNAPSHOTバージョンからRELEASEバージョンに書き換える。
	- ※SNAPSHOTバージョンが付いていないと pom.xml だとエラーになる。
2. pom.xml の scm/tag をRELEASEタグに書き換える。
3. [git] commit する
4. [git] RELEASEタグを付ける。
5. [git] gitリポジトリに push する。
6. pom.xml を RELEASEバージョンからバージョンを上げたSNAPSHOTバージョンに書き換える。
7. pom.xml の scm/tag を HEAD に書き換える。
8. [git] gitリポジトリに push する。


★TODO testをスキップする方法
  そもそもデフォルトでは、 mvn clean verify して、その結果テストが動いている。  
  ※verity は package と install の間のフェーズ  
  この部分は Plugin の以下の設定で変更可能（らしい）
```xml
<configuration>
	<preparationGoals>clean verify</preparationGoals>
</configuration>
```
ここを clean だけにすれば、とりあえずバージョンを変えた pom.xml を git に pushするという目的は果たせる。


- DryRun（必要に応じて実行）
	- `mvn release:clean release:prepare --batch-mode -DdryRun=true`
- リリースバージョンを指定して実行する(releaseブランチ作成直後を想定)
	- Release時のバージョンは指定したバージョン
	- タグ:v{{Release時のバージョン}}
	- Snapshotバージョンは、Releaseバージョン x.y.z の zの値をインクリメントしたもの
	- `mvn release:clean release:prepare --batch-mode -DreleaseVersion=1.0.0`
- バージョンとタグをお任せで実行する(releaseブランチでの故障改修を想定)
	- Release時のバージョンは SNAPSHOTを外したバージョン
	- タグ:v{{Release時のバージョン}}
	- Snapshotバージョンは、バージョン x.y.z の zの値をインクリメントしたもの
	- `mvn release:clean release:prepare --batch-mode`


※メモ：push するタイミングとか気に入らない場合
```
mvn release:clean release:prepare -DdryRun=true

は以下の中間ファイルを作成されるが、git の操作(add, commit, push)はしない。

	pom.xml.relaseBackup
	pom.xml.tag
	pom.xml.next

そのため、上記コマンド実行後に pom.xml を更新したいタイミングで

	find . -name "pom.xml.tag" -execdir cp '{}' pom.xml \;
	や
	find . -name "pom.xml.next" -execdir cp '{}' pom.xml \;

を実行するという手もある(かもしれない)。
```


### TODO 以下書きかけ（消すかも）

$ mvn release:perform

	タグのソースをとってきて mvn deploy する
		<build><defaultGoal>を見ているわけではなさそう

release.properties を参考にする。
なければ、以下のような感じで実行することもできる。
mvn org.apache.maven.plugins:maven-release-plugin:3.0.0-M5:perform -DconnectionUrl=scm:svn:https://svn.mycompany.com/repos/path/to/myproject/tags/myproject-1.2.3


	<releaseProfiles>release</releaseProfiles>
	と定義して、release プロファイルの defaultGoal を install にしても、mvn deploy される。なぜだ？




mvn release:branch -DbranchName=release/0.0.7 -DupdateBranchVersions=true -DupdateWorkingCopyVersions=false
	SNAPSHOTバージョンを上げたものをブランチに作成
	元のブランチは、merge --ff して、SUNAPSHOTバージョンを戻すという不思議な動き

	release:prepare の タグの代わりにブランチを使う版
	


mvn --batch-mode release:branch -DbranchName=my-branch-1.2 -Dproject.rel.org.myCompany:projectA=1.2 \
     -Dproject.dev.org.myCompany:projectA=2.0-SNAPSHOT
     
mvn --batch-mode -Dtag=my-proj-1.2 release:prepare \
                 -DreleaseVersion=1.2 \
                 -DdevelopmentVersion=2.0-SNAPSHOT
mvn --batch-mode -Dtag=my-proj-1.2 -Dproject.rel.org.myCompany:projectA=1.2 \
     -Dproject.dev.org.myCompany:projectA=1.3-SNAPSHOT release:prepare


release:update-versions
	mvn --batch-mode release:update-versions
		SNAPSHOTバージョンのままバージョンを上げる

	mvn --batch-mode release:update-versions -DdevelopmentVersion=1.2.0-SNAPSHOT
		バージョン指定もできるが、
		SNAPSHOTなしのバージョンにはできない。

