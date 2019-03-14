# javascript-tuning-tips

### React

#### React.memoとコンポーネント単位でのチューニング

React.memoはコンポーネントを[メモ化](https://ja.wikipedia.org/wiki/%E3%83%A1%E3%83%A2%E5%8C%96)し、  
ウォッチしている変数に変更があったときにメモ化していたコンポーネントを破棄し、  
再レンダ、再びメモ化します。

第1引数にコンポーネントを渡し、第2引数に**Propsが同一であるかを判断する関数**を渡します。  
shouldComponentUpdateでは再レンダを行う際にtrueを返すが、  
memoでは再レンダを**行わない**際にtrueを返すので注意。  
第2引数に何も渡さなかった場合はPureComponent(shallowEqual)と同等の比較によりメモの更新判断がなされます。

PureComponentとshouldComponentUpdateがFunctional Componentになる上で合体したという理解で大体合ってます。

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

慣れてしまえばだいぶすっきりした記法ですね。

ここからが本題で、memoを利用しどうやってチューニングをしていくかの話になります。  
私はすべてのコンポーネントに同一に適応できるチューニングTipsというのはないと考えていて、  
**泥臭く各コンポーネントのpropに応じて最善手を選択する**のが現実的なお話になると思います。  
銀の弾丸なんてなかった。

ただし、最善手の選択アルゴリズムは構築できるのはずなのでそれを書こうと思います。

1. propsがない場合
2. propsがshallow equalで対応しきれる場合
3. propsがdeep equalする必要がある場合

**※ 以下のパフォーマンス計測は、create-rect-appで作成したプロジェクトを`yarn build`したもので計測しています。**  
　 **よって、開発環境での結果と異なる場合があります**

#### 1.propsがない場合
この場合は`() => true`なmemoか、普通のSFCが早いかの比較になります。計測しましょう。

```jsimport React from 'react'
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

但し、開発環境では何度やってもSFCのほうが早い様子・・・

### 結論：React.memoしたほうが早い

__以下wip__

[shallow-equal(浅い比較)](https://efcl.info/2017/11/30/shallow-equal/)はその名の通り浅い比較（オブジェクトの1段階のプロパティのみ比較）なので、 

よって、**deep-equal**を使用します。  
deep-equalはshallow-equalより低速であるというデメリットはありますが、  
再帰的にネストしたプロパティまで比較してくれるのが特徴です。  
deep-equalライブラリはdeep-equal系ライブラリの中で最速を謳っている[fast-deep-equal](https://github.com/epoberezkin/fast-deep-equal)を選定しました。

```javascript
import equal from 'fast-deep-equal'

/* snip */
shouldComponentUpdate(nextProps, nextState) {
  return !equal(this.props, nextProps) || !equal(this.state, nextState)
}
```

- react-virtualized

画面表示範囲外のDOMを描画しないことにより、DOMの再レンダコストを下げるライブラリです。  
DOMの再レンダが頻繁に行われるアプリケーション（証券系、カレンダー、スプレッドシートetc...）で効果を発揮します。

詳細は[githubのレポジトリ](https://github.com/bvaughn/react-virtualized)を参照ください。

### Vanilla JS

- インライン展開

関数の呼び出しに変えて、関数側のコードを関数の呼び出し側に展開する手法です。  
本来はコンパイラの最適化の手法ですが、JavaScriptにおいても効果を発揮します。

下記コードをブラウザのコンソール上で実行してみてください。

```js
function powManyTimes (x) { return x * x * x * x * x * x * x * x * x * x * x * x * x  }

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
