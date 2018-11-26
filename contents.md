# Introduction </br>dependency injection
</br>

2018-11-27

@orepuri

---

### アジェンダ

</br>

Dependency Injection(DI)とは

DIの方法

DIフレームワークについて

---

### DI = 依存性の注入

</br>

依存するオブジェクトを外部から</br>
注入できるようにすること

---

### コード例: SNSつぶやき

```scala
class TweetService {

  def tweet(userId: UserId, message: Message): Eithter[TweetError, Tweet] = {
    // MessageValidator, UserRepository, TweetRepositoryに依存している
    val messageValidator = new MessageValidator(...)
    val userRepo = new UserRepository(...)
    val tweetRepo = new TweetRepository(...)

    for {
      validMessage <- messageValidator.apply(message)
      user <- userRepo.findBy(userId)
    } yield {
      val tweet = Tweet(user, validMessage)
      tweetRepo.save(tweet)
    }
  }
}
```
---

### コード例: DIを使った場合

```scala
// 3つの依存オブジェクトをコンストラクタで引き渡し
class TweetService(
  messageValidator: MessageValidator, 
  userRepo: UserRepository, 
  tweetRepo: TweetRepository) {

  def tweet(userId: UserId, message: Message): Eithter[TweetError, Tweet] = {
    for {
      validMessage <- messageValidator.apply(message)
      user <- userRepo.findBy(userId)
    } yield {
      val tweet = Tweet(user, validMessage)
      tweetRepo.save(tweet)
    }
  }
}
```

---

### コード例: DIを使った場合

```scala
val validator = new MessageValidator(...)
val userRepo = new UserRepository(...)
val tweetRepo = new TweetRepository(...)

val tweetService = new TweetService(validator, userRepo, tweetRepo)

tweetService.tweet(UserId(1), Message("hogehoge"))
```

---

### DIのメリット

</br>

オブジェクト間の結合性を低下できる

オブジェクト(実装)の入れ替えがやりやすい

テストが書きやすい

---

### オブジェクト間の結合性の低下

```scala
class TweetService {

  def tweet(userId: UserId, message: Message): Eithter[TweetError, Tweet] = {
    // 依存オブジェクトの生成方法を知っている
    val messageValidator = new MessageValidator(...)
    val userRepo = new UserRepository(...)
    val tweetRepo = new TweetRepository(...)

    for {
      // 依存オブジェクトの使い方を知っている
      validMessage <- messageValidator.apply(message)
      user <- userRepo.findBy(userId)
    } yield {
      val tweet = Tweet(user, validMessage)
      tweetRepo.save(tweet)
      ...
```

TweetServiceが依存オブジェクトに強く結合している

---

### Tweet Service

</br>

Messageの検証やTweetの永続化など</br>
tweetユースケース(処理のフロー)の責務を持つ

ValidatorやRepositoryのインタフェース(使い方)は</br>
知っている必要はあるが生成方法や実装を知る必要はない

---

### DIを使ったパターン

```scala
class TweetService(
  messageValidator: MessageValidator, 
  userRepo: UserRepository, 
  tweetRepo: TweetRepository) {

  def tweet(userId: UserId, message: Message): Eithter[TweetError, Tweet] = {
    for {
      validMessage <- messageValidator.apply(message)
      user <- userRepo.findBy(userId)
    } yield {
　　  ...
```

依存オブジェクトを外部から渡すことで</br>
依存オブジェクトの生成方法を知らなくてもよい


---

### テストが書きやすい

```scala
for {
  validMessage <- messageValidator.apply(message)
  user <- userRepo.findBy(userId)
} yield {
  val tweet = Tweet(user, validMessage)
  tweetRepo.save(tweet)
}
```

---

### テストのパターン

</br>

validationが成功した場合と失敗した場合

Userの取得が見つかった場合と見つかった場合とエラーになった場合

Tweetの保存が成功した場合とエラーになった場合

*上記の組み合わせのパターンが必要*

---

### DIを使わない場合

</br>

実際のオブジェクトを使うため, すべてのパターンを網羅する</br>
データを用意する必要がある

レポジトリの実装がDBならDBにテスト用のデータを登録

---

### DIを使う場合

```scala
// Specs2 + Mockito
"#tweet" should {
  "return TooLongMessage if length of message is greater than 140" in {
    val validator = mock[MessageValidator]
    validator.apply(longMessage) returns
       TooLongMessageError(longMessage.length)

    val service = new TweetService(validator, ...)
    service.tweet(userId, longMessage) must
       beLeft(TooLongMessageError(longMessage.length))
  }
}
```

モックを使うことで実際のデータがなくてもパターンを網羅できる

---

### DIを使う場合

```scala
"#tweet" should {
  "return UserNotFound if not found the user" in {
    val userRepo = mock[UserRepository]
    userRepo.findBy(userId) returns UserNotFound(userId)

    val service = new TweetService(..., userRepo, ...)
    service.tweet(userId, message) must beLeft(UserNotFound(userId))
  }
}
```

---

### DIの方法

</br>

コンストラクタインジェクション

インタフェースインジェクション

setterインジェクション

フィールドインジェクション

---

### コンストラクタインジェクション

```scala
class TweetService(
  messageValidator: MessageValidator, 
  userRepo: UserRepository, 
  tweetRepo: TweetRepository) {
  ...
```

コンストラクタで依存オブジェクトを渡す

コンストラクタを見れば依存がわかる

---

### インタフェースインジェクション

```scala
def tweet(
  userId: UserId,
  message: Message,
  messageValidator: MessageValidator,
  userRepo: UserRepository,
  tweetRepo: TweetRepository): Eithter[TweetError, Tweet] = {

  ...
}
```

メソッドで注入する

面倒

---

### setterインジェクション

```scala
class TweetService {
  private var messageValidator: MessageValidator

  def setMessageValidator(messageValidator: MessageValidator): Unit = {
    this.messageValidator = messageValidator
  }

  def tweet(...) { ... }
}
```

依存を注入する専用のメソッドを用意

Immutableになりいつ設定, 変更されるかわかりずらい

メソッドを確認しないと何が注入されるかわからない

---

### フィールドインジェクション

```scala
class TweetService {
  var messageValidator: MessageValidator

  def tweet(...) { ... }
}

...

tweetService.messageValidator = new MessageValidator

```

フィールドに直接注入する

Immutableになりいつ設定, 変更されるかわかりずらい

---

基本的にコンストラクタインジェクションを使う

Immutable

依存が明確

インタンスが生成されるタイミングで依存オブジェクトも</br>
存在することが保証される

---

### DIの問題点

</br>

(DIしない場合に比べ)依存関係が生まれる

アプリケーションが大規模なほど依存関係も増える

手動で1つ1つオブジェクトを生成して依存関係を定義するのは面倒

注入するために依存オブジェクトを引き回さないといけない(場合がある)
---


### DIフレームワーク

</br>

オブジェクトの生成や注入を自動でやってくれる

Google Guice, Spring Frameworkなど

---

### Google Guice

```java
public class RealBillingService implements BillingService {
  private final CreditCardProcessor processor;
  private final TransactionLog transactionLog;

  // JSR-330 (javax.inject)
  @Inject
  public RealBillingService(CreditCardProcessor processor,
      TransactionLog transactionLog) {
    this.processor = processor;
    this.transactionLog = transactionLog;
  }
  ...
```

---

### Google Guice

```java
public class BillingModule extends AbstractModule {
  @Override 
  protected void configure() {
    bind(TransactionLog.class).to(DatabaseTransactionLog.class);
    bind(CreditCardProcessor.class).to(PaypalCreditCardProcessor.class);
    bind(BillingService.class).to(RealBillingService.class);
  }
}
...
public static void main(String[] args) {
  Injector injector = Guice.createInjector(new BillingModule());
  BillingService billingService = injector.getInstance(BillingService.class);
  ...
}
```

自動で依存オブジェクトを生成してくれる

---

### 個人的なDIフレームワークの感想

</br>

特になくてもよい気も (特にScalaでは)

便利な機能も多いが学習コストが高い(Guice)

アプリケーションのコードにDIフレームワークの依存を</br>
持ち込みたくない(@Injectとか)

---

### DIフレームワークは依存関係の夢を見るか?

</br>

DIフレームワークだけで大規模なアプリケーションの依存関係の</br>
問題をすべて解決してくれるとは思えない

依存を注入するためにオブジェクトを引き回さないといけない問題

依存オブジェクトを引き回さないといけない場合</br>
設計を見直したほうがいい場合もある

適切な依存関係の分離やコンポーネント化は必須


---

### DIのポイント

</br>

大事なのはフレームワークではなくDIできるコードになっていること

ただし, 下記の機能はあるとうれしい

依存オブジェクトのライフサイクルの管理

コンポーネント単位での依存管理

---

### ライフサイクル

```scala
// AirFrameによるライフサイクル管理
trait MyServerService {
  val service = bind[Server]
    .onInit( _.init )   // Called when the object is initialized
    .onInject(_.inject) // Called when the object is injected 
    .onStart(_.start)   // Called when session.start is called
    .beforeShutdown( _.notify) // Called right before all shutdown hook is called
                               // Useful for adding pre-shutdown step 
    .onShutdown( _.stop ) // Called when session.shutdown is called
  )
}
```

---

### コンポーネント単位での依存管理

```

<ServiceComponent1> ---> <RepositoryComponent1> ---> <DBComponent>
        |                                                        |
        |                                                        |
       +--->  <UtilityComponent1>         +---> <KvsComponent>
        |
        |
<ServiceComponent2> ---> <RepositoryComponent2> ---> <DBComponent>
        |
        |
       +---> <UtilityComponent2>

```

---

### まとめ

</br>

DIによりオブジェクト間の結合性を低下でき</br>
変更やテストが書きやすいコードになる

修正しずらい/テストが書きづらい複雑なコードは</br>
責務を分離した上でDIする

DIフレームワークはメリットとデメリットを理解した上で導入しよう

