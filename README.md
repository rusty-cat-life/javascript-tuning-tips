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
執筆中

### Vanilla JS

- インライン展開
