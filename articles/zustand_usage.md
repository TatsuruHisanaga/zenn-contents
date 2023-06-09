---
title: "状態管理ライブラリ ''Zustand'' をTypeScriptで使ってみよう"
emoji: "⚡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript", "React", "Zustand", "状態管理ライブラリ"]
published: true
---

## 参考
https://docs.pmnd.rs/zustand/guides/typescript



## はじめに


### Zustandとは
Zustandは、主にReactで利用される小さくてシンプルな**状態管理ライブラリ**です。Zustandを利用することによって、**状態管理に関連するコードの複雑さを大幅に解消できます**。また、HookベースのAPIを提供しているため、React Hooksに慣れている開発者にとっては直感的に使うことが可能です。


Zustandには、以下の主要な概念が含まれています。

1. **Store**

   ZustandのStoreは、アプリケーションの状態とその操作を提供します。

```tsx
type Store = {
  count: number;
  increase: () => void;
}
```

2. **State**

   ZustandのStoreは、状態をJavaScriptのオブジェクトとして保持します。この例では、`count`が状態を表現します。

```tsx
const useStore = create<Store>((set) => ({
  count: 0, 
increase: () => set((state) => ({ count: state.count + 1 })),
}));
```

3. **create**

   ZustandのStoreは、`create`関数を使用して作成します。

```tsx
const useStore = create<Store>((set) => ({
  count: 0,
  increase: () => set((state) => ({ count: state.count + 1 })),
}));
```

4. **set**

   `set`関数は、状態を更新するために使用されます。

```tsx
const useStore = create<Store>((set) => ({
  count: 0,
  increase: () => set((state) => ({ count: state.count + 1 })),
}));
```

5. **Actions**

   Actionsは、状態を操作するメソッドです。この例では、アクションは`increase`メソッドです。

```tsx
  increase: () => set((state) => ({ count: state.count + 1 }))
```

6. **Hooks**

   Zustandは、Reactとの統合を容易にするために、カスタムフックの形式でStoreを提供します。

```tsx
import React from 'react';

const Component = () => {
  const count = useStore((state) => state.count);
  const increase = useStore((state) => state.increase);

  return (
    <div>
      <div>{count}</div>
      <button onClick={increase}>Increase</button>
    </div>
  );
};

```

7. **Middleware**

   Zustandは、ミドルウェアをサポートしています。

```tsx
import create from 'zustand';
import { devtools } from 'zustand/middleware'

const useStore = create(devtools((set) => ({
  count: 0,
  increase: () => set((state) => ({ count: state.count + 1 })),
})));

```

8. **Async actions**

   Zustandは、非同期操作をサポートしています。

```tsx
const useStore

 = create<Store>((set) => ({
  count: 0,
  increase: async () => {
    const result = await fetchSomeData(); // some async operation
    set((state) => ({ count: state.count + result }));
  },
}));

```

これらの例は、Zustandがどのように動作するかを示しています。ただし、実際の使用方法はアプリケーションのニーズによります。

### この記事で取り扱う内容
この記事の主な内容は以下の通りです。なお、Zustandのインストールなど、初期設定についてこの記事では言及しておりませんので、各自ご確認ください。

##### 1. Zustandの基本的な使用法

TypeScriptでZustandを使用する際の基本的な使用法について説明します。

##### 2. 型推論の問題とその解決方法

TypeScriptでZustandを使う際に遭遇する可能性のある型推論の問題と、その解決方法について説明します。

##### 3. ミドルウェアの使用

Zustandでミドルウェアを使用する方法とその注意点について説明します。

##### 4. スライスパターン

大きな状態オブジェクトを、各々が独自の状態とアクションを持つ小さな「スライス」に分割するスライスパターンの利用方法について説明します。


### この記事の対象読者
この記事の対象読者は、Zustandを使用してステート管理を行いたいと考えているTypeScriptの学習者です。


## **Zustandの基本的な使用法**

TypeScriptでZustandを使用する際には、状態の型を指定するために`create<T>()(...)` の形式を使用します。この`<T>`の部分には、状態の型を表すインターフェイスや型が入ります。

:::message
**補足：インターフェイスとは**
インターフェイスは、**特定のオブジェクトが必ず持っていなければならないプロパティやメソッドの一覧を定義するための構造**です。

あるオブジェクトが特定のインターフェイスを実装していると宣言すると、そのオブジェクトがインターフェイスで定義されたすべてのプロパティとメソッドを持っていることが保証されます。これにより、プログラムの**安全性が向上**します。
:::

以下に、ZustandとTypeScriptを使った基本的なコード例を示します:


```tsx
import { create } from 'zustand'

interface BearState {
  bears: number
  increase: (by: number) => void
}

// 以下の部分で`create`関数に`BearState`を型として渡しています
const useBearStore = create<BearState>()((set) => ({
  bears: 0,
  increase: (by) => set((state) => ({ bears: state.bears + by })),
}))

```


`<BearState>`という記述により、このZustandストアが`BearState`という型の状態を管理することを指定しています。このため、このストアの状態やその更新関数を使用する際には、TypeScriptが型のチェックを行ってくれます。

:::message
**補足：ストアとは**
ストアとは、アプリケーションの状態を保持してその変更を監視するためのものです。これは一般的にフロントエンドのアプリケーション開発でよく使われ、とくに大規模なアプリケーションや複数のコンポーネントが共有する状態を管理する際に有効です。
:::

### **▶ なぜ単純に初期状態から型を推論できないのか？**

`create(...)`ではなく`create<T>()(...)`ように記述する必要がある理由は、結論、TypeScriptのジェネリック型Tが「**型を返すとともに型を取得する**」という厄介な性質を持っているためです。

:::message
**補足：ジェネリック型とは**
ジェネリック型はプログラミング言語の機能で、型が決まっていない、または柔軟に型を変えることができるようにする機能です。TypeScriptにおいてもジェネリック型が提供されています。

ジェネリック型は、一般的に大文字のTを使って表現されますが、これはTypeの頭文字を取っているだけで、任意の文字を使うことができます。
:::

たとえば、以下に示すコードは、create関数の簡易版を示しています。

```tsx
declare const create: <T>(f: (get: () => T) => T) => T

const x = create((get) => ({
  foo: 0,
  bar: () => get(),
}))
// このコードにおいて、xの推論される型は"unknown"になります。
// 期待する推論結果は以下のインターフェイスXに従う型です：
// interface X {
//   foo: number,
//   bar: () => X
// }

```

このコードでは、create関数の引数fが```(get: () => T) => T```という形式をしています。これは、Tの型を"返す"とともに、getという関数を通してTの型を"取得する"ことを意味します。

しかし、**TypeScriptはTが具体的にどのような型であるのか、その情報がどこから得られるのかを推論することはできません**。これはTypeScriptが「卵が先か鶏が先か」というような問題につまづいてしまうことに起因します。

よって、TypeScriptはこの難問を解決できず、Tの型を不明（unknown）と判断してしまう訳です。

そのため、`create(...)`の代わりに`create<T>()(...)`を記述して開発者が型をあらかじめ指定することで、Zustandの状態管理オブジェクトの構造と、それがどのように使われるべきかをTypeScriptに伝えています。
## **ミドルウェアの適用**

TypeScriptを用いたミドルウェアの使用は直感的で、特別な設定をする必要はありません。

以下に`devtools`と`persist`という2つのミドルウェアを組み合わせた例を示します。

```tsx
import { create } from 'zustand'
import { devtools, persist } from 'zustand/middleware'

interface BearState {
  bears: number
  increase: (by: number) => void
}

const useBearStore = create<BearState>()(
  devtools(
    persist((set) => ({
      bears: 0,
      increase: (by) => set((state) => ({ bears: state.bears + by })),
    }))
  )
)

```

しかし、TypeScriptの型推論を最大限に活かすためには、create関数の直後にミドルウェアを適用するのがベストです。もし自分でカスタムミドルウェアを作成する場合、下記のようにより高度な型管理が必要となることに注意してください。

```tsx
import { create } from 'zustand'
import { devtools, persist } from 'zustand/middleware'

const myMiddlewares = (f) => devtools(persist(f))

interface BearState {
  bears: number
  increase: (by: number) => void
}

const useBearStore = create<BearState>()(
  myMiddlewares((set) => ({
    bears: 0,
    increase: (by) => set((state) => ({ bears: state.bears + by })),
  }))
)

```

さらに、devtoolsミドルウェアは、他のミドルウェアの後に適用すべきです。

たとえば、immerというミドルウェアを使用する際は、`immer(devtools(...))`の順序が推奨され、`devtools(immer(...))`の順序は避けるべきです。その理由は、もしdevtoolsが先に適用された場合、それらの変更が他のミドルウェアによって上書きされてしまう可能性があるからです。

そのため、devtoolsを最後に適用することで、その影響を避けることができます。

### **スライスパターン**

スライスパターンとは、**大きな状態オブジェクトを、各々が独自の状態とアクションを持つ小さな「スライス」に分割する方法**です。以下にその例を示します。

```tsx
import { create, StateCreator } from 'zustand'

// クマのスライスを定義
const createBearSlice: StateCreator = (set) => ({
  bears: 0,
  addBear: () => set((state) => ({ bears: state.bears + 1 })),
  eatFish: () => set((state) => ({ fishes: state.fishes - 1 })),
})

// 魚のスライスを定義
const createFishSlice: StateCreator = (set) => ({
  fishes: 0,
  addFish: () => set((state) => ({ fishes: state.fishes + 1 })),
})

// 全体の状態を作成
const useBoundStore = create((...a) => ({
  ...createBearSlice(...a),
  ...createFishSlice(...a),
}))
```

ここで、`createBearSlice`と`createFishSlice`が各スライスを生成します。`bears`と`fishes`は状態、`addBear`、`eatFish`、`addFish`はアクションを表します。

このスライスパターンにより、**アプリケーションの状態管理が容易になり、メンテナンス性や再利用性が向上**します。また、特定のスライスだけをテストすることも可能になります。

## 終わりに

本記事では、ZustandとTypeScriptを用いた状態管理について記述しました。Zustandの基本的な使用法、型推論の問題、ミドルウェアの適用、そしてスライスパターンについて触れています。

ZustandとTypeScriptを利用することで、アプリケーションの状態管理をより簡潔に、効率的に行うことができます。これにより、ソフトウェア開発全体の生産性が向上し、結果としてより高品質なプロダクトを生み出すことが可能となります。

より詳細な情報を知りたい場合は、下記の[公式ドキュメント](https://docs.pmnd.rs/zustand/guides/typescript#using-middlewares)をご参照ください。また、上記の内容に間違いや改善点がある場合は、ご遠慮なくお申し出ください。
https://docs.pmnd.rs/zustand/guides/typescript#using-middlewares


*Written with ChatGPT-4*