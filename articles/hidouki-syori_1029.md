---
title: "非同期処理について"
emoji: "🖥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['javascript']
published: true
---

# 同期処理と非同期処理

まずは用語の定義からです。

**同期処理は、最初のコードから次のコードへと順次処理（実行）されていくことです**

対して、**非同期処理とは、ある処理が終了するのを待たずに、別の処理を実行することです**

# 非同期処理について詳しく

**非同期処理として最も典型的なのは、時間がかかる処理です。**
たとえば、**通信**には必ず時間がかかります。

Webページを関覧する際には、プラウザにURLを入力してからそのページが表示される前に多かれ少なかれ時間がかかります。
この時間の一部は自分の端末とWebサーバの間の通信に費やされています。自分の端末とWebサーバの間には物理的な距離がありますから、通信に時間がかかるのは自然なことです。javascriptが動作する環境では、基本的に通信は非同期処理となります。

また、別の例としては**ファイルの読み書き**も挙げられます。
ファイルが存在するのは自分の端末の中のHDDやSSDであり、一見物理的距離の問題がないように思われます。
しかし、プログラムを実行する中枢であるCPUの速度に比べるとHDDやSSDにアクセスするのは時間がかかるため、CPUから見たら時間がかかる処理ということになります。また、そもそも大容量のデータを読み書きする場合はいくら距離が近いとはいえ時間がかかります。

この2つが時間のかかる処理の代表例です。
また、アプリケーションサーバを書く場合にはほかのサーバの処理を待つ必要がある場合もあり、これも時間がかかる処理となります。
たとえば、データベースにアクセスする場合がこれにあたります。
この場合、データベースサーバにアクセスするための通信時間と、データベースサーバ側で実際に処理が行われる時間の合計が待ち時間となります。
このように時間のかかる処理がプログラムでどう表現されるかは言語や環境によって大きく異なりますが、JavaScript・TypeScript では非同期処理として表現されるのがとても一般的です。

# JavaScriptには、なぜ非同期処理が必要なのか

**同期処理だと、時間がかかる処理が終了するまで次の処理を行えないと効率が悪いため、非同期処理が対策としてあります。**

家事で例えれば、洗濯が終わるまで、家の掃除をしないと聞くと、極めて非効率のように感じますよね。

しかし、あるサーバからネットワークを介してデータ取得されるのを待ってから、簡単な四則演算を開始する。この処理も、条件によっては同期的な処理でも人の感覚だと違和感がないほど一瞬かもしれません。ですが、ネットワークを介したデータ取得と、ローカルのPC内だけで完結するような四則演算ではコストが大きく異なります。

**コストが大きく異なる２つの処理を同期的に順次処理していくのは効率が悪い**ので、その対策として非同期処理が必要になりました。

# メインスレッドと非同期処理の関係
まず、大事なポイントとして、JavaScriptはシングルスレッドの実装しかできません。

つまり、2つ以上の処理を並行して実行できないということです。

### 非同期処理のイメージ
非同期処理を実装すると、その処理はメインスレッドの並びから離れて次の処理に譲ります

![](/images/img-01.png)

代表的な非同期処理（非同期処理API）は、Promise や setTimeout などです。

### setTimeoutを用いた非同期処理
```js
console.log(1);
setTimeout(() => console.log(2), 5000);
console.log(3);
```

:::details 実行結果
1
3
2
:::

### タスクキューとコールスタック
では、メインスレッドの外から一時的に離れるとはどういう事なのでしょうか

非同期処理が実現されるために必要なブラウザの機能であるコールスタックとタスクキューについて説明します。

- コールスタック(Call Stack)
  - 関数は呼び出されるとコールスタックに追加されます
  - メインスレッドが占有されている状態はコールスタックにコンテキストが積まれている状態とも言えます
    - コールスタックにコンテキストが積まれている時は、タスクキューは待ちの状態で、コールスタックにあるコンテクストがはけるまではタスクは処理されません
  - また、コールスタックは、LIFO(Last In First Out)の構造を持った領域です
  - JavaScriptエンジンの内部に実装されています（メインスレッド）

- タスクキュー(Task Queue)
  - 実行待ちの非同期処理の行列のことを指します。別の言い方をすれば、非同期処理の実行順序を管理しているとも言えます
  - 非同期処理はタスクキューに入った順番で処理は実行されます
  - また、タスクキューは、FIFO(First In First Out)の構造を持った領域です
  - JavaScriptエンジンの外部に実装されています（メインスレッド外）

![](/images/img-02.png)

```js
const btn = document.querySelector('button');
btn.addEventListener('click', function task2() {
    console.log('task2 done');
});

function a() {
    setTimeout(function task1() {
        console.log('task1 done');
    }, 4000);

    const startTime = new Date();
    while (new Date() - startTime < 5000);

    console.log('fn a done');
}

```

# コールバック関数と非同期処理
```js
const first = () => {
  setTimeout(() => console.log("task1 done"), 2000);
  console.log("function first done");
};

const second = () => console.log("function second done");

first();
second();
```

:::details 実行結果
function first done
function second done
task1 done
:::

:::details 処理の流れ
-  スクリプトが実行される
-  コールスタックにグローバルコンテクストから関数first()が実行される
-  関数first()が実行されると、WEB API（非同期APIのsetTimeout()）が実行される
-  WEB API（非同期APIのsetTimeout()）が実行されると、タスクキューの中にsetTimeout内のコールバック関数が登録される
-  関数first()の実行が終了すると、グローバルコンテクストから関数first()がpopされる
-  グローバルコンテクストから関数second()が実行される
-  グローバルコンテクストから関数second()がpopされる
-  グローバルコンテクストが空になる
-  空になったことをイベントループがタスクキューに伝える
-  タスクキューに最初に入った順（今回はsetTimeout内のコールバック関数だけ）実行される
:::

### 問題　上のコードに変更を加え、出力結果を以下にしてください。
```
function first done
task1 done
function second done
```

:::details ヒント
:::message
- スクリプトが実行されると、コールスタックにグローバルコンテクストから関数first()が実行される
- WEB API（非同期APIのsetTimeout()）が実行される
- setTimeoutの中にはconsole.logの後に、関数second()がある
- グローバルコンテクストから関数first()がpopされる
- グローバルコンテクストが空になる
- 空になったことをイベントループが伝える
- タスクキューに最初に入った順（console.logの後に、関数second()実行される
:::

## 非同期処理のチェーン

複数の非同期処理をコールバック関数を使って連続的につなげて処理する方法を紹介します！

0秒から+1づつカウントアップさせたい場合、以下のように書きます。

```js
const sleep = (callback, val) => {
  setTimeout(() => {
    console.log(val++);
    callback(val);
  }, 1000);
};

sleep((val) => {
  sleep((val) => {
    sleep((val) => {
      sleep((val) => {
        sleep((val) => {}, val);
      }, val);
    }, val);
  }, val);
}, 0);
```

ネストが深すぎますね。。

コールバック関数の中に、コールバック関数を実行させる方法で変数`val`をカウントアップさせています。

これが、コールバック関数を用いた複数の非同期処理を順番に実行（非同期のチェーン）するための方法です。

### 問題

これで、実行順序を確かに制御できますが、**問題は可読性が悪いことです。**
この問題を解決するために、ES6では`Promise`というオブジェクトが生まれました。

# Promise とは

**Promiseとは、非同期処理をより簡単かつ可読性が上がるように書けるようにしたJavaScriptのオブジェクトです。**

### Promise の状態

Promiseには3つの状態があります。

```
pending：非同期処理の実行中の状態を表す
fulfilled：非同期処理が正常終了した状態を表す
rejected：非同期処理が異常終了した状態を表す
```

### Promise の書き方

一般的なPromiseの書き方は下記です。

```js
new Promise(
  // something
).then(
  // something
).catch(
  // something
).finally(
  // something
);
```

Promiseの引数としてコールバック関数を設定します。
↓のコールバック関数は引数を2つとります。

- resolve
- reject

```js
new Promise((resolve, reject) => {
}).then(
  // something
).catch(
  // something
).finally(
  // something
);
```

### resolve

Promiseの状態が、fulfilledになったら resolve が実行されます。
resolve が実行された場合は、thenメソッド内部が実行されます。
thenメソッド内部のコールバック関数には、resolve実行時の実引数が渡されます。それが（"hoge"）です。

```js
new Promise((resolve, reject) => {
  resolve("hoge");
}).then((data) => console.log(data)  // => "hoge"
).catch(
  // something
).finally(
  // something
);
```

thenメソッドが実行された場合、catchメソッドは無視されます。
最後にfinallyメソッドが実行されます。

```js
new Promise((resolve, reject) => {
  resolve("hoge");
}).then((data) => console.log(data) // => "hoge"
).catch(
  // something
).finally(() => console.log("finally"));
```

### reject

Promiseの状態が、rejectedになったら reject が実行されます。

**rejectと言うのは、Promise内のコールバック実行中に何らかのエラーが発生した場合、それをPromiseに通知するために使用する関数です。**

rejectが実行される場合は、cathchメソッド内のコールバック関数が実行されます。promise内にある、rejectの実引数が渡され、実行されます。

そして、catchメソッド内のコールバック関数が実行された後に、finallyメソッド内のコールバック関数が実行されます。

```js
new Promise((resolve, reject) => reject("fuga"))
    .then()
    .catch((data) => console.log(data)) // => "fuga"
    .finally(() => console.log("finally")); // => "finally"
```

### Promiseを実際に書いてみる

```js
new Promise((resolve, reject) => {
  console.log("Promise");
  resolve();
}).then(() => console.log("then"));

console.log("global end");
```

:::details 実行結果
Promise
global end
then
:::message
2番目に global end が出力されていることがわかります。
これはなぜかと言うと、thenメソッド内部は非同期処理なので、タスクキューに積まれたからです。あと、グローバルコンテキスト上にあるconsole.log("global end")がコールスタックに2番目に積まれているからです。
それが実行されれば、コールスタック（メインスレッド）が空になります。そのタイミングでthenメソッドが実行されます。
:::

### 問題　上のコードに変更を加え、出力結果を以下にしてください。  (Promiceチェーンを使うこと)


```txt
Promise 
global end 
then1 
then2 
then3
```

:::details 答え
```js
new Promise((resolve, reject) => {
  console.log("Promise");
  resolve();
})
  .then(() => {
    console.log("then1");
  })
  .then(() => {
    console.log("then2");
  })
  .then(() => {
    console.log("then3");
  });
console.log("global end");
```
:::

# AsyncとAwait

**AsyncとAwaitは、Promiseをさらに直感的に書けるようにしたものです。**

### Async

**Asyncを使って宣言された関数は、Promiseオブジェクトを返却します。**

次のように関数の前に置くことができます:

```js
async function f() {
  return 1;
}
```

Asyncが返すものは、Promiseなので、thenメソッドをつなげられます

注意点としては、Asyncは関数コンテクストにしか使えないところです。

上のコードは 結果 1 を持つ解決された promise を返します。

```js
async function f() {
  return 1;
}

f().then(alert); // 1
```

これは、以下と同義です。

```js
async function f() {
  return Promise.resolve(1);
}

f().then(alert); // 1
```

### Async FunctionはPromiseを返す
Async Functionとして定義した関数は必ずPromiseインスタンスを返します。 具体的にはAsync Functionが返す値は次の3つのケースが考えられます。

1. Async Functionが値をreturnした場合、その返り値を持つFulfilledなPromiseを返す
2. Async FunctionがPromiseをreturnした場合、その返り値のPromiseをそのまま返す
3. Async Function内で例外が発生した場合は、そのエラーを持つRejectedなPromiseを返す

以下は、Async Functionがそれぞれの返り値によってどのようなPromiseインスタンスを返すかを示すものです。

```js
// 1. resolveFnは値を返している
// 何もreturnしていない場合はundefinedを返したのと同じ扱いとなる
async function resolveFn() {
    return "返り値";
}
resolveFn().then(value => {
    console.log(value); // => "返り値"
});

// 2. rejectFnはPromiseインスタンスを返している
async function rejectFn() {
    return Promise.reject(new Error("エラーメッセージ"));
}

// rejectFnはRejectedなPromiseを返すのでcatchできる
rejectFn().catch(error => {
    console.log(error.message); // => "エラーメッセージ"
});

// 3. exceptionFnは例外を投げている
async function exceptionFn() {
    throw new Error("例外が発生しました");
    // 例外が発生したため、この行は実行されません
}

// Async Functionで例外が発生するとRejectedなPromiseが返される
exceptionFn().catch(error => {
    console.log(error.message); // => "例外が発生しました"
});
```

### Await

**Awaitは、promise が確定しその結果を返すまで、JavaScript を待機させます。**

**関数の中でawaitが記載されている場合、必ずasyncを関数の先頭に書かないとエラーなります。**

```js
async function f() {

  let promise = new Promise((resolve, reject) => {
    setTimeout(() => resolve("done!"), 1000)
  });

  let result = await promise; // promise が解決するまで待ちます (*)

  alert(result); // "done!"
}

f();
```

関数の実行は行 (*) で “一時停止” し、promise が確定したときに再開し、result がその結果になります。 そのため、上のコードは1秒後に “done!” を表示します。

await は文字通り promise が確定するまで JavaScript を待ってから、その結果で続きます。
その間、エンジンは他のジョブ(他のスクリプトを実行し、イベントを処理するなど)を実行することができるため、CPUリソースを必要としません。

では、なぜawaitの「非同期処理の結果を待つ」という処理が重要なのでしょうか。

それは、以下のコードを見れば理解できると思います。

```js
async function getUserAccount() {
    // 非同期処理
    const response = await fetch('https://api.aoikujira.com/tenki/week.php?fmt=json&city=319');
    if (!response.ok) {
        return '気象情報を取得できませんでした';
    }
    return '気象情報を取得できました';
}

function log() {
    // 気象情報を取得
    const message = getUserAccount();
    console.log(message);
}

log();
```

getUserAccountはasyncをつけたことによって非同期処理となっているので、getUserAccount()を実行する際もawaitをしないと処理が終わる前に次の行のconsole.logを実行してしまいます。

getUserAccountの処理が完了してから次を実行してほしいため、log関数を以下の様に修正します。

```js
// getUserAccountをawaitしたいのでasyncを前につける
async function getUserAccount() {
    // 非同期処理
    const response = await fetch('https://api.aoikujira.com/tenki/week.php?fmt=json&city=319');
    if (!response.ok) {
        return '気象情報情報を取得できませんでした';
    }
    return '気象情報情報を取得できました';
}

async function log() {
    // getUserAccountは非同期関数のため、awaitする
    const message = await getUserAccount();
    console.log(message);
}

log()
```

### エラー処理
もし promise が正常に解決すると、await promise は結果を返します。しかし拒否(reject) の場合はエラーをスローします。それはちょうどその行に throw 文があるように振る舞います。


この以下のコードは:
```js
async function f() {
  await Promise.reject(new Error("Whoops!"));
}
```

これと同義です。

```js
async function f() {
  throw new Error("Whoops!");
}
```

### 問題　async/awaitを用いて、クジラ週間天気APIから今日の日付と今日の天気を出力してください

:::details 解答例
```js
const weatherFn = async () => {
  const weather = await fetch(
    "https://api.aoikujira.com/tenki/week.php?fmt=json&city=319"
  );
  return weather;
};

weatherFn().then(async (weather) => {
  const json = await weather.json();
  const todayWeather = json[319][1].forecast
  console.log(todayWeather);
});
```
:::