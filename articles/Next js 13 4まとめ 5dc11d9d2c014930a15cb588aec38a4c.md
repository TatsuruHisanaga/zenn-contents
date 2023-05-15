# Next.js 13.4まとめ

ステータス: アイデア
作成者: 久永竜琉
種別: 公式ドキュメント
言語: Next.js
published: false

[Next.js 13.4](https://nextjs.org/blog/next-13-4)

[Next.js Cacheのアツさをシェアしたい(App Router)](https://zenn.dev/sumiren/articles/664c86a28ec573)

# **Next.js 13.4**

Friday, May 5th 2023

Next.js 13.4は基盤となるリリースであり、App Routerの安定性を確認します:

- App Router (安定版):
    - React Server Components
    - ネストされたルートとレイアウト
    - データ取得の簡素化
    - ストリーミングとサスペンス
    - 組み込みSEOサポート
- Turbopack (ベータ版): ローカル開発サーバーがより高速かつ安定性が向上
- Server Actions (アルファ版): クライアントのJavaScriptをゼロにしてサーバー上でデータを変更

Next.js 13のリリースから6ヶ月間、我々は必要以上の破壊的変更を避けつつ、段階的に採用可能な方法でNext.jsの未来の基盤を構築することに焦点を当ててきました。

今日、13.4のリリースにより、プロダクションでApp Routerを採用することが可能となりました。

```
npm i next@latest react@latest react-dom@latest eslint-config-next@latest
```

Next.js App Router
私たちは2016年にNext.jsをリリースし、よりダイナミックでパーソナライズされたグローバルなウェブを作成する目的で、Reactアプリケーションをサーバーレンダリングする簡単な方法を提供しました。

[最初の発表投稿](https://vercel.com/blog/next)では、Next.jsの設計原則をいくつか共有しました:

- セットアップゼロ。ファイルシステムをAPIとして使用
- JavaScriptのみ。全てが関数
- 自動的なサーバーレンダリングとコード分割
- データの取得は開発者次第

Next.jsは今や6年目です。当初の設計原則はそのまま残っており、Next.jsがより多くの開発者や企業に採用されるにつれて、これらの原則をより良く実現するためのフレームワークの基盤的なアップグレードに取り組んできました。

私たちはNext.jsの次世代版に取り組んでおり、今日の`13.4`により、この次世代版は安定して採用可能となりました。この投稿では、App Routerのための設計決定と選択について詳しく説明します。

## ゼロセットアップ、ファイルシステムをAPIとして使用

[ファイルシステムベースのルーティング](https://nextjs.org/docs/app/building-your-application/routing)はNext.jsのコア機能でした。当初の投稿では、1つのReactコンポーネントからルートを作成するこの例を示しました:

```
// ページルーター
// pages/about.js

import React from 'react';
export default () => <h1>私たちについて</h1>;

```

追加で設定する必要は何もありませんでした。ファイルをpages/内にドロップすれば、Next.jsのルーターが残りの部分を処理します。私たちは依然としてこのルーティングのシンプルさを愛しています。しかし、フレームワークの使用が増えるにつれて、開発者がそれを使って構築したいインターフェースの種類も増えました。

開発者たちはレイアウトの定義、UIのレイアウトとしてのネスティング、ローディングとエラー状態の定義についてより多くの柔軟性を持つための改善されたサポートを求めていました。これは既存のNext.jsルーターに後付けするのは容易なことではありませんでした。

フレームワークのすべての部分はルーターを中心に設計されている必要があります。ページ遷移、データフェッチング、キャッシュ、データの変更と再検証、ストリーミング、コンテンツのスタイリングなどです。

私たちは、ルーターをストリーミングと互換性があるようにし、レイアウトの強化されたサポートのためのこれらの要望を解決するために、ルーターの新しいバージョンを構築することにしました。

これは、私たちが[レイアウトのRFC](https://nextjs.org/blog/layouts-rfc)の最初のリリース後に着地した場所です。

```
// 新: App Router ✨
// app/layout.js
export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}

// app/page.js
export default function Page() {
  return <h1>Hello, Next.js!</h1>;
}

```

ここで重要なのは、あなたが見ることができるものよりも、見ることができないものです。この新しいルーター（`app/`ディレクトリを通じて徐々に採用することができます）は、全く異なるアーキテクチャを持っており、[React Server Components](https://nextjs.org/docs/getting-started/react-essentials)と[Suspense](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming)の基盤の上に構築されています。

この基盤により、我々は初めてReactのプリミティブを拡張するために開発されたNext.js特有のAPIを削除することができました。例えば、グローバル共有レイアウトをカスタマイズするためにカスタム`_app`ファイルを使用する必要はもうありません：

```
// ページルーター
// pages/_app.js

// この"global layout"は全てのルートをラップします。他のレイアウトコンポーネントを
// 組み合わせる方法はありませんし、このファイルからグローバルなデータをフェッチすることはできません。
export default function MyApp({ Component, pageProps }) {
  return <Component {...pageProps} />;
}

```

ページルーターでは、レイアウトを組み合わせることができず、データフェッチングをコンポーネントと同じ場所に配置することができませんでした。新しいApp Routerでは、これがサポートされています。

```
// 新: App Router ✨
// app/layout.js
//
// ルートレイアウトはアプリケーション全体で共有されます
export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );

// app/dashboard/layout.js
//
// レイアウトはネストや組み合わせが可能です
export default function DashboardLayout({ children }) {
  return (
    <section>
      <h1>Dashboard</h1>
      {children}
    </section>
  );
}

```

ページルーターでは、`_document`を使用してサーバーからの初期ペイロードをカスタマイズしていました。

```
// ページルーター
// pages/_document.js

// このファイルを使用すると、サーバーリクエストの<html>と<body>タグをカスタマイズできますが、
// HTML要素を書くのではなく、フレームワーク固有の機能が追加されます。
import { Html, Head, Main, NextScript } from 'next/document';

export default function Document() {
  return (
    <Html>
      <Head />
      <body>
        <Main />
        <NextScript />
      </body>
    </Html>
  );
}

```

App Routerでは、Next.jsから`<Html>`、`<Head>`、`<Body>`をインポートする必要はもうありません。代わりに、Reactを使用します。

```
// 新: App Router ✨
// app/layout.js
//
// ルートレイアウトはアプリケーション全体で共有されます
export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}

```

新しいファイルシステムルーターを構築する機会は、ルーティングシステムに関連する他の多くの機能要求に対応するための適切なタイミングでもありました。例えば：

-以前は、グローバルスタイルシートを`_external npm packages`（コンポーネントライブラリなど）から`_app.js`にインポートすることしかできませんでした。これは、開発者体験としては理想的ではありませんでした。App Routerでは、任意のCSSファイルを任意のコンポーネントにインポート（および配置）することができます。
-以前は、Next.jsでサーバーサイドレンダリングを選択する（`getServerSideProps`を通じて）と、全ページがハイドレートされるまでアプリケーションとの対話がブロックされてしまいました。App Routerでは、React Suspenseと深く統合されたアーキテクチャにリファクタリングしています。これにより、ページの一部を選択的にハイドレートすることができ、UIの他のコンポーネントがインタラクティブであることをブロックすることなく、コンテンツをサーバーから即座にストリームできます。これにより、ページの読み込み性能を向上させることができます。
[ルーター](https://nextjs.org/docs/app/building-your-application/routing)は、Next.jsを機能させるための核となるものです。しかし、ルーターそのものが重要なのではなく、ルーターがフレームワークの残りの部分、たとえば[データフェッチ](https://nextjs.org/docs/app/building-your-application/data-fetching)をどのように統合しているかが重要です。

## "JavaScriptだけ。全てが関数"

Next.jsとReactの開発者は、JavaScriptとTypeScriptのコードを書き、アプリケーションのコンポーネントをまとめて使用したいと考えています。私たちの最初の投稿から引用します：

```
import React from 'react';
import Head from 'next/head';

export default () => (
  <div>
    <Head>
      <meta name="viewport" content="width=device-width, initial-scale=1" />
    </Head>
    <h1>Hi. I'm mobile-ready!</h1>
  </div>
);

```

このコンポーネントは、アプリケーション内のどこでも再利用して組み合わせることができるロジックをカプセル化します。ファイルシステムルーティングと組み合わせると、JavaScriptとHTMLを書く感覚でReactアプリケーションを作り始める簡単な方法が得られました。

たとえば、データをフェッチしたい場合、Next.jsの最初のバージョンは次のようになります：

```
import React from 'react';
import 'isomorphic-fetch';

export default class extends React.Component {
  static async getInitialProps() {
    const res = await fetch('<https://api.company.com/user/123>');
    const data = await res.json();
    return { username: data.profile.username };
  }
}

```

~~Next.jsの将来のバージョンでは、isomorphic-fetchやnode-fetchをインポートする必要がなく、クライアントとサーバーの両方でWeb fetch APIを使用できるように、fetchをポリフィルするDX改善を追加しました。~~

採用が拡大し、フレームワークが成熟するにつれて、データフェッチングの新しいパターンを探求しました。

`getInitialProps`はサーバーとクライアントの両方で実行されました。このAPIはReactコンポーネントを拡張し、`Promise`を作成して結果をコンポーネントのpropsに転送することを可能にしました。

`getInitialProps`は今でも動作しますが、その後、顧客のフィードバックに基づいて次世代のデータフェッチングAPI、`getServerSideProps`と`getStaticProps`に向けて進化しました。

```
// ルートの静的バージョンを生成する
export async function getStaticProps(context) {
  return { props: {} };
}
// または、ルートを動的にサーバーサイドレンダリングする
export async function getServerSideProps(context) {
  return { props: {} };
}

```

これらのAPIは、コードがクライアント側かサーバー側のどちらで実行されているかをより明確にし、Next.jsアプリケーションを[自動的に静的に最適化](https://nextjs.org/docs/pages/building-your-application/rendering/automatic-static-optimization)することを可能にしました。さらに、[静的エクスポート](https://nextjs.org/docs/app/building-your-application/deploying/static-exports)を可能にし、サーバーをサポートしていない場所（例：AWS S3バケット）にNext.jsをデプロイすることができました。

しかし、これは「ただのJavaScript」ではなく、私たちは当初のデザイン原則により密接に従いたいと考えていました。

Next.jsが作成されて以来、私たちはMetaのReactコアチームと緊密に協力して、Reactの基本要素の上にフレームワークの機能を構築してきました。私たちのパートナーシップと、Reactコアチームによる数年間の研究と開発の組み合わせが、Next.jsがReactアーキテクチャの最新バージョン、特に[サーバーコンポーネント](https://nextjs.org/docs/getting-started/react-essentials)を含む形で目標を達成する機会を生み出しました。

App Routerでは、おなじみのasyncとawaitの構文を使用して[データをフェッチ](https://nextjs.org/docs/app/building-your-application/data-fetching)します。新たに学ぶべきAPIはありません。デフォルトでは、すべてのコンポーネントはReact Server Componentsであるため、データフェッチングはサーバー上で安全に行われます。例えば：

```
// app/page.js

export default async function Page() {
  const res = await fetch('<https://api.example.com/>...');
  // 返り値は*シリアライズされません*
  // Date、Map、Setなどを使用できます。
  const data = res.json();

  return '...';
}

```

これは極めて重要で、"データの取得は開発者次第"という原則が実現されています。あなたはデータを取得し、任意のコンポーネントを組み合わせることができます。そしてそれは、自社のコンポーネントだけでなく、Server Componentsと統合されてサーバー上で完全に動作するように設計された[Twitterの埋め込み](https://github.com/vercel-labs/react-tweet)`react-tweet`のような、Server Componentsのエコシステム内の任意のコンポーネントも対象となります。

```
// app/page.js

import { Tweet } from 'react-tweet';

export default async function Page() {
  return <Tweet id="790942692909916160" />;
}

```

ルーターが[React Suspense](https://react.dev/reference/react/Suspense)と統合されているため、コンテンツの一部がロード中である間にフォールバックコンテンツをより流動的に表示し、必要に応じて徐々にコンテンツを表示することができます。

```
// app/page.js

import { Suspense } from 'react';
import { PostFeed, Weather } from './components';

export default function Page() {
  return (
    <section>
      <Suspense fallback={<p>Loading feed...</p>}>
        <PostFeed />
      </Suspense>
      <Suspense fallback={<p>Loading weather...</p>}>
        <Weather />
      </Suspense>
    </section>
  );
}

```

さらに、ルーターはページのナビゲーションを[トランジション](https://react.dev/reference/react/useTransition)としてマークし、ルートのトランジションを中断可能にします。

## 自動的なサーバーレンダリングとコード分割

Next.jsを作成したとき、開発者がWebpack、Babel、およびその他のツールを手動で設定し、Reactアプリケーションを稼働させることはまだ一般的でした。サーバーレンダリングやコード分割などのさらなる最適化を追加することは、カスタムソリューションではしばしば実装されていませんでした。Next.jsおよび他のReactフレームワークは、これらのベストプラクティスを実装し、強制するための抽象化レイヤーを作成しました。

ルートベースのコード分割は、`pages/`ディレクトリ内の各ファイルがそれぞれのJavaScriptバンドルにコード分割されることを意味し、ファイルシステムを軽減し、初期ページロードのパフォーマンスを向上させるのに役立ちました。

これは、Next.jsのサーバーレンダリングアプリケーションとシングルページアプリケーションの両方にとって有益でした。後者はしばしばアプリケーションの起動時に単一の大きなJavaScriptバンドルをロードします。ただし、コンポーネントレベルのコード分割を実装するためには、開発者は`next/dynamic`を使用してコンポーネントを動的にインポートする必要がありました。

```
// app/page.tsx

import dynamic from 'next/dynamic';

const DynamicHeader = dynamic(() => import('../components/header'), {
  loading: () => <p>Loading...</p>,
});

export default function Home() {
  return <DynamicHeader />;
}

```

App Routerを使用すると、Server ComponentsはブラウザのJavaScriptバンドルに含まれません。[Client components](https://nextjs.org/docs/getting-started/react-essentials#client-components)はデフォルトで自動的にコード分割されます（Next.jsのwebpackまたはTurbopackを使用）。さらに、ルーターアーキテクチャ全体がストリーミングとSuspenseを有効にしているため、サーバーからクライアントへのUIの一部を逐次送信することができます。

例えば、条件付きロジックで全体のコードパスをコード分割することができます。この例では、ログアウトしたユーザーのクライアントサイドJavaScriptをロードする必要はありません。

```
// app/layout.tsx

import { getUser } from './auth';
import { Dashboard, Landing } from './components';

export default async function Layout() {
  const isLoggedIn = await getUser();
  return isLoggedIn ? <Dashboard /> : <
```

## Turbopack（ベータ版）

[Turbopack](https://turbo.build/pack)は、新たに試験的に導入し、Next.jsを通じて安定化を図っている私たちの新しいバンドラーです。Turbopackを使用すると、Next.jsアプリケーションの開発（`next dev --turbo`を使用）や、間もなく本番ビルド（`next build --turbo`を使用）のスピードが向上します。

Next.js 13のアルファリリース以降、バグ修正や欠落している機能のサポート追加を行なってきた結果、採用率は着実に上昇しています。私たちは、[Vercel.com](http://vercel.com/)や多数のVercelの顧客が運用する大規模なNext.jsウェブサイトでTurbopackをドッグフーディングし、フィードバックを収集し、安定性を向上させてきました。私たちのチームへのバグ報告に協力していただいたコミュニティの皆さんに感謝いたします。

今から6ヶ月後、私たちはベータフェーズに進む準備が整いました。

TurbopackはまだwebpackやNext.jsと完全な機能の互換性を持っていません。私たちは[この問題](https://github.com/vercel/next.js/issues/49174)でその機能のサポートを追跡しています。ただし、大半のユースケースは現在サポートされているはずです。このベータ版の目標は、採用率の増加から発生する残りのバグに対応し、将来のバージョンでの安定性を準備することです。

Turbopackのインクリメンタルエンジンとキャッシングレイヤーを改善するための投資は、ローカルの開発速度だけでなく、間もなく本番ビルドの速度も向上させます。インスタントビルドが可能な`next build --turbo`を実行できる将来のNext.jsバージョンをお楽しみに。今日からNext.js 13.4で`next dev --turbo`を使って[Turbopack](https://nextjs.org/docs/architecture/turbopack)ベータ版を試してみてください。

Server Actions（アルファ版）
Reactエコシステムは、フォーム、フォームステートの管理、データのキャッシングと再検証に関するアイデアの革新と探求を見てきました。時間の経過とともに、Reactはこれらのパターンについてより意見を持つようになりました。例えば、フォームステートには「[非制御コンポーネント](https://react.dev/learn/sharing-state-between-components#controlled-and-uncontrolled-components)」を推奨しています。

現在のエコシステムの解決策は、再利用可能なクライアントサイドのソリューションか、フレームワークに組み込まれたプリミティブのどちらかでした。しかし、これまでサーバーの変更とデータプリミティブを組み合わせる方法はありませんでした。Reactチームは、[変更に対する初のソリューションを開発してきました。](https://react.dev/blog/2023/03/22/react-labs-what-we-have-been-working-on-march-2023)

私たちは、Next.jsでの実験的なServer Actionsのサポートを発表することに興奮しています。これにより、APIレイヤーを介さずにサーバー上のデータを変更し、関数を直接呼び出すことが可能になります。

```
// app/post/[id]/page.tsx (Server Component)

import kv from './kv';

export default function Page({ params }) {
  async function increment() {
    'use server';
    await kv.incr(`post:id:${params.id}`);
  }

  return (
    <form action={increment}>
      <button type="submit">Like</button>
    </form>
  );
}

```

Server Actionsを使用すると、強力なサーバーファーストのデータ変更、クライアントサイドのJavaScriptの減少、そして段階的に強化されたフォームが利用できます。

```
// app/dashboard/posts/page.tsx (Server Component)

import db from './db';
import { redirect } from 'next/navigation';

async function create(formData: FormData) {
  'use server';
  const post = await db.post.insert({
    title: formData.get('title'),
    content: formData.get('content'),
  });
  redirect(`/blog/${post.slug}`);
}

export default function Page() {
  return (
    <form action={create}>
      <input type="text" name="title" />
      <textarea name="content" />
      <button type="submit">Submit</button>
    </form>
  );
}

```

Next.jsのServer Actionsは、データライフサイクルの他の部分、Next.js Cache、Incremental Static Regeneration（ISR）、およびクライアントルーターと深く統合するように設計されています。

新しいAPIであるrevalidatePathとrevalidateTagを通じてデータを再検証することで、変更、ページの再レンダリング、リダイレクトが一回のネットワークラウンドトリップで行われ、アップストリームプロバイダが遅い場合でもクライアント上に正しいデータが表示されることが確保されます。

```
// app/dashboard/posts/page.tsx (Server Component)

import db from './db';
import { revalidateTag } from 'next/cache';

async function update(formData: FormData) {
  'use server';
  await db.post.update({
    title: formData.get('title'),
  });
  revalidateTag('posts');
}

export default async function Page() {
  const res = await fetch('https://...', { next: { tags: ['posts'] } });
  const data = await res.json();
  // ...
}

```

Server Actionsは組み合わせ可能に設計されています。Reactコミュニティの誰もがServer Actionsを構築し、公開し、エコシステム内で配布することができます。Server Componentsと同様に、クライアントとサーバーの両方で利用可能な組み合わせ可能な基本要素の新時代にわくわくしています。

[Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions)は、Next.js 13.4で今日からアルファ版で利用可能です。Server Actionsを使用することを選択すると、Next.jsはReactの実験的リリースチャンネルを使用します。

```
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    serverActions: true,
  },
};

module.exports = nextConfig;

```

その他の改善点

- [ドラフトモード](https://nextjs.org/docs/app/building-your-application/configuring/draft-mode)：ヘッドレスCMSからドラフトコンテンツをフェッチしてレンダリングします。ドラフトモードはページとアプリの両方で動作します。既存のPreview Mode APIを強化し、簡素化しましたが、これはページ用に引き続き動作します。Preview Modeはアプリでは動作しません - ドラフトモードを使用してください。

よくある質問
App Routerの安定性とは何を意味しますか？
今日、App Routerを安定したものとしてマークすることは、私たちの仕事が終わったことを意味するわけではありません。安定性とは、App Routerのコアが本番環境に適しており、私たち自身の内部テストと多くのNext.jsの早期採用者によって検証されたことを意味します。

私たちは今後もさらなる最適化を行いたいと考えています。これには、Server Actionsが完全に安定することも含まれます。私たちがコアの安定性に向けて推進することが重要だったのは、コミュニティに対して、今日からどこから学び、アプリケーションを構築すべきか明確にするためです。

App Routerは、Reactのcanaryチャンネル上に構築されており、Server Componentsのような機能を採用するためのフレームワークが今や準備できています。詳細はこちら。[https://react.dev/blog/2023/05/03/react-canaries](https://react.dev/blog/2023/05/03/react-canaries)

これはNext.js betaドキュメントに何を意味しますか？
今日から、新しいアプリケーションの構築にはApp Routerを使用することを推奨します。App Routerを説明し、一から書き直されたNext.js betaドキュメンテーションは、現在、[安定したNext.jsドキュメンテーション](https://nextjs.org/docs)に戻されました。現在では、App RouterとPages Routerの間を簡単に切り替えることができます。

[App Routerの段階的な導入ガイドを](https://nextjs.org/docs/app/building-your-application/upgrading/app-router-migration)読むことを推奨します。これにより、App Routerの導入方法を学ぶことができます。

Pages Routerはなくなるのですか？
いいえ。私たちは、pages/の開発をサポートし続けることを約束しています。これには、バグ修正、改善、セキュリティパッチが含まれ、今後も複数のメジャーバージョンで続けられます。開発者が準備ができたときに、App Routerを段階的に導入できるようにするためです。

pages/とapp/を一緒に本番環境で使用することは、サポートされており、推奨されています。App Routerは、[ルートごと](https://nextjs.org/docs/app/building-your-application/upgrading/app-router-migration)に導入することができます。

これは、Server Componentsが「完成」したという意味ですか？
Next.jsは、Server Componentsを含むReactアーキテクチャを採用するフレームワークの一つです。App Routerで提供される体験が、他のフレームワーク（あるいは新しいフレームワーク）にこのアーキテクチャを使用することを検討するきっかけとなることを願っています。

無限スクロールのようなパターンはまだこのエコシステムで定義されていません。現在では、エコシステムが成長し、ライブラリが作成または更新される間、これらのパターンについてはクライアント側のソリューションを使用することを推奨しています。

## コミュニティ

Next.jsは、2,600人以上の個々の開発者、GoogleやMetaのような業界パートナー、そして私たちVercelのコアチームの共同作業の結果です。[GitHub](https://github.com/vercel/next.js/discussions)の[ディスカッション](https://github.com/vercel/next.js/discussions)、[Reddit](https://www.reddit.com/r/nextjs/)、[Discord](https://nextjs.org/discord)でコミュニティに参加してください。

このリリースは以下の皆様により提供されました：

Next.jsチーム: Andrew, Balazs, Jan, Jiachi, Jimmy, JJ, Josh, Sebastian, Shu, Steven, Tim, and Wyatt.
Turbopackチーム: Alex, Donny, Justin, Leah, Maia, OJ, Tobias, and Will.
そして貢献者たち: @shuding, @huozhi, @wyattfry, @styfle, @sreetamdas, @afonsojramos, @timneutkens, @alexkirsz, @chriswdmr, @jankaifer, @pn-code, @kdy1, @sokra, @kwonoj, @martin-wahlberg, @Kikobeats, @JTaylor0196, @sebmarkbage, @ijjk, @gnoff, @jridgewell, @sagarpreet-xflowpay, @balazsorban44, @cprussin, @ForsakenHarmony, @li-jia-nan, @dciug, @albertothedev, @DuCanhGH, @feedthejim, @patrick91, @padmaia, @sophiebits, @eps1lon, @reconbot, @acdlite, @cjmling, @nabsul, @motopods, @hanneslund, @tunamagur0, @devknoll, @apeltop, @maranomynet, @y-tsubuku, @EndangeredMassa, @ykzts, @AviAvinav, @adilansari, @wyattjoh, @charkour, @delbaoliveira, @agadzik, @Just-Moh-it, @rodrigofeijao, @leerob, @juliusmarminge, @koba04, @Phiction, @jessewarren-aa, @ryo-manba, @Yovach, and @dylanjha.