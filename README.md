# javascript-tuning-tips

### React
- shouldComponentUpdate

Reactコンポーネントの再レンダを行うか意思決定をする関数です。  
boolを返します。

Reactコンポーネントの再レンダ処理が実行されるタイミングは

1. props, stateが更新された際
2. forceUpdateが呼び出された際

の2種類あります。

このうち、2のforceUpdate呼び出し時に関してはshouldComponentUpdateはスキップされるのですが、  
基本的にforceUpdateは極力使わない設計が求められるのでここでは解説しません。

shouldComponentUpdateはnextPropsとnextStateを引数に取ります。  
これらを現在のprops, stateと比較し差分が認められた場合にのみ  
再レンダリングを行うのがReactでのチューニングの基本です。

但し、JavaScriptのObjectの比較（比較演算子を利用したもの）に関しては、
**同一のインスタンス**であるかの比較となっているため、使用できません。  

対応としては2通りあり、

1. shallow-equalの使用
2. deep-equalの使用

です。

[shallow-equal(浅い比較)](https://efcl.info/2017/11/30/shallow-equal/)はその名の通り浅い比較（オブジェクトの1段階のプロパティのみ比較）なので、  
ネストしたプロパティまでは比較できていません。  
結果、shallow-equalとshouldComponentUpdateの組み合わせでは上手くいかないパターンが発生しえます。  
（このあたりはオブジェクトの設計のルール決めで2段階以上のprops, stateは作成しないという方針でもカバーできますが、  
技術者のレベルが玉石混淆であること、propsがバケツリレーになりがちな観点から現実的には難しいと感じています。）

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

関数を呼び出す際に、関数側のコードを関数の呼び出し側に展開する手法です。  
本来はコンパイラの最適化の手法ですが、JavaScriptにおいても効果を発揮します。

下記コードをブラウザのコンソール上で実行してみてください。

```js
const time0 = performance.now()

function powManyTimes (x) { return x * x * x * x * x * x * x * x * x * x * x * x * x  }

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
