# Django tutorialをやってみる
## 参照先

公式のもの

https://docs.djangoproject.com/ja/1.11/intro/tutorial01/

## tutorial02

データベースをやるらしいですよ。

## Databaseの設定

mysite/settings.py を編集する。デフォルトではSQLiteを使用する。
（本番ではポスグレとか使ってね  [参照先](https://docs.djangoproject.com/ja/1.11/topics/install/#database-installation)）

DATABASESの'default'項目内の設定を変える。
ENGINEがデータベースの種類（SQliteとか、postgresqlとか、mysqlとか）
NAMEがデータベースの名前（デフォルト値もあるみたい）

SQLite以外の場合は、USER、PASSOWORD、HOSTなどの設定も必要。

**TIPS**
`TIME_ZONE` の設定は罠がある。'JST' とか正しそうだけど、実際はエラーになる。
https://torina.top/detail/339/

> TIME_ZONE = 'Asia/Tokyo'

これらの設定をしたら、下記コマンドを実行してやるとこのようなログが流れる。
`INSTALLED_APP` の設定に従い、必要なデータベーステーブルを作成してくれる。

```
[10/25 23:48:05] $ python manage.py migrate                      (git)-[master]
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying sessions.0001_initial... OK
```
時間があったら中身を見てみましょう。（ちょっと今は流す）

## モデルの作成

DjangoのモデルはRoRと若干違うらしい。読んでもあまり理解できない…。（RoRをあまり知らないので）

DRY則に従っていて、ただ1つの場所でデータモデルを定義し、そこから自動的にデータを取り出すことにある、そしてこれはマイグレーションを含む、と記載してある。
マイグレーションはモデルのファイルから作成されるようだ。

polls/models.py を編集する。ここでは `django.db.models.Model` のサブクラスで示され、各モデルのクラス変数はモデルのデータベースフィールドを表す。

- CharField: 文字フィールド
- DateTimeField: 日時フィールド
- IntegerField: 数値フィールド 

Fieldインスタンスそれぞれの名前はPythonコードで使うとともに、データベースの列名として使うことになる。

Fieldの最初の引数位置には、オプションとして人間が読みやすいフィールド冥を指定可能である。（ドキュメントや内省機能で使用可能）
例では、 `Question.pub_date` にのみ適用している。

CharFieldでは、`max_length` の指定が必須。データベーススキーマやバリデーションで使用されることになる。

## モデルを有効にする

モデルを記載すると、Djangoは

- アプリケーションのデータスキーマを作成（CREATE TABLE文の実行）
- QuestionやChoiceオブジェクトにPythonからアクセスするためのデータベースAPIを作成可能

上記のことを実行可能になる。しかしプロジェクトへ登録する必要がある。

TIPS: Djangoアプリケーションは「プラガブル(pluggable)」であり、特定アプリケーションと結びついていない。そのため、アプリケーションを複数アプリケーションで使用することや、単体での配布が可能。

構成クラスへの参照を `INSTALLED_APPS` 設定に追加する必要がある。
PollsConfigクラスなら、polls/app.pyにあるので、パスは `polls.apps.PollsConfig` となる（ドットつなぎになる）。
mysite/settings.py を編集し、パスを追加する。

その後、コマンドを実行する。登録していると見つからず、pollsの方でエラーが有ると例外で落ちたりする。

```shell
python manage.py makemigrations polls
```

成功時ダイアログ

```
Migrations for 'polls':
  polls/migrations/0001_initial.py
    - Create model Choice
    - Create model Question
    - Add field question to choice
```

makemigrationsにより、Djangoにモデルの変更があったことを伝え、マイグレーションで保存できた。Djangoはマイグレーションを実行し、データベーススキーマを自動で管理するコマンドが付属している。

sqlmigration コマンドを使ってみよう。create table文を発行できる。SQLはDB毎に異なるので注意。（私の環境ではSQLite設定なので、サイトに乗っているPostgreSQLのものと異なる)

```shell
python manage.py sqlmigrate polls 0001
```

TIPS: `python manage.py check` を実行するとプロジェクトに問題がないかをマイグレーション作成やデータベースアクセスを行うことなく、確認可能。

migrate を再度実行し、モデルのテーブルをデータベースに作成する。

```shell
$ python manage.py migrate 
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, polls, sessions
Running migrations:
  Applying polls.0001_initial... OK
```

migrate コマンドは全ての適応されていないマイグレーションを保続し、データベースに対して実行する。（Djangoはデータベース内の `django_migrations` と呼ばれる特別なテーブルを利用してどれが適用されているのか追跡している）
重要なのはモデルに対して行った変更をデータベースのスキーマに同期すること。

マイグレーションは強力なツールでプロジェクトの開発・拡張に合わせてモデルを変更可能。（データベース、テーブルを削除して作り直す必要もない）

今のところのモデル変更を実行するための3 STEP

- モデルを変更する（models.py)
- モデルの変更によるマイグレーション作成のため、`python manage.py makemigrations` を実行
- データベースに変更を適用するため、 `python manage.py migrate` を実行する

manage.py ユーティリティでできることはココを見る。 https://docs.djangoproject.com/ja/1.11/ref/django-admin/

## API で遊んでみる

次回はここから。

