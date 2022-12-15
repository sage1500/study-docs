# Tymeleaf with Spring

## 参考
- https://www.thymeleaf.org/
  - 3.1ドキュメント
    - https://www.thymeleaf.org/doc/tutorials/3.1/usingthymeleaf.html
  - 3.0ドキュメント日本語訳
    - https://www.thymeleaf.org/doc/tutorials/3.0/usingthymeleaf_ja.html
  - Tymeleaf+Spring 3.1ドキュメント
    - https://www.thymeleaf.org/doc/tutorials/3.1/thymeleafspring.html  


## 関連クラス

- StandardDialect
  - StandardExpressionObjectFactory : 式オブジェクトのファクトリクラス
- SpringStandardDialect
  - SpringStandardExpressionObjectFactory : 式オブジェクトのファクトリクラス

## 式オブジェクト

|定義元|Context依存|式オブジェクト名|クラス|備考
|-|-|-|-|-
|Thymeleaf|○|object|IExpressionContext
|Thymeleaf|○|root|IExpressionContext
|Thymeleaf|○|vars|IExpressionContext
|Thymeleaf|○|ctx|IExpressionContext
|Thymeleaf|○|locale|Locale
|Thymeleaf|○|request|HttpServletRequest
|Thymeleaf|○|response|HttpServletResponse
|Thymeleaf|○|session|HttpSession
|Thymeleaf|○|servletContext|ServletContext
|Thymeleaf|○|~~httpServletRequest~~|HttpServletRequest|deprecated
|Thymeleaf|○|~~httpSession~~|HttpSession|deprecated
|Thymeleaf|○|conversions|Conversions
|Thymeleaf|-|uris|Uris
|Thymeleaf|○|calendars|Calendars|現在のLocaleに依存
|Thymeleaf|○|dates|Dates|現在のLocaleに依存
|Thymeleaf|-|bools|Bools
|Thymeleaf|○|numbers|Numbers|現在のLocaleに依存
|Thymeleaf|-|objects|Objects
|Thymeleaf|o|strings|Strings|現在のLocaleに依存
|Thymeleaf|-|arrays|Arrays
|Thymeleaf|-|lists|Lists
|Thymeleaf|-|sets|Sets
|Thymeleaf|-|maps|Maps
|Thymeleaf|-|aggregates|Aggregates
|Thymeleaf|o|messages|Messages
|Thymeleaf|o|ids|Ids
|Thymeleaf|o|execInfo|ExecutionInfo
|Tymeleaf-Spring|o|fields|Fields
|Tymeleaf-Spring|o|themes|Themes
|Tymeleaf-Spring|-|mvc|Mvc
|Tymeleaf-Spring|o|requestdatavalues|RequestDataValues

