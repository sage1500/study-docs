Git
---

## 1. チュートリアル

### 1.1. ローカル操作

#### 1.1.1. ローカルリポジトリを作成する。

```
$ git init
$ git branch -m main
$ git status
```

#### 1.1.2. ファイルの修正をローカルリポジトリにコミットする。

ファイルの修正（追加、変更、削除）をローカルリポジトリにコミットするには、
対象のファイルを `git add` コマンドで登録し、`git commit` コマンドで一括でローカルリポジトリにコミットします。

```
$ echo "first" > file1.txt
$ ls
$ cat file1.txt
$ git status
$ git add .
$ git status
$ git commit -m "first commit on main"
$ git status
$ git log --oneline
```

※本チュートリアルでは分かりやすくするために、コミットメッセージの最後に「`on コミットしたブランチ名`」をつけるようにしています。


#### 1.1.3. 作業ブランチ作成と切替

ブランチの作成と切替のコマンドはそれぞれ `git branch`、`git checkout` です。
また、ブランチの作成と切替を一度に実施するコマンドとして `git checkout -b` があります。

```
$ git checkout -b feature/1
```

#### 1.1.4. 作業ブランチにコミットする。

前述のとおり、
ファイルの修正（追加、変更、削除）をローカルリポジトリにコミットするには、
`git add` および、`git commit` コマンドを使用します。

```
$ echo "second" >> file1.txt
$ cat file1.txt
$ git diff
$ git status
$ git add .
$ git status
$ git diff --staged
$ git commit -m "change on feature/1"
$ git status
$ git log --oneline
```

これから `git add` しようとするファイルの差分は `git diff` コマンド、
これから `git commit` しようとするファイルの差分は `git diff --staged` コマンドで表示できます。

#### 1.1.5. 作業ブランチをブランチ元にマージする。

```
$ git status
$ git checkout main
$ git log --oneline --graph
$ git log --oneline --graph feature/1
$ git diff HEAD..feature/1
$ git merge feature/1 --no-ff
$ cat file1.txt
$ git log --oneline --graph
$ git branch -d feature/1
$ git branch
```

★TODO コンフリクトした場合は？

#### 1.1.6. ブランチ元の修正を作業ブランチに取り込む。

準備1
- 作業ブランチを作成してから、ブランチ元でコミットを2つ作成する。
	- ここでコミットを2つ作成しているのは、一部のコミットを作業ブランチのコミットとコンフリクトするとどうなるか確認したいため。
```
$ git checkout main
$ git branch feature/2
$ echo "new-file" > file2.txt
$ git add .
$ git commit -m "add file2.txt on main"
$ echo "3rd" >> file1.txt
$ git add .
$ git commit -m "change file1.txt on main"
$ git log --oneline
```

準備2
- 作業ブランチに切替え、タグを作成する。
	- タグを作成しているのは、まずコンフリクトしないケースでの作業ブランチへの取り込み動作を確認し、その後、一旦元の状態の戻してからコンフリクトするケースでの作業ブランチの取り込み動作を確認するため。
```
$ git checkout feature/2
$ git log --oneline
$ git diff HEAD..main
$ git tag tag1
$ git log --oneline
```

ブランチ元の修正取り込みは `git rebase` コマンドで実施するのが無難。
まずは、コンフリクトしないケース。

```
$ git diff HEAD..main
$ git rebase main
$ git log --oneline
$ ls
$ cat file1.txt
$ cat file2.txt
```

一旦、rebase 前に戻す。

```
$ git reset --hard tag1
$ git log --oneline
$ git status
$ ls
$ cat file1.txt
```

コンフリクトするようなコミットを作成。

```
$ echo "change on feature/2" >> file1.txt
$ git status
$ git add .
$ git commit -m "change file1.txt on feature/2"
```

ブランチ元の修正取り込みを `git rebase` コマンドで再度実施。

```
$ git tag tag2
$ git log --oneline
$ git diff HEAD..main
$ git rebase main
$ git status
$ git diff
$ vim file1.txt
$ git status
$ git diff
$ git add file1.txt
$ git rebase --continue
$ git status
$ git log --oneline
$ git diff tag2
$ git tag -d tag2
```

コンフリクトしたコミットの数だけ、`git add`、`git rebase --continue` を繰り返す。
この時、`git add` の後に `git commit` は不要。ただ、`git commit` してしまっても問題はない。

rebase後の差分確認や、rebase前に戻せるように rebase 前に tag を作成しておくのが無難。


## 2. ローカル操作

### 2.1. ローカルリポジトリを作成する。

```
$ git init
$ git branch -m main
```


### TODO
git status
git add {{file}}
git add . -n
git add . 
git add . -v
git commit 

git restore --staged {{file}}
git restore --staged .

git log --oneline

git checkout -b {{branch-name}}
git checkout {{branch-name}}

git diff {{branch-name}}..{{branch-name}}

git diff {{branch-name}}

git diff
	# add していないファイルとの差分
git diff --staged
	# staged と最新commitとの差分

	
git merge {{branch-name}}
..編集..
git add
git commit
  -> この場合、コミットコメントがデフォルトで入っている。そのまま採用してもよいかも

TODO 以下の意味は？

```
# If this is not correct, please run
#       git update-ref -d MERGE_HEAD
# and try again.
```

git branch -d {{branch-name}}

git log --oneline --graph


## ブランチ元の変更を取り込む
git diff {{branch-name}}
git merge {{branch-name}}

	※ --no-ff だとどうなる？
	このケースではあまりうれしくなさそう

	git merge --squash だと、修正内容だけ取り込む
		→手動マージと同じ動き？
			→mergeした記録は残る？残らないなら複数実行すると？
				→commit はされないが、add はされている
				→記録は残っていて複数実行するとエラーになる
		→ブランチ元のコミットを取り込んでいないことになるので
		　ブランチ元にマージしなおす際にコンフリクトする？ TODO

git rebase {{branch-name}}
	# コンフリクトしていない自分のコミットはそのまま取り込まれる
--ここから
編集
git add .
git commit ## TODO この commit は不要
	-> この場合、過去コミットコメントがデフォルトで入っている。
git rebase --continue
--ここまでをコンフリクトしているコミットの数だけ繰り返す

※rebase をやめたい場合は
	git rebase --abort



TODO ブランチ元の変更（コミット）をすべて取り込むなら --ff がよさげ。

git commit --amend
	直前（現在）のコミットのコミットコメントを変更する

### コミットをまとめる

$ git rebase -i HEAD~{{n}}  # n はまとめる数(以上)を設定
・[EDIT]まとめるコミットのpick を s に変更して保存
・[EDIT]まとめた後のコメントを編集して保存

### TODO コミットを取りやめる

※ git rebase -i HEAD~{{n}} してから、pick を d に変えて保存するという方法もある


### TODO 過去のコミットを変更する

★ git rebase -i HEAD~{{n}} してから、pick を edit に変えて保存する-> git add -> git rebase --continue ※git commit は不要

### TODO 過去のコミットコメントを変更する

★ git rebase -i HEAD~{{n}} してから、pick を reward に変えて保存する


### 情報まとめ中
- マージ
	- $ git merge {{branch-name}}
		- --ff
		- --no-ff
		- --squash ## 統合ブランチへマージ時にコミットをまとめる。。。？
		
- コミット取消 TODO
	- $ git reset --hard HEAD~

- 先頭のコミットを修正する
	- ファイルの内容とコミットコメントを修正する
		$ git add .
		$ git commit --amend
	- コミットコメントのみ修正する
		$ git commit --amend
- ccc TODO
	- コミット内容を取り消すコミットを新しく作る
		- コミットは履歴に残ったまま。
		- $ git revert {{commit}}
		- コミットメッセージを編集しない
			- $ git revert {{commit}} --no-edit
		- コミットしない
			- $ git revert {{commit}} -n
		- マージコミットの取り消し
			- $ git show {{commit}}
			- $ git revert -m {{n} {{commit}}
		- 例
			- 例1
				- $ git revert HEAD
			- 例2
				- $ git revert ★ -n
				- $ git revert ★ -n
				- $ git commit
			- HEAD以外も取り消せる。その場合、コンフリクトする可能性が高いが、やれなくはない。
	- $ git rebase -i ★
		- TODO コミットの書き換え、入れ替え、削除、統合を行う
	- $ git reset ★
		- $ git reset --soft HEAD~ ## add した状態にする
		- $ git reset --mixed HEAD~ ## add する前の状態にする
		- $ git reset --hard HEAD~ ## ファイルの状態も元に戻す
		TODO 2つ以上戻して、再コミットすることでコミットをまとめられる？rebase -i しなくてもOK？

- 別ブランチのコミットを現在のブランチにも追加する
	- $ git cherry-pick {{commit}}
		- 履歴はつながらない。revert と同系列
	
### TODO
- stash の話


## 3. リモート操作

### TODO github でリポジトリを作った直後のメッセージ

Quick setup — if you’ve done this kind of thing before
or	
https://github.com/sage1500/study-docs.git
Get started by creating a new file or uploading an existing file. We recommend every repository include a README, LICENSE, and .gitignore.

…or create a new repository on the command line
echo "# study-docs" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/sage1500/study-docs.git
git push -u origin main

…or push an existing repository from the command line
git remote add origin https://github.com/sage1500/study-docs.git
git branch -M main
git push -u origin main

…or import code from another repository
You can initialize this repository with code from a Subversion, Mercurial, or TFS project.



## 付録A. コミット指定方法

疑似BNF記法的に表現すると、以下のとおり。

```
{{commit}} :=
	  HEAD
	| {{commit-hash}}
	| {{branch-name}}
	| {{tag-name}}
	| {{commit}}~
	| {{commit}}~{{n}}
	| {{commit}}^
	| {{commit}}^{{n}}
```

|指定内容|意味|
|-|-|
|`HEAD`				|現在のブランチの最新コミット|
|`{{commit-hash}}`	|特定のコミット||
|`{{branch-name}}`	|特定のブランチの最新コミット|
|`{{tab-name}}`		|タグ|
|`{{commit}}~`		|(あるコミット指定からの)1世代前のコミット|
|`{{commit}}~{{n}}`	|(あるコミット指定からの)n世代前のコミット|
|`{{commit}}^`		|(あるコミット指定からの)1世代前の1番目のコミット|
|`{{commit}}^{{n}}`	|(あるコミット指定からの)1世代前のn番目のコミット(マージコミットに対してのみ使用可)|

具体例：
|事例|例|
|-|-|
|特定のコミット		|`15a7b33`
|				|※ ハッシュは全桁指定してもよいが、この例のように `git log --oneline` で表示される数桁あれば十分|
|特定のブランチの最新コミット	|`main`
|							|`develop`
|							|`feature/1234`
|							|`realease/1.0`
|特定のタグ			|`tag20201224`
|最新のコミット		|`HEAD`|
|1世代前のコミット	|`HEAD~`|
|					|`HEAD^`|
|2世代前のコミット	|`HEAD~~`|
|					|`HEAD~2`|
|					|`HEAD^^`|
|2世代前のコミット	|`HEAD~~~`|
|					|`HEAD~3`|
|					|`HEAD~2~`|
|					|`HEAD^^^`|
|1世代前のマージコミットの2番目のコミット|`HEAD^2`|
|1世代前のマージコミットの2番目のコミットのさらに2世代前のコミット|`HEAD^2~~`|


## 付録B. 簡易コマンドマニュアル

### B.x. タグを操作する。

|事例|コマンド例|
|-|-|
|一覧表示		|`git tag`|
||`git tag -n`|
|作成(軽量タグ)		|`git tag {{tag-name}} [{{commit}}]`|
|作成(注釈付きタグ)	|`git tag -a -m {{message}} {{tag-name}} [{{commit}}]`|
|削除			|`git tag -d {{tag-name}}`

### B.x. ブランチを操作する。

|事例|コマンド例|備考|
|-|-|-|
|一覧表示		 |`git branch [-a]`
|				|`git branch [-a] -l {{pattern}}`
|作成		|`git branch {{branch-name}}`
|削除			|`git branch -d {{branch-name}}`|マージ済でないと削除できない
|削除(強制)			|`git branch -D {{branch-name}}`
|切替	|`git checkout {{branch-name}}`
|作成＆切替|`git checkout -b {{branch-name}}`
|名前変更|`git branch -m {{branch-name}}`

### B.x. 差分を見る。

#### B.x.1. 差分の区間別のコマンド

|事例|コマンド例|備考|
|-|-|-|
|addするファイルの差分|`git diff`
|commitするファイルの差分|`git diff --staged`
|最新コミットの差分|`git diff HEAD~`
||`git show`|単に差分を見るだけなら `git show` の方が楽
|特定のコミットの差分|`git diff {{commit-hash}}~..{{commit-hash}}`
||`git show {{commit-hash}}`|単に差分を見るだけなら `git show` の方が楽
|任意区間の差分|`git diff {{commit}}..{{commit}}`|`{{commit}}`の指定方法は付録A参照
|pullするファイルの差分|`git diff HEAD..{{remote-branch-name}}`|先に `git fetch` しておく
|pushするファイルの差分|`git diff {{remote-branch-name}}..HEAD`|先に `git fetch` しておく


#### B.x.2. 対象ファイルを絞り込むコマンド

|事例|コマンド例|備考|
|-|-|-|
|特定ファイルだけの差分|`git diff {{略}} -- {{flie-path}}`|コマンドの最後に `-- {{file-path}}` を書くことで対象を絞れる


#### B.x.3. ファイル内容の差分以外を表示するコマンドオプション

|事例|コマンド例|備考|
|-|-|-|
|差分のあるファイル名のみ表示|`git diff {{略}} --name-only`
|差分のあるファイルの変更状態のみ表示|`git diff {{略}} --stat`
