# javascript-tuning-tips

## React

#### React.memoとコンポーネント単位でのチューニング

React.memoはコンポーネントを[メモ化](https://ja.wikipedia.org/wiki/%E3%83%A1%E3%83%A2%E5%8C%96)し、  
ウォッチしている変数に変更があったときにメモ化していたコンポーネントを破棄し、  
再レンダ、再びメモ化します。

第1引数にコンポーネントを返す関数を渡し、第2引数に**Propsが同一であるかを判断する関数**を渡します。  
shouldComponentUpdateでは再レンダを行う際にtrueを返しますが、  
memoでは再レンダを**行わない**際にtrueを返すので注意が必要です。  

第2引数に何も渡さなかった場合はPureComponent(shallowEqual)と同等の比較によりメモの更新判断がなされます。

PureComponentとshouldComponentUpdateがFunctional Componentになる上で合体したという理解で大体OKです。

```js
// below two components are same
class TodoListClassStyle extends React.PureComponent {
  constructor(props) {
    super(props)
  }

  render() {
    return (
      <ul>
        { 
          this.props.list.map((item, index) => (
            <li key={index}>
              { item }
            </li>
          ))
        }
      </ul>
    )
  }
}

const TodoListFCStyle = React.memo(({ list }) => (
  <ul>
    { 
      list.map((item, index) => (
        <li key={index}>
          { item }
        </li>
      ))
    }
  </ul>
))
```

上が従来の記法、下がmemoを使用した記法です。  
慣れてしまえばだいぶすっきりした記法ですね。

ここからが本題で、memoを利用しどうやってチューニングをしていくかの話になります。  
私はすべてのコンポーネントに同一に適応できるチューニングTipsというのはないと考えていて、  
**各コンポーネントのpropに応じて最善手を選択する**事になると考えています。  
銀の弾丸なんてなかった。

そうは言っても、大体以下のパターンのどれかに収まるので身構えなくても良かったりします。

1. propsがない場合
2. propsがshallow equalで対応しきれる場合
3. propsがdeep equalで対応する必要がある場合
4. 変化しうるプロパティが特定できている場合

#### 1.propsがない場合
この場合は`() => true`なmemoか、普通のSFCが早いかの比較になります。計測しましょう。

**※ 以下のパフォーマンス計測は、create-react-appで作成したプロジェクトを`yarn build`したもので計測しています。**  
　 **よって、開発環境での結果と異なる場合があります**

```js
import React from 'react'
import ReactDOM from 'react-dom'

const ConflictSFC = () => (
   <p>
     conflict歌います。<br />
     ズォールヒ～～↑ｗｗｗｗヴィヤーンタースｗｗｗｗｗワース フェスツｗｗｗｗｗｗｗルオルｗｗｗｗｗ
     プローイユクｗｗｗｗｗｗｗダルフェ スォーイヴォーｗｗｗｗｗスウェンネｗｗｗｗヤットゥ ヴ ヒェンヴガｒジョｊゴアｊガオガオッガｗｗｗじゃｇｊｊ
   </p>
)
  
const ConflictMemo = React.memo(() =>
  <p>
    conflict歌います。<br />
    ズォールヒ～～↑ｗｗｗｗヴィヤーンタースｗｗｗｗｗワース フェスツｗｗｗｗｗｗｗルオルｗｗｗｗｗ
    プローイユクｗｗｗｗｗｗｗダルフェ スォーイヴォーｗｗｗｗｗスウェンネｗｗｗｗヤットゥ ヴ ヒェンヴガｒジョｊゴアｊガオガオッガｗｗｗじゃｇｊｊ
  </p>
, () => true)
  
const app = document.querySelector('#app')

const time0 = performance.now()

for(let i = 0; i < 10000; i++) {
    ReactDOM.render(
        <ConflictSFC />,
        app
    )
    ReactDOM.unmountComponentAtNode(app)
}

const time1 = performance.now()

console.log(`With Stateless Functional Component -> ${time1 - time0} milliseconds.`)

const time2 = performance.now()

for(let j = 0; j < 10000; j++) {
    ReactDOM.render(
        <ConflictMemo />,
        app
    )
    ReactDOM.unmountComponentAtNode(app)
}

const time3 = performance.now()

console.log(`With React.memo -> ${time3 - time2} milliseconds.`)
```

SFC Component -> 323.2399999978952 milliseconds.  
React.memo    -> 206.68999999179505 milliseconds.  

何度かやりましたがmemo > SFCの関係は変わりませんでした。  
逆に開発環境では何度やってもSFCのほうが早い様子・・・

### 結論：本番ではReact.memoしたほうが早い

#### 2. propsがshallow equalで対応しきれる場合
[shallow equal](https://efcl.info/2017/11/30/shallow-equal/)はその名の通り浅い比較を指します。  
具体的には、オブジェクトの**1段階目**のプロパティの比較を行うものです。

```ts
type Props = {
  name: string
  age: number
}

// below two components works same
const MyLovelyCat = React.memo(({ name, age } : Props) => (
  <>
    <h1>My Lovely Cat {'<3'}</h1>
    <p>Name: {name}</p>
    <p>Age: {age}</p>
  </>
), (prevProps, nextProps) => {
  // shallow equalの実態
  // オブジェクトの一段階目のプロパティのみ比較を行う
  const keys = Object.keys(prevProps)
  
  for (const key of keys) {
    if(prevProps[key] !== nextProps[key]) {
      return false
    }
  }
  
  return true
})

const MyLovelyCaaaaaaaaat = React.memo(({ name, age } : Props) => (
  <>
    <h1>My Lovely Cat {'<3'}</h1>
    <p>Name: {name}</p>
    <p>Age: {age}</p>
  </>
))
```

JavaScriptに詳しい方ならご存知だと思うのですが、  
比較演算子でObjectを比較する際は**オブジェクトが同一のインスタンスであるか**を比較します。  
つまり**KeyValueが完全に一致していても別インスタンスでは比較演算子は同一でないと判断する**ということです。

```js
class Hoge {
  constructor(a, b) { this.a = a; this.b = b; }
}

console.log(new Hoge(1, 2) === new Hoge(1, 2)) // This will be false!!!
```

この時点で、shallow equalにネストしたプロパティを渡しても意図した結果にならない事が分かりました。  
ネストしたプロパティまで対応したい場合、deep equalの出番です。  
但し、shallow equalはdeep equalより高速であることを忘れないでください。

#### 3. propsがdeep equalで対応する必要がある場合
ネストしたオブジェクトをpropsに持つ場合にはdeep equalの出番です。  
再帰的にオブジェクトのプロパティを比較します。  
但し、ここで紹介する方法の中で最も遅くなることを留意してください。  
もし変化しうるプロパティを分かりきっている場合は、**4. 変化しうるプロパティが特定できている場合**の実装を優先してください。  
deep-equalライブラリはdeep-equal系ライブラリの中で最速を謳う[fast-deep-equal](https://github.com/epoberezkin/fast-deep-equal)を使用します。

```js
import equal from 'fast-deep-equal'

const ComponentWithDeepEqual = React.memo(props => (
// snip
), (prev, next) => equal(prev, next))
```

#### 4. 変化しうるプロパティが特定できている場合
もし**特定のプロパティが変化した時だけメモを更新する**のであれば  
自分で更新を判定する関数をカスタマイズして対応することがベストプラクティスになります。

```ts
type TheBrave = {
  name: string
  gender: string
  equips: Equips
  maxHealth: number
  currentHealth: number
  maxMagicPoint: number
  currentMagicPoint: number
}
```

例えばこんなプロパティを持つ勇者が居たとしましょう。  
そしてあなたは戦闘中に何度も更新されるコンポーネントを実装することになったとします。  
`Equips`がオブジェクトであろうので、deep equalが適切かと思いますが、  
冷静になって考えると戦闘中に変化するであろうプロパティは`currentHealth`と`currentMagicPoint`のみです。

```ts
const BattleComponent = React.memo(
  ({ theBrave }: { theBrave : TheBrave }) => {
    // snip
  },
  // p -> prevProps, n -> nextProps
  (p, n) => p.theBrave.currentHealth === n.theBrave.currentHealth &&
            p.theBrave.currentMagicPoint === n.theBrave.currentMagicPoint
)
```
必要なプロパティのみ比較しているので、shallow equalやdeep equalより高速なコンポーネントの完成です！

#### まとめ
a. propsがあるか？ Yes -> b. No -> 1.  
b. 特定のプロパティだけ気にすればいいか？ Yes -> 4. No -> c.  
c. ネストしたプロパティがあるか？ Yes -> 3. No -> 2.

チューニングに関する知見は正直あまりこれ！という決定版が見つからないのでとりあえず自分の持っている知見を全放出しました。  
あくまで一人が調べた結果でしかないので改善の余地があるかと思います。  
ﾌﾛﾝﾄｴﾝﾄﾞﾂﾖｲｵｼﾞｻﾝの議論の話題になれば・・・！

## react-virtualized

画面表示範囲外のDOMを描画しないことにより、DOMの再レンダコストを下げるライブラリです。  
DOMの再レンダが頻繁に行われるアプリケーション（証券系、カレンダー、スプレッドシートetc...）で効果を発揮します。

詳細は[githubのレポジトリ](https://github.com/bvaughn/react-virtualized)を参照ください。

## Vanilla JS

- インライン展開

関数の呼び出しに変えて、関数側のコードを関数の呼び出し側に展開する手法です。  
本来はコンパイラの最適化の手法ですが、JavaScriptにおいても効果を発揮します。

下記コードをブラウザのコンソール上で実行してみてください。

```js
function powManyTimes (x) { x * x * x * x * x * x * x * x * x * x * x * x * x  }

const time0 = performance.now()

for (let i = 0; i < 100000; i++) {
  powManyTimes(i)
}

const time1 = performance.now()

console.log(`Before Inline Expansion : Time -> ${time1 - time0} milliseconds.`)

const time2 = performance.now()

for (let j = 0; j < 100000; j++) {
  j * j * j * j * j * j * j * j * j * j * j * j * j
}

const time3 = performance.now()

console.log(`After Inline Expansion : Time -> ${time3 - time2} milliseconds.`)
```
