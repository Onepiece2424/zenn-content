---
title: "Cannot update a component while rendering ~.というwarningの原因と対処方法について"
emoji: "🗂"
type: "tech"
topics:
  - "react"
  - "redux"
  - "reduxform"
published: true
published_at: "2023-03-12 11:45"
---

### 背景
reactとredux-formを用いて、隠しフィールドからサーバ側へ値を送信するフォームを作成していたら、「Warning: Cannot update a component (`Connect(Form(Change Form))`) while rendering a different component (`Change Form`).  ( **警告: 別のコンポーネント (`Change Form`) のレンダリング中にコンポーネント (`Connect(Form(Change Form))`) を更新することはできません。**)  」というwarningが発生し、原因と解決方法を見つけたので解説いたします。

**warningの原因：compoentのレンダリング中に別のcomponentの状態を更新しようとしていたため**

**解決方法：useEffectを使用し、レンダリング後にstate更新用関数を呼び出しstateを更新させる**

### **今回warningが発生したコードと解説**

```jsx
import React, { useState } from 'react'
import { useDispatch } from 'react-redux';
import { reduxForm, Field, change } from 'redux-form'
import renderField from '../renderField'
import { Button } from '@material-ui/core';
import { validate } from '../../func/validate';
import ModalForm from '../modal/ModalForm';

const Changeform = ({ handleSubmit, pristine, submitting, reset }) => {

  // Modalフォームを表示・非表示にするためのフラグ
  const [flag, setFlag] = useState(false)
  const flagChange = () => {
    setFlag(prev => !prev)
  }

  const dispatch = useDispatch();

  // 隠しFieldに値を設定
  dispatch(change('changeform', 'hiddenvalue', '隠しFieldの値だよ〜。'));

  // チェックが変更されたら、'checkboxValue'フィールドの値を動的に変更する
  const onCheckboxClick = (e) => {
    const newValue = e.target.checked ? '今日も良い天気ですね！' : '';
    dispatch(change('changeform', 'checkboxValue', newValue));
  };

  return (
    <>
      <form onSubmit={handleSubmit}>
        <Field name="username" component={renderField} label="名前" />
        <br></br>
        <div>
        <label>チェックボックス（チェックすると設定された値が表示されます。）</label>
        <div>
          <input type="checkbox" onChange={onCheckboxClick} />
          <Field name="checkboxValue" component={renderField} type="text" />
        </div>
        <br></br>
        <Field name="hiddenvalue" component={renderField} type="hidden" />
        </div>
        <br></br>
        <Button onClick={flagChange}>Modal入力フォームの表示</Button>
        <br></br>
        { flag && <ModalForm />}
        <br></br>
        <Button color="secondary" variant="contained" type="submit">送信</Button>
        <Button color="secondary" variant="contained" disabled={pristine || submitting} onClick={reset}>
          クリア
        </Button>
      </form>
    </>
  )
}

export default reduxForm({
  form: 'changeform',
  validate
})(Changeform)
```

上記のwarningは、[公式](https://ja.reactjs.org/blog/2020/02/26/react-v16.13.0.html#warnings-for-some-updates-during-render)によると、compoentのレンダリング中に副作用である別のcompoentの状態を更新しようとした時に発生するようです。

つまり、raectで**はcompoentのレンダリング中に別のcomponentの状態を更新してはいけない**ようです。

今回の実装の場合、Changeform componentの中でstateを用いてボタンクリックによるModalFormというcomponentの表示・非表示の切り替えを行なっていたので、その時に使用したstate更新用関数 setFlag が原因なのではないかと考え、コメントアウトしたり、useEffect内でstate更新用関数 setFlagを実行してみましたが、上記のwarningは消えませんでした。

他の箇所でstateを更新している箇所がないか調べていると、下記の部分が原因でした。

下記の部分で　Changeform componentのレンダリング中にdispatch とredux-formのchangeメソッドによるField component の 値（隠しFieldの値だよ〜。のこと）を更新していたために今回のwarningが発生していたと考えられます。

```jsx
// 隠しフィールドに値を設定
dispatch(change('changeform', 'hiddenvalue', '隠しFieldの値だよ〜。'));

--- 省略 ---

<Field name="hiddenvalue" component={renderField} type="hidden" />
```

warning解決のため、[公式](https://ja.reactjs.org/blog/2020/02/26/react-v16.13.0.html#warnings-for-some-updates-during-render)を参考に**useEffectを用いてdispatchによるField component の 値の更新を　Changeform componentのレンダリング後に行う**ようにしてみました。

すると、「Cannot update a component ~」という  warnig は削除されました。

```jsx
// 隠しフィールドに値を設定
useEffect(() => {
  dispatch(change('changeform', 'hiddenvalue', '隠しFieldの値だよ〜。'));
}, [dispatch]);
```

### まとめ

今回のwarningにより、reactでは**compoentのレンダリング中に別のcomponentの状態を更新してはいけない**ということを初めて知りました。

普段どのタイミングでFieldに値を保持させるべきかあまり意識したことがなかったので、良い経験になりました。

今後はcomponentのレンダリング中に別のcomponentの状態を更新しようとした時は、useEffectを適切に使用していき対処していきたいです。