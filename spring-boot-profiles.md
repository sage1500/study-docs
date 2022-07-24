Spring Boot Profiles
---

## 1. 参考

- 公式ドキュメント(version 2.7.1)
    - https://spring.pleiades.io/spring-boot/docs/2.7.1/reference/html/features.html#features.external-config

### 1.1. 公式ドキュメントまとめ

#### 1.1.1. プロパティの優先順
プロパティは次の順序で考慮されます（1 から順番に評価され、後の指定により項目単位で上書きされます）。

1. デフォルトのプロパティ（SpringApplication.setDefaultProperties の設定により指定されます）。
1. `@Configuration` クラスの `@PropertySource` アノテーション。
    - ※このようなプロパティソースは、アプリケーションコンテキストがリフレッシュされるまで Environment に追加されないことに注意してください。これは、リフレッシュが始まる前に読み込まれる logging.* や spring.main.* などの特定のプロパティを設定するには遅すぎます。
1. 設定データ（application.properties ファイルなど）。※1 ※2 ※4
    1. jar（application.properties および YAML バリアント）内にパッケージ化されたアプリケーションプロパティ。
    1. jar（application-{profile}.properties および YAML バリアント）内にパッケージ化されたプロファイル固有のアプリケーションプロパティ。
    1. パッケージ化された jar 以外のアプリケーションプロパティ（application.properties および YAML バリアント）。
    1. パッケージ化された jar 以外のプロファイル固有のアプリケーションプロパティ（application-{profile}.properties および YAML バリアント）。
1. random.* のみにプロパティを持つ RandomValuePropertySource。
1. OS 環境変数。
1. Java システムプロパティ（System.getProperties()）。
1. java:comp/env からの JNDI 属性。
1. ServletContext 初期化パラメーター。
1. ServletConfig 初期化パラメーター。
1. SPRING_APPLICATION_JSON のプロパティ（環境変数またはシステムプロパティに埋め込まれたインライン JSON）。
1. コマンドライン引数。
1. テストの properties 属性。@SpringBootTest (Javadoc) およびアプリケーションの特定のスライスをテストするためのテストアノテーションで利用可能。
1. テストに関する @TestPropertySource (Javadoc) アノテーション。
1. devtools がアクティブな場合、$HOME/.config/spring-boot ディレクトリの Devtools グローバル設定プロパティ。

※1：`.properties` 形式と `.yml` 形式の両方の設定ファイルが同じ場所にある場合は、`.properties` が優先されます。

※2：
次の場所から application.properties(※3) ファイルと application.yaml ファイルを自動的に検索してロードします。

1. クラスパスから
    1. クラスパスのルート
    1. クラスパス /config パッケージ
1. カレントディレクトリから
    1. 現在のディレクトリ
    1. 現在のディレクトリの /config サブディレクトリ
    1. /config サブディレクトリの直接の子ディレクトリ

リストは優先順位で並べられます（前の項目よりも低い項目の値が優先されます）。ロードされたファイルのドキュメントは、PropertySources として Spring Environment に追加されます。

`spring.config.location` を使用して設定された場所は、デフォルトの場所を置き換えます。
場所を置き換えるのではなく、場所を追加したい場合は、`spring.config.additional-location` を使用できます。

`spring.config.location` プロパティは、チェックする 1 つ以上の場所の**コンマ区切り**リストを受け入れます。`spring.config.location` に（ファイルではなく）ディレクトリが含まれている場合、それらは / で終わる必要があります。実行時に、ロードされる前に `spring.config.name` から生成された名前が追加されます。`spring.config.location` で指定されたファイルは直接インポートされます。
ロケーショングループは、すべて同じレベルと見なされるロケーションのコレクションです。例: すべてのクラスパスの場所をグループ化し、次にすべての外部の場所をグループ化することができます。ロケーショングループ内のアイテムは **; で区切る**必要があります。

※注意：カンマ区切りとセミコロン区切りで意味が違う。


※3: 設定ファイル名として `application` が気に入らない場合は、`spring.config.name` プロパティを指定して別のファイル名に切り替えることができます。

※4: 「2.3.4. 追加データのインポート」 「2.3.5. 拡張子のないファイルのインポート」※拡張機能→拡張子 「2.3.6. 設定ツリーの使用」
- アプリケーションプロパティは、`spring.config.import` プロパティを使用して、他の場所からさらに設定データをインポートできます。
    - インポートは、検出されたときに処理され、インポートを宣言するドキュメントのすぐ下に挿入された追加のドキュメントとして扱われます。
    - インポートされたプロパティ値は、インポートをトリガーしたファイルよりも優先されます。
- 必要に応じて、プロファイル固有のバリアントもインポートの対象と見なされます。
- 一部のクラウドプラットフォームは、ボリュームにマウントされたファイルにファイル拡張子を追加できません。これらの拡張子のないファイルをインポートするには、Spring Boot にヒントを与えて、ロードする方法を認識させる必要があります。これを行うには、角括弧に拡張ヒントを配置します。
    - 例: `spring.config.import=file:/etc/config/myconfig[.yaml]`
- 多くのクラウドプラットフォームで、設定をマウントされたデータボリュームにマッピングできるようになりました。
    - 複数のファイルがディレクトリツリーに書き込まれ、ファイル名が「キー」になり、内容が「値」になります。
    - `spring.config.import` で `configtree:` プレフィックスを使用して、すべてのファイルをプロパティとして公開する必要があることを Spring Boot が認識できるようにする必要があります。
        - 例: `spring.config.import=optional:configtree:/etc/config/`

※5: 「2.8. 環境に応じて設定を変更する」
- ドキュメントに spring.config.activate.on-profile キーが含まれている場合、プロファイル値（プロファイルのコンマ区切りリストまたはプロファイル式）が Spring Environment.acceptsProfiles() メソッドに入力されます。  
    例）  
    ```properties
    server.port=9000
    #---
    spring.config.activate.on-profile=development
    server.port=9001
    #---
    spring.config.activate.on-profile=production
    server.port=0
    ```

★TODO
- Spring Boot には、さまざまな異なるロケーションアドレスをサポートできるプラグ可能な API が含まれています。デフォルトでは、Java プロパティ、YAML、「設定ツリー」をインポートできます。
- サードパーティの jar は、追加のテクノロジーのサポートを提供できます（ファイルがローカルである必要はありません）。例: Consul、Apache ZooKeeper、NetflixArchaius などの外部ストアからの設定データを想像できます。
- 独自の場所をサポートする場合は、org.springframework.boot.context.config パッケージの ConfigDataLocationResolver クラスと ConfigDataLoader クラスを参照してください。

## 2. コンフィグファイル(application.yml等)の優先度

|ファイル|優先度||
|-|-|-|
|config/application.properties| 1 |
|config/application.yml| 2 |
|application.properties| 3 |
|application.yml| 4 |

## 3. コンフィグファイル(application.yml等)読込の実装確認

### 3.1. コンフィグファイルを Environment Bean に反映する仕組み

EnvironmentPostProcessor の各種実装クラスが、Environment に PropertySource を追加する。

仕組み：
1. ApplicationEnvironmentPreparedEvent イベントが発火
1. EnvironmentPostProcessorApplicationListener がイベントを処理
1. EnvironmentPostProcessor 一覧取得
    - 例：通常のWebMVCアプリケーションの場合
        - イベント一回目
            - RandomValuePropertySourceEnvironmentPostProcessor
            - SystemEnvironmentPropertySourceEnvironmentPostProcessor
            - SpringApplicationJsonEnvironmentPostProcessor
            - CloudFoundryVcapEnvironmentPostProcessor
            - ConfigDataEnvironmentPostProcessor
        - イベント二回目
            - DevToolsHomePropertiesPostProcessor
            - DevToolsPropertyDefaultsPostProcessor
            - DebugAgentEnvironmentPostProcessor
            - IntegrationPropertiesEnvironmentPostProcessor
    - 補足
        - コンフィグファイル(application.yml等)を扱うのは ConfigDataEnvironmentPostProcessor
1. 各 EnvironmentPostProcessor.postProcessEnvironment() を呼び出す。
1. 各 EnvironmentPostProcessor の実装クラスが Environment に PropertySource を追加する。

### 3.2. コンフィグファイル読込クラス
PropertySourceLoader インタフェースの実装を実装したクラス。

|クラス|対応サフィックス|備考|
|-|-|-|
|PropertiesPropertySourceLoader| yml, yaml |
|PropertiesPropertySourceLoader| properties, xml | XML形式のプロパティもサポート？ |

- ※メモ
  - プロパティ形式でも `#---` で区切ることで 1ファイルに複数プロファイルを盛り込める？

### 3.3. memo

- locations:
    - ConfigDataEnvironmentContributor.getImports()
        - ConfigDataProperties.getImports()
    - デフォルトでは以下のとおり。
        - 一つ目の ConfigDataEnvironmentContributor
            - optional:file:./
            - optional:file:./config/
            - optional:file:./config/*/
        - 二つ目の ConfigDataEnvironmentContributor
            - optional:classpath:/
            - optional:classpath:/config/
        - ※上記は公式ドキュメント「2.3. 外部アプリケーションプロパティ」の記載の通りになっている。
    - 以下のプロパティで変更可（TODO 未検証）
        - `spring.config.location` : 変更
        - `spring.config.additional-location` : 追加
- configNames:
    - `spring.config.name` プロパティで変更可能
    - デフォルトは `[ application ]`
        - プロファイル確定後は、`[ application-{プロファイル名} ]` も対象に追加
- extensions:
    - PropertySourceLoader がサポートする全拡張子
    - デフォルトは `[ yml, yaml, properties, xml ]`
- 上記 locations, configNames, extensions の組み合わせが読込対象
    - locations が "/" で終わっている場合は locations + configNames + extensions の組み合わせ
    - locations が "/" で終わっていない場合は locations + extensions の組み合わせ



ConfigDataEnvironmentContributor:
- Kind:
    - ROOT
        - 既存 PropertySource を EXISTING とした children を持つのみ
        - `defaultProperties` という名前の PropertySource は最後に配置
    - INITIAL_IMPORT
        - 以下のプロパティが示すロケーション
            - `spring.config.import`
            - `spring.config.additional-location`
            - `spring.config.location`
	- EXISTING
        - 既存 PropertySource
	- UNBOUND_IMPORT
	- BOUND_IMPORT
	- EMPTY_LOCATION


### 3.4. memo2

- ConfigDataEnvironmentPostProcessor.postProcessEnvironment()
    - ConfigDataEnvironment.processAndApply()
        - processInitial
        - processWithoutProfiles
        - processWithProfiles





1.	ConfigDataEnvironment.processAndApply() 行: 224	
1.	ConfigDataEnvironmentPostProcessor.postProcessEnvironment(ConfigurableEnvironment, ResourceLoader, Collection<String>) 行: 102	
1.	ConfigDataEnvironmentPostProcessor.postProcessEnvironment(ConfigurableEnvironment, SpringApplication) 行: 94	
1.	EnvironmentPostProcessorApplicationListener.onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent) 行: 102	
1.	EnvironmentPostProcessorApplicationListener.onApplicationEvent(ApplicationEvent) 行: 87	
1.	EventPublishingRunListener.environmentPrepared(ConfigurableBootstrapContext, ConfigurableEnvironment) 行: 85	
1.	SpringApplicationRunListeners.lambda$environmentPrepared$2(ConfigurableBootstrapContext, ConfigurableEnvironment, SpringApplicationRunListener) 行: 66	
	139487339.accept(Object) 行: 使用不可	
1.	ArrayList<E>.forEach(Consumer<? super E>) 行: 1540	
1.	SpringApplicationRunListeners.doWithListeners(String, Consumer<SpringApplicationRunListener>, Consumer<StartupStep>) 行: 120	
1.	SpringApplicationRunListeners.doWithListeners(String, Consumer<SpringApplicationRunListener>) 行: 114	
1.	SpringApplicationRunListeners.environmentPrepared(ConfigurableBootstrapContext, ConfigurableEnvironment) 行: 65	
1.	SpringApplication.prepareEnvironment(SpringApplicationRunListeners, DefaultBootstrapContext, ApplicationArguments) 行: 339	
1.	SpringApplication.run(String...) 行: 297	
1.	SpringApplication.run(Class<?>[], String[]) 行: 1312	
1.	SpringApplication.run(Class<?>, String...) 行: 1301	
1.	AdhocWebApplication.main(String[]) 行: 42	
