# Gruntを無効化する

Gruntタスクの組み込みを無効にするためには、Gruntfile(と場合によっては[`tasks/`](/#/documentation/anatomy/myApp/tasks)フォルダーのこともあります)を削除するだけで大丈夫です。同様にGrunt hookの無効化も出来ます。これをやるには`.sailsrc`のhooksで以下のように`grunt`を`false`にするだけで出来ます:


```json
{
    "hooks": {
        "grunt": false
    }
}
```
###これをSASSやAngular、クライアントサイドJadeテンプレートなどのためにカスタマイズできますか。

もちろん！あなたのプロジェクトファイルの`tasks/`ディレクトリにある関係するタスクを書き換えるだけでできますし、例えば[SASS](https://github.com/sails101/using-sass)のように新しいものを作ることでも実現できます。
もしもGruntを使いたいけれどもデフォルトのフロントエンドのを使いたくない場合はあなたのプロジェクトのアセットフォルダーを削除して関連するフロントエンド関連のタスクを`grunt/register/`と`grunt/config/`から削除してください。

また、`sails new myCoolApi --no-frontend`を実行することでアセットフォルダーやフロントエンド関連のGruntタスクを将来のプロジェクトのために省略することも出来ます。 

さらに`sails-generate-frontend`モジュールを別のサードパーティの物に変えたり[自分で開発する](https://github.com/balderdashy/sails-generate-generator)ことも出来ます。これをすることでネイティブのiOSアプリやAndroidアプリ、CordovaアプリSteroidsJSアプリをSailsで開発できる新たなboilerplateを`sails new`で作ることが出来ます。


<docmeta name="uniqueID" value="DisablingGrunt970874">
<docmeta name="displayName" value="Disabling Grunt">

