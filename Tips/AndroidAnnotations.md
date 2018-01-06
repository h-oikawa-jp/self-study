※2015年くらいに書いたメモ書き(遺物)

# 概要：AndroidAnnotationsとは
Android開発に付き物となるViewやリソースのアタッチ、各種イベントリスナーの登録などの定型コードをアノテーションによって簡略化するライブラリ。
- [公式HP](http://androidannotations.org/)
- [Githubリポジトリ](https://github.com/excilys/androidannotations)
公式トップのサンプルコードにあるように、Android特有のボイラープレートが削減される事でソースコードの量や見通しの良さが格段に改善される。


# 導入
AndroidAnnotationsは[android-apt](https://bitbucket.org/hvisser/android-apt)でビルド時にコードを自動生成する事で機能を実現している。
よって導入にはAndroidAnnotationsライブラリ本体の依存関係に加えて、android-aptも共にビルド設定に追加する必要がある。

ここではAndroidStudio + Gradleのプロジェクト構成での導入についてのみ簡単に記述する。

## <プロジェクトルート>/build.gradle の設定
buildscript.dependenciesにandroid-aptを追加。
```gradle
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.1.0'
        classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8' // 追加
    }
}
...(略)...
```

## <モジュール>/build.gradle の設定
gradleのandroid-aptプラグイン適用＆AndroidAnnotationsの依存関係を追加。
```gradle
...(略)...
apply plugin: 'android-apt' // 追加

dependencies {
    ...
    apt "org.androidannotations:androidannotations:{バージョン}" // 追加
    compile "org.androidannotations:androidannotations-api:{バージョン}" // 追加
}

// ↓ 追加
apt {
    arguments {
        androidManifestFile variant.outputs[0]?.processResources?.manifestFile
    }
}
// ↑ 追加
...(略)...
```


# 基本アノテーション
**各アノテーションの詳細な使用方法は[Cookbook](https://github.com/excilys/androidannotations/wiki/Cookbook)を参照。**
使用可能なアノテーション一覧は[こちら](https://github.com/excilys/androidannotations/wiki/AvailableAnnotations)。

以下、個人的によく使用したアノテーションと注意点などをPickUpする。

## コンポーネント有効化アノテーション
AndroidAnnotationsによる各種アノテーションを使用するコンポーネントに付加するアノテーション。
基本的にこれらをつけたクラス内でないと他のAndroidAnnotationsが使えない。

例）@EActivity、@EFragment、@EApplication、@EBean、@EService ...

このアノテーションを付けたクラスを継承して、その他各アノテーションの処理を実装したクラスがビルド時に自動生成される。
その際のクラス名は元のクラス名の末尾にアンダーバー(_)を付与した名前となる。

またEActivityなどのアノテーションでは引数にレイアウトのIDを渡す事でsetContentViewで適用するレイアウトを指定できる。

例）
```java
@EActivity(R.layout.activity_main)
public class MainActivity extends AppCompatActivity {
}
```
↓自動生成されるクラス
```java
public final class MainActivity_ extends MainActivity implements HasViews {
    private final OnViewChangedNotifier onViewChangedNotifier_ = new OnViewChangedNotifier();

    @Override
    public void onCreate(Bundle savedInstanceState) {
        OnViewChangedNotifier previousNotifier = OnViewChangedNotifier.replaceNotifier(onViewChangedNotifier_);
        init_(savedInstanceState);
        super.onCreate(savedInstanceState);
        OnViewChangedNotifier.replaceNotifier(previousNotifier);
        setContentView(R.layout.activity_main);
    }
    ...(略)...
}
```

またおまけとしてActivityのintentやFragmentのインスタンスを作成するbuilderメソッドが自動実装される。
@Extraや@FragmentArgを使用したクラスではメソッドチェインでBundleデータを設定できる。
```java
MainActivity_.intent(this).extraArg("someArg").start();
```
```java
Fragment fragment = SomeFragment_.builder().fragmentArg("someArg").build();
```

### 注意点
実際のAndroidコンポーネントとして動作するのはあくまで自動生成された方のクラス。
その為、''AndroidManifestやインテントなどでコンポーネントを指定する際はアンダーバー付きのクラス名を指定しなければいけない。''


## インジェクション系アノテーション
インスタンスフィールドに付与する事でViewやコンポーネントを自動でインジェクションするアノテーション。

例）@ViewById、@App、@Bean、@Extra、@FragmentArg、@RootContext、@SystemService ...

### 注意点1
実際に各要素がフィールドにセットされるタイミングに注意。
AndroidのサンプルでありがちなonCreateなどでViewを操作するコードをそのまま流用してきたりすると、
@ViewByIdを付けたフィールドにまだインスタンスがインジェクトされておらずヌルポで落ちたりする事がある。

必ず各アノテーションの要素がインジェクトされた後に処理を実行したい場合は、下記メソッドアノテーションが使える。
例）@AfterExtras、@AfterInject、@AfterViews

### 注意点2
''付与するメンバーの可視性はprotected以上でないとならない。''
⇒対象へのインジェクション処理は自動生成されるサブクラスで実装される為、privateメンバーに適用すると当然エラーとなる。


## イベントリスナー系アノテーション
onClickListenerなどのイベント処理をアノテーションベースでメソッド単位に切り分けて実装できる。
アノテーションの引数に対象ViewのリソースIDを指定する。

例）@Click、@LongClick、@ItemSelect、@KeyDown、@TextChange、@FocusChange、@CheckedChange ...
```java
@Click(R.id.myButton1)
void onClickBtn1(View clickedView) {
    ...
}
```


# その他アノテーション
上記基本アノテーション以外で、状況によって使用すると便利なアノテーション。

## [@Background、@UiThread](https://github.com/excilys/androidannotations/wiki/WorkingWithThreads#background)
対象メソッドの実行スレッドを指定できるアノテーション。**かなり便利**

3.0以降のAndroidではUI(メイン)スレッドからネットワークへ直接アクセスしようとするとエラーとなる。
その為、本来なら一々AsyncTaskなどで非同期処理として実装してやらないといけない。(...スゴく面倒)

参考）https://akira-watson.com/android/asynctask-1.html

このアノテーションを使用すると、ネットワーク通信部分などの処理をメソッドに切り出し@Backgroundを付けるだけで
該当処理をバックグラウンドスレッドで実行してくれるようになる。
@UiThreadはその逆。@Backgroundなどの結果を受けてUIスレッド上でViewに反映する処理等に使用する。

おまけとして@Background(delay=2000)のように引数を指定する事で、非同期処理の遅延実行なども出来る。

## [@InstanceState](https://github.com/excilys/androidannotations/wiki/Save-instance-state)
プリミティブ型やParcelable実装クラスのフィールドに付与すると、InstanceStateのsave&load時に自動でシリアライズorデシリアライズしてくれる。

## [@SharedPref、@Pref](https://github.com/excilys/androidannotations/wiki/SharedPreferencesHelpers)
interfaceを定義するだけでSharedPreferenceの保存＆読込メソッドを備えたデータモデルクラス(のようなもの)を自動生成してくれる。
SharedPreference用のキーを自分で定義せずとも良くなるのが地味に便利。

## その他
（気が向いたら随時詳細を追記予定）
- @OptionsMenu、@OptionsMenuItem
- @OnActivityResult、@OnActivityResult.Extra
- @Receiver、@Receiver.Extra、@ReceiverAction、@ReceiverAction.Extra
- @Rest、@RestService


# 参考リンク一覧
- [Android Developers | Android 入門 (公式)](http://developer.android.com/intl/ja/guide/index.html)
- [Android開発 基礎トレーニング](http://mixi-inc.github.io/AndroidTraining/)
- [AndroidAnnotations公式HP](http://androidannotations.org/)
- [【Android】ソースコードダイエットのためにAndroidAnnotationsを使おう！](https://blog.yohei.org/android-androidannotations-01/)


# Android関連おまけメモ
## Activity・Fragmentのライフサイクルについて
[公式のライフサイクル説明](https://developer.android.com/training/basics/activity-lifecycle/starting.html?hl=ja)では特定シチュエーションだけが抜粋されていて全容を把握しきれない。
実際の開発時には以下のサイトを参考にした方が良い。
- https://github.com/xxv/android-lifecycle
- http://qiita.com/calciolife/items/39b2696a9a03e8591d40

## AndroidでLombok
Android ProjectにてLombokを導入しようとした際にエラーでハマッたのでその[対処法](https://projectlombok.org/setup/android.html)

⇒プロジェクトルートにlombok.configというファイルを作り以下を追記
```java
lombok.anyConstructor.suppressConstructorProperties = true
```
