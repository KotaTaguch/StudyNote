# Next.js 完全ガイド（App Router対応）

Next.js 14/15とReact 19.2を使用したモダンなフロントエンド開発の完全リファレンス

## 目次

- [Next.jsとは](#nextjsとは)
- [レンダリング戦略](#レンダリング戦略)
- [ルーティング（App Router）](#ルーティングapp-router)
- [データフェッチング](#データフェッチング)
- [Server ComponentsとClient Components](#server-componentsとclient-components)
- [Server Actions](#server-actions)
- [キャッシング戦略](#キャッシング戦略)
- [最適化機能](#最適化機能)
- [API Routes](#api-routes)
- [認証とミドルウェア](#認証とミドルウェア)
- [デプロイメント](#デプロイメント)

---

## Next.jsとは

Next.jsは、React上に構築されたフルスタックWebアプリケーションフレームワークです。

### 主な特徴

✅ **複数のレンダリング戦略** - SSR、CSR、SSG、ISRをサポート  
✅ **ファイルベースルーティング** - 直感的なルーティングシステム  
✅ **Server Components** - サーバーサイドコンポーネント  
✅ **自動コード分割** - ページごとに最適化  
✅ **画像最適化** - 自動的な画像最適化  
✅ **API Routes** - バックエンドAPIの構築  
✅ **TypeScriptサポート** - 完全なTypeScript統合

---

## レンダリング戦略

Next.jsでは、ページやコンポーネントごとに異なるレンダリング戦略を選択できます。

### 1. SSR (Server-Side Rendering) - サーバーサイドレンダリング

リクエストごとにサーバーでHTMLを生成します。

#### 特徴
- ✅ SEOに最適
- ✅ 常に最新のデータ
- ✅ 初期表示が速い
- ❌ サーバー負荷が高い
- ❌ TTFBが遅い

#### 実装例

```typescript
// app/products/[id]/page.tsx
interface Product {
  id: string;
  name: string;
  price: number;
}

// Server Componentはデフォルトで動的レンダリング（SSR）
export default async function ProductPage({ 
  params 
}: { 
  params: { id: string } 
}) {
  // サーバーで毎回実行される
  const res = await fetch(`https://api.example.com/products/${params.id}`, {
    cache: 'no-store' // キャッシュしない = SSR
  });
  const product: Product = await res.json();

  return (
    <div>
      <h1>{product.name}</h1>
      <p>価格: ¥{product.price}</p>
      <p>生成時刻: {new Date().toLocaleString()}</p>
    </div>
  );
}
```

#### 動的データの取得

```typescript
// app/dashboard/page.tsx
export const dynamic = 'force-dynamic'; // SSRを強制

async function getUserData(userId: string) {
  const res = await fetch(`https://api.example.com/users/${userId}`, {
    cache: 'no-store'
  });
  return res.json();
}

export default async function DashboardPage() {
  const user = await getUserData('current-user-id');
  
  return (
    <div>
      <h1>ダッシュボード</h1>
      <p>こんにちは、{user.name}さん</p>
      <p>最終ログイン: {new Date().toLocaleString()}</p>
    </div>
  );
}
```

---

### 2. SSG (Static Site Generation) - 静的サイト生成

ビルド時にHTMLを生成します。

#### 特徴
- ✅ 最も高速
- ✅ CDNでの配信に最適
- ✅ サーバー負荷ゼロ
- ❌ ビルド時のデータのみ
- ❌ 更新にはリビルドが必要

#### 実装例

```typescript
// app/blog/[slug]/page.tsx
interface Post {
  slug: string;
  title: string;
  content: string;
  publishedAt: string;
}

// ビルド時に生成するパスを指定
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then(res => res.json());
  
  return posts.map((post: Post) => ({
    slug: post.slug,
  }));
}

// Server Componentでキャッシュを有効化 = SSG
export default async function BlogPost({ 
  params 
}: { 
  params: { slug: string } 
}) {
  const res = await fetch(`https://api.example.com/posts/${params.slug}`, {
    cache: 'force-cache' // キャッシュする = SSG
  });
  const post: Post = await res.json();

  return (
    <article>
      <h1>{post.title}</h1>
      <time>{new Date(post.publishedAt).toLocaleDateString()}</time>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  );
}
```

#### 静的ページの例

```typescript
// app/about/page.tsx
// デフォルトでキャッシュされる = SSG
export default function AboutPage() {
  return (
    <div>
      <h1>会社について</h1>
      <p>私たちは2020年に設立されました。</p>
      <p>このページはビルド時に生成されます。</p>
    </div>
  );
}
```

---

### 3. ISR (Incremental Static Regeneration) - 増分静的再生成

静的ページを定期的に再生成します。

#### 特徴
- ✅ SSGの速度
- ✅ 定期的な更新
- ✅ オンデマンド再生成
- ❌ 一時的に古いデータ表示
- ❌ 設定が複雑

#### 実装例

```typescript
// app/news/[id]/page.tsx
interface NewsArticle {
  id: string;
  title: string;
  content: string;
  updatedAt: string;
}

export default async function NewsPage({ 
  params 
}: { 
  params: { id: string } 
}) {
  // 60秒ごとに再検証
  const res = await fetch(`https://api.example.com/news/${params.id}`, {
    next: { revalidate: 60 }
  });
  const article: NewsArticle = await res.json();

  return (
    <article>
      <h1>{article.title}</h1>
      <time>更新: {new Date(article.updatedAt).toLocaleString()}</time>
      <div>{article.content}</div>
    </article>
  );
}

// 生成するパスを指定
export async function generateStaticParams() {
  const articles = await fetch('https://api.example.com/news').then(res => res.json());
  
  return articles.slice(0, 10).map((article: NewsArticle) => ({
    id: article.id,
  }));
}
```

#### オンデマンド再検証

```typescript
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from 'next/cache';
import { NextRequest } from 'next/server';

export async function POST(request: NextRequest) {
  const secret = request.nextUrl.searchParams.get('secret');
  
  // シークレットキーの検証
  if (secret !== process.env.REVALIDATE_SECRET) {
    return Response.json({ message: 'Invalid secret' }, { status: 401 });
  }

  const path = request.nextUrl.searchParams.get('path');
  
  if (path) {
    // 特定のパスを再検証
    revalidatePath(path);
    return Response.json({ revalidated: true, path });
  }

  return Response.json({ message: 'Missing path' }, { status: 400 });
}
```

#### タグベースの再検証

```typescript
// app/products/page.tsx
async function getProducts() {
  const res = await fetch('https://api.example.com/products', {
    next: { 
      revalidate: 3600, // 1時間
      tags: ['products'] // タグを付与
    }
  });
  return res.json();
}

export default async function ProductsPage() {
  const products = await getProducts();
  
  return (
    <div>
      {products.map((product: any) => (
        <div key={product.id}>{product.name}</div>
      ))}
    </div>
  );
}

// app/api/revalidate-products/route.ts
import { revalidateTag } from 'next/cache';

export async function POST() {
  // 'products'タグを持つすべてのデータを再検証
  revalidateTag('products');
  return Response.json({ revalidated: true });
}
```

---

### 4. CSR (Client-Side Rendering) - クライアントサイドレンダリング

ブラウザでJavaScriptによってレンダリングします。

#### 特徴
- ✅ インタラクティブ
- ✅ リアルタイムデータ
- ✅ サーバー負荷軽減
- ❌ SEOに不向き
- ❌ 初期表示が遅い
- ❌ JavaScript必須

#### 実装例

```typescript
// app/dashboard/client-stats.tsx
'use client'; // Client Componentとして宣言

import { useState, useEffect } from 'react';

interface Stats {
  views: number;
  likes: number;
  shares: number;
}

export default function ClientStats() {
  const [stats, setStats] = useState<Stats | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // クライアントサイドでデータ取得
    fetch('/api/stats')
      .then(res => res.json())
      .then(data => {
        setStats(data);
        setLoading(false);
      });
  }, []);

  if (loading) return <div>読み込み中...</div>;

  return (
    <div>
      <h2>統計情報</h2>
      <p>閲覧数: {stats?.views}</p>
      <p>いいね: {stats?.likes}</p>
      <p>シェア: {stats?.shares}</p>
    </div>
  );
}
```

#### リアルタイムデータの表示

```typescript
'use client';

import { useState, useEffect } from 'react';

export default function LivePriceTracker({ symbol }: { symbol: string }) {
  const [price, setPrice] = useState<number | null>(null);

  useEffect(() => {
    // WebSocketでリアルタイムデータを取得
    const ws = new WebSocket(`wss://api.example.com/prices/${symbol}`);
    
    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setPrice(data.price);
    };

    return () => ws.close();
  }, [symbol]);

  return (
    <div>
      <h2>{symbol}</h2>
      <p className="text-3xl font-bold">
        ¥{price?.toLocaleString() ?? '---'}
      </p>
      <p className="text-sm text-gray-500">リアルタイム価格</p>
    </div>
  );
}
```

---

### 5. ハイブリッドレンダリング

異なるレンダリング戦略を組み合わせます。

```typescript
// app/product/[id]/page.tsx
import ClientReviews from './client-reviews';
import RelatedProducts from './related-products';

interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
}

// サーバーコンポーネント（SSR/SSG）
export default async function ProductPage({ 
  params 
}: { 
  params: { id: string } 
}) {
  // SSGで製品情報を取得
  const product: Product = await fetch(
    `https://api.example.com/products/${params.id}`,
    { next: { revalidate: 3600 } }
  ).then(res => res.json());

  return (
    <div>
      {/* サーバーで生成される静的コンテンツ */}
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <p className="text-2xl">¥{product.price}</p>

      {/* クライアントサイドで動的に表示（CSR） */}
      <ClientReviews productId={product.id} />

      {/* サーバーで生成される関連商品（SSG） */}
      <RelatedProducts productId={product.id} />
    </div>
  );
}
```

```typescript
// app/product/[id]/client-reviews.tsx
'use client';

import { useState, useEffect } from 'react';

interface Review {
  id: string;
  author: string;
  rating: number;
  comment: string;
}

export default function ClientReviews({ productId }: { productId: string }) {
  const [reviews, setReviews] = useState<Review[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(`/api/reviews?productId=${productId}`)
      .then(res => res.json())
      .then(data => {
        setReviews(data);
        setLoading(false);
      });
  }, [productId]);

  if (loading) return <div>レビュー読み込み中...</div>;

  return (
    <div className="mt-8">
      <h2>カスタマーレビュー</h2>
      {reviews.map(review => (
        <div key={review.id} className="border p-4 mb-4">
          <div className="flex items-center">
            <span className="font-bold">{review.author}</span>
            <span className="ml-2">{'⭐'.repeat(review.rating)}</span>
          </div>
          <p className="mt-2">{review.comment}</p>
        </div>
      ))}
    </div>
  );
}
```

---

### レンダリング戦略の選択ガイド

| 戦略 | 使用ケース | 例 |
|------|-----------|-----|
| **SSR** | 動的データ、SEO重要 | ユーザーダッシュボード、検索結果 |
| **SSG** | 静的コンテンツ | ブログ、ドキュメント、LP |
| **ISR** | 定期更新コンテンツ | ニュース、商品一覧 |
| **CSR** | 高度にインタラクティブ | チャット、リアルタイムデータ |

---

## ルーティング（App Router）

Next.js 13以降の新しいルーティングシステム。

### 基本的なルーティング

```
app/
├── page.tsx                 # / (ホーム)
├── about/
│   └── page.tsx            # /about
├── blog/
│   ├── page.tsx            # /blog
│   └── [slug]/
│       └── page.tsx        # /blog/[slug]
└── products/
    ├── page.tsx            # /products
    └── [id]/
        ├── page.tsx        # /products/[id]
        └── reviews/
            └── page.tsx    # /products/[id]/reviews
```

### ページの作成

```typescript
// app/page.tsx (ホームページ)
export default function HomePage() {
  return (
    <div>
      <h1>ホームページ</h1>
      <p>Next.jsへようこそ！</p>
    </div>
  );
}
```

```typescript
// app/about/page.tsx
export default function AboutPage() {
  return (
    <div>
      <h1>会社について</h1>
    </div>
  );
}
```

### 動的ルート

```typescript
// app/blog/[slug]/page.tsx
interface PageProps {
  params: { slug: string };
  searchParams: { [key: string]: string | string[] | undefined };
}

export default function BlogPost({ params, searchParams }: PageProps) {
  return (
    <div>
      <h1>ブログ記事: {params.slug}</h1>
      <pre>{JSON.stringify(searchParams, null, 2)}</pre>
    </div>
  );
}

// メタデータの生成
export async function generateMetadata({ params }: PageProps) {
  const post = await fetch(`https://api.example.com/posts/${params.slug}`)
    .then(res => res.json());
  
  return {
    title: post.title,
    description: post.excerpt,
  };
}
```

### キャッチオールルート

```typescript
// app/docs/[...slug]/page.tsx
export default function DocsPage({ 
  params 
}: { 
  params: { slug: string[] } 
}) {
  // /docs/a/b/c → slug = ['a', 'b', 'c']
  return (
    <div>
      <h1>ドキュメント</h1>
      <p>パス: {params.slug.join('/')}</p>
    </div>
  );
}
```

### オプショナルキャッチオール

```typescript
// app/shop/[[...slug]]/page.tsx
export default function ShopPage({ 
  params 
}: { 
  params: { slug?: string[] } 
}) {
  // /shop → slug = undefined
  // /shop/electronics → slug = ['electronics']
  // /shop/electronics/phones → slug = ['electronics', 'phones']
  
  return (
    <div>
      <h1>ショップ</h1>
      {params.slug ? (
        <p>カテゴリ: {params.slug.join(' > ')}</p>
      ) : (
        <p>すべての商品</p>
      )}
    </div>
  );
}
```

### ルートグループ

```
app/
├── (marketing)/
│   ├── about/
│   │   └── page.tsx        # /about
│   └── contact/
│       └── page.tsx        # /contact
├── (shop)/
│   ├── products/
│   │   └── page.tsx        # /products
│   └── cart/
│       └── page.tsx        # /cart
└── page.tsx                # /
```

```typescript
// app/(marketing)/layout.tsx
export default function MarketingLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div>
      <nav>
        <a href="/about">About</a>
        <a href="/contact">Contact</a>
      </nav>
      {children}
    </div>
  );
}
```

### パラレルルート

```
app/
└── dashboard/
    ├── @analytics/
    │   └── page.tsx
    ├── @team/
    │   └── page.tsx
    ├── layout.tsx
    └── page.tsx
```

```typescript
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
  analytics,
  team,
}: {
  children: React.ReactNode;
  analytics: React.ReactNode;
  team: React.ReactNode;
}) {
  return (
    <div>
      <div>{children}</div>
      <div className="grid grid-cols-2 gap-4">
        <div>{analytics}</div>
        <div>{team}</div>
      </div>
    </div>
  );
}
```

### インターセプトルート

```
app/
├── feed/
│   └── page.tsx
├── photos/
│   └── [id]/
│       └── page.tsx
└── @modal/
    └── (.)photos/
        └── [id]/
            └── page.tsx
```

```typescript
// app/@modal/(.)photos/[id]/page.tsx
// フィードページからモーダルとして開く
export default function PhotoModal({ params }: { params: { id: string } }) {
  return (
    <div className="modal">
      <img src={`/photos/${params.id}.jpg`} alt="Photo" />
    </div>
  );
}
```

---

## レイアウトとテンプレート

### レイアウト

```typescript
// app/layout.tsx (ルートレイアウト)
import './globals.css';
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'My Next.js App',
  description: 'Created with Next.js',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ja">
      <body>
        <header>
          <nav>ナビゲーション</nav>
        </header>
        <main>{children}</main>
        <footer>フッター</footer>
      </body>
    </html>
  );
}
```

### ネストされたレイアウト

```typescript
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="dashboard">
      <aside>サイドバー</aside>
      <div className="content">{children}</div>
    </div>
  );
}
```

### テンプレート

```typescript
// app/template.tsx
// レイアウトと違い、ナビゲーション時に再マウントされる
export default function Template({ children }: { children: React.ReactNode }) {
  return (
    <div className="animate-fade-in">
      {children}
    </div>
  );
}
```

---

## データフェッチング

### Server Componentsでのデータ取得

```typescript
// app/posts/page.tsx
interface Post {
  id: number;
  title: string;
  body: string;
}

async function getPosts(): Promise<Post[]> {
  const res = await fetch('https://jsonplaceholder.typicode.com/posts', {
    next: { revalidate: 3600 } // 1時間キャッシュ
  });
  
  if (!res.ok) {
    throw new Error('Failed to fetch posts');
  }
  
  return res.json();
}

export default async function PostsPage() {
  const posts = await getPosts();
  
  return (
    <div>
      <h1>ブログ記事一覧</h1>
      {posts.map(post => (
        <article key={post.id}>
          <h2>{post.title}</h2>
          <p>{post.body}</p>
        </article>
      ))}
    </div>
  );
}
```

### 並列データフェッチング

```typescript
// app/dashboard/page.tsx
async function getUser() {
  const res = await fetch('https://api.example.com/user');
  return res.json();
}

async function getPosts() {
  const res = await fetch('https://api.example.com/posts');
  return res.json();
}

async function getStats() {
  const res = await fetch('https://api.example.com/stats');
  return res.json();
}

export default async function DashboardPage() {
  // 並列で実行
  const [user, posts, stats] = await Promise.all([
    getUser(),
    getPosts(),
    getStats(),
  ]);

  return (
    <div>
      <h1>こんにちは、{user.name}さん</h1>
      <div>記事数: {posts.length}</div>
      <div>閲覧数: {stats.views}</div>
    </div>
  );
}
```

### 順次データフェッチング

```typescript
// app/user/[id]/page.tsx
async function getUser(id: string) {
  const res = await fetch(`https://api.example.com/users/${id}`);
  return res.json();
}

async function getUserPosts(userId: string) {
  const res = await fetch(`https://api.example.com/users/${userId}/posts`);
  return res.json();
}

export default async function UserPage({ 
  params 
}: { 
  params: { id: string } 
}) {
  // 1. まずユーザー情報を取得
  const user = await getUser(params.id);
  
  // 2. ユーザー情報を使って投稿を取得
  const posts = await getUserPosts(user.id);

  return (
    <div>
      <h1>{user.name}</h1>
      <h2>投稿一覧</h2>
      {posts.map((post: any) => (
        <div key={post.id}>{post.title}</div>
      ))}
    </div>
  );
}
```

### ストリーミングとSuspense

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react';

async function SlowComponent() {
  // 遅いデータ取得
  await new Promise(resolve => setTimeout(resolve, 3000));
  const data = await fetch('https://api.example.com/slow-data').then(r => r.json());
  
  return <div>{data.content}</div>;
}

async function FastComponent() {
  // 速いデータ取得
  const data = await fetch('https://api.example.com/fast-data').then(r => r.json());
  
  return <div>{data.content}</div>;
}

export default function DashboardPage() {
  return (
    <div>
      <h1>ダッシュボード</h1>
      
      {/* 速いコンポーネントはすぐに表示 */}
      <Suspense fallback={<div>読み込み中...</div>}>
        <FastComponent />
      </Suspense>
      
      {/* 遅いコンポーネントは後から表示 */}
      <Suspense fallback={<div>データ読み込み中...</div>}>
        <SlowComponent />
      </Suspense>
    </div>
  );
}
```

---

## Server ComponentsとClient Components

### Server Components（デフォルト）

```typescript
// app/components/server-component.tsx
// 'use client'がない = Server Component

interface User {
  id: string;
  name: string;
  email: string;
}

async function getUser(): Promise<User> {
  const res = await fetch('https://api.example.com/user');
  return res.json();
}

export default async function ServerComponent() {
  const user = await getUser();
  
  // サーバーでのみ実行される
  console.log('これはサーバーのログ');
  
  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      {/* 環境変数を直接使用可能 */}
      <p>API Key: {process.env.SECRET_API_KEY}</p>
    </div>
  );
}
```

#### Server Componentsの特徴

✅ サーバーでのみ実行  
✅ データベース直接アクセス可能  
✅ 環境変数に直接アクセス  
✅ バンドルサイズに含まれない  
❌ Hooks使用不可  
❌ ブラウザAPIアクセス不可  
❌ イベントハンドラー使用不可

### Client Components

```typescript
// app/components/client-component.tsx
'use client'; // Client Componentとして宣言

import { useState } from 'react';

export default function ClientComponent() {
  const [count, setCount] = useState(0);
  
  // ブラウザで実行される
  console.log('これはブラウザのログ');
  
  return (
    <div>
      <p>カウント: {count}</p>
      <button onClick={() => setCount(count + 1)}>
        増やす
      </button>
    </div>
  );
}
```

#### Client Componentsの特徴

✅ React Hooks使用可能  
✅ ブラウザAPI使用可能  
✅ イベントハンドラー使用可能  
✅ インタラクティブ  
❌ サーバー専用機能使用不可  
❌ バンドルサイズに含まれる

### 組み合わせパターン

#### パターン1: Server Component内にClient Componentを配置

```typescript
// app/page.tsx (Server Component)
import ClientCounter from './client-counter';

async function getData() {
  const res = await fetch('https://api.example.com/data');
  return res.json();
}

export default async function Page() {
  const data = await getData();
  
  return (
    <div>
      <h1>サーバーで取得: {data.title}</h1>
      {/* Client Componentを埋め込む */}
      <ClientCounter initialCount={data.count} />
    </div>
  );
}
```

```typescript
// app/client-counter.tsx
'use client';

import { useState } from 'react';

export default function ClientCounter({ initialCount }: { initialCount: number }) {
  const [count, setCount] = useState(initialCount);
  
  return (
    <button onClick={() => setCount(count + 1)}>
      カウント: {count}
    </button>
  );
}
```

#### パターン2: Client Component内にServer Componentを配置（children経由）

```typescript
// app/layout.tsx (Server Component)
import ClientLayout from './client-layout';
import ServerSidebar from './server-sidebar';

export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <ClientLayout>
      {/* Server Componentをchildrenとして渡す */}
      <ServerSidebar />
      {children}
    </ClientLayout>
  );
}
```

```typescript
// app/client-layout.tsx
'use client';

import { useState } from 'react';

export default function ClientLayout({ children }: { children: React.ReactNode }) {
  const [isOpen, setIsOpen] = useState(true);
  
  return (
    <div>
      <button onClick={() => setIsOpen(!isOpen)}>
        トグル
      </button>
      {isOpen && <div>{children}</div>}
    </div>
  );
}
```

#### パターン3: サードパーティライブラリのラップ

```typescript
// app/components/map-wrapper.tsx
'use client';

import { MapContainer, TileLayer } from 'react-leaflet';

export default function MapWrapper() {
  return (
    <MapContainer center={[35.6812, 139.7671]} zoom={13}>
      <TileLayer url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png" />
    </MapContainer>
  );
}
```

---

## Server Actions

フォーム送信やデータ変更をサーバーで処理します。

### 基本的なServer Action

```typescript
// app/actions/user-actions.ts
'use server';

import { revalidatePath } from 'next/cache';

export async function createUser(formData: FormData) {
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;
  
  // データベースに保存
  const response = await fetch('https://api.example.com/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ name, email }),
  });
  
  if (!response.ok) {
    throw new Error('ユーザー作成に失敗しました');
  }
  
  // キャッシュを再検証
  revalidatePath('/users');
  
  return { success: true };
}
```

```typescript
// app/users/new/page.tsx
import { createUser } from '@/app/actions/user-actions';

export default function NewUserPage() {
  return (
    <form action={createUser}>
      <input name="name" placeholder="名前" required />
      <input name="email" type="email" placeholder="メール" required />
      <button type="submit">作成</button>
    </form>
  );
}
```

### useActionStateを使ったServer Action

```typescript
// app/actions/post-actions.ts
'use server';

export async function createPost(prevState: any, formData: FormData) {
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;
  
  // バリデーション
  if (!title || title.length < 3) {
    return { 
      error: 'タイトルは3文字以上必要です',
      success: false 
    };
  }
  
  try {
    await fetch('https://api.example.com/posts', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ title, content }),
    });
    
    return { 
      success: true, 
      message: '投稿を作成しました' 
    };
  } catch (error) {
    return { 
      error: '投稿の作成に失敗しました',
      success: false 
    };
  }
}
```

```typescript
// app/posts/new/page.tsx
'use client';

import { useActionState } from 'react';
import { createPost } from '@/app/actions/post-actions';

export default function NewPostPage() {
  const [state, formAction, isPending] = useActionState(createPost, {
    success: false,
    message: '',
  });

  return (
    <form action={formAction}>
      <input name="title" placeholder="タイトル" required />
      <textarea name="content" placeholder="内容" required />
      
      <button type="submit" disabled={isPending}>
        {isPending ? '作成中...' : '投稿'}
      </button>
      
      {state.error && (
        <p className="text-red-500">{state.error}</p>
      )}
      {state.success && (
        <p className="text-green-500">{state.message}</p>
      )}
    </form>
  );
}
```

### useOptimisticを使った楽観的更新

```typescript
// app/components/todo-list.tsx
'use client';

import { useOptimistic } from 'react';
import { addTodo, toggleTodo } from '@/app/actions/todo-actions';

interface Todo {
  id: string;
  title: string;
  completed: boolean;
}

export default function TodoList({ initialTodos }: { initialTodos: Todo[] }) {
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    initialTodos,
    (state, newTodo: Todo) => [...state, { ...newTodo, pending: true }]
  );

  async function handleSubmit(formData: FormData) {
    const title = formData.get('title') as string;
    const newTodo = { 
      id: Date.now().toString(), 
      title, 
      completed: false 
    };
    
    // UIを即座に更新
    addOptimisticTodo(newTodo);
    
    // サーバーアクションを実行
    await addTodo(formData);
  }

  return (
    <div>
      <ul>
        {optimisticTodos.map(todo => (
          <li 
            key={todo.id}
            style={{ opacity: todo.pending ? 0.5 : 1 }}
          >
            {todo.title}
          </li>
        ))}
      </ul>
      
      <form action={handleSubmit}>
        <input name="title" placeholder="新しいタスク" />
        <button type="submit">追加</button>
      </form>
    </div>
  );
}
```

### プログラム的なServer Action呼び出し

```typescript
// app/components/delete-button.tsx
'use client';

import { deletePost } from '@/app/actions/post-actions';
import { useTransition } from 'react';

export default function DeleteButton({ postId }: { postId: string }) {
  const [isPending, startTransition] = useTransition();

  function handleDelete() {
    if (confirm('本当に削除しますか？')) {
      startTransition(async () => {
        await deletePost(postId);
      });
    }
  }

  return (
    <button 
      onClick={handleDelete}
      disabled={isPending}
      className="text-red-500"
    >
      {isPending ? '削除中...' : '削除'}
    </button>
  );
}
```

---

## キャッシング戦略

Next.jsには複数のキャッシング層があります。

### 1. Request Memoization

同一レンダリング内で同じURLへのfetchリクエストを自動的にメモ化します。

```typescript
// app/page.tsx
async function getUser() {
  const res = await fetch('https://api.example.com/user');
  return res.json();
}

export default async function Page() {
  // これらは同じリクエストで、自動的にメモ化される
  const user1 = await getUser();
  const user2 = await getUser();
  const user3 = await getUser();
  
  // 実際にはHTTPリクエストは1回のみ
  return <div>{user1.name}</div>;
}
```

### 2. Data Cache

fetchリクエストの結果をキャッシュします。

```typescript
// キャッシュされる（デフォルト）
await fetch('https://api.example.com/data');

// キャッシュされない
await fetch('https://api.example.com/data', { cache: 'no-store' });

// 60秒間キャッシュ
await fetch('https://api.example.com/data', { 
  next: { revalidate: 60 } 
});

// タグ付きキャッシュ
await fetch('https://api.example.com/data', { 
  next: { tags: ['products'] } 
});
```

### 3. Full Route Cache

ビルド時に生成されたページをキャッシュします。

```typescript
// app/products/page.tsx

// 静的にレンダリング（キャッシュされる）
export default async function ProductsPage() {
  const products = await fetch('https://api.example.com/products', {
    next: { revalidate: 3600 }
  }).then(r => r.json());
  
  return <div>{/* ... */}</div>;
}

// 動的にレンダリング（キャッシュされない）
export const dynamic = 'force-dynamic';

export default async function DynamicPage() {
  return <div>{/* ... */}</div>;
}
```

### 4. Router Cache

クライアント側のルートキャッシュ。

```typescript
// app/layout.tsx
import Link from 'next/link';

export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <div>
      <nav>
        {/* プリフェッチされる（デフォルト） */}
        <Link href="/about">About</Link>
        
        {/* プリフェッチしない */}
        <Link href="/contact" prefetch={false}>Contact</Link>
      </nav>
      {children}
    </div>
  );
}
```

### キャッシュの再検証

```typescript
// app/actions.ts
'use server';

import { revalidatePath, revalidateTag } from 'next/cache';

export async function updateProduct(productId: string) {
  // データを更新
  await fetch(`https://api.example.com/products/${productId}`, {
    method: 'PUT',
    // ...
  });
  
  // 特定のパスを再検証
  revalidatePath('/products');
  revalidatePath(`/products/${productId}`);
  
  // タグで再検証
  revalidateTag('products');
}
```

### オプトアウト

```typescript
// キャッシュを完全に無効化
export const dynamic = 'force-dynamic';
export const revalidate = 0;

export default async function Page() {
  // このページは毎回サーバーでレンダリングされる
  return <div>動的ページ</div>;
}
```

---

## 最適化機能

### 画像最適化

```typescript
import Image from 'next/image';

export default function Gallery() {
  return (
    <div>
      {/* 自動最適化 */}
      <Image
        src="/photos/hero.jpg"
        alt="Hero"
        width={800}
        height={600}
        priority // LCPに重要な画像
      />
      
      {/* レスポンシブ画像 */}
      <Image
        src="/photos/product.jpg"
        alt="Product"
        fill
        style={{ objectFit: 'cover' }}
        sizes="(max-width: 768px) 100vw, 50vw"
      />
      
      {/* 外部画像 */}
      <Image
        src="https://example.com/image.jpg"
        alt="External"
        width={400}
        height={300}
        loader={({ src, width, quality }) => {
          return `${src}?w=${width}&q=${quality || 75}`;
        }}
      />
    </div>
  );
}
```

```typescript
// next.config.js
module.exports = {
  images: {
    domains: ['example.com'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920, 2048, 3840],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
    formats: ['image/webp', 'image/avif'],
  },
};
```

### フォント最適化

```typescript
// app/layout.tsx
import { Inter, Noto_Sans_JP } from 'next/font/google';

const inter = Inter({ 
  subsets: ['latin'],
  display: 'swap',
});

const notoSansJP = Noto_Sans_JP({
  weight: ['400', '700'],
  subsets: ['latin'],
  display: 'swap',
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja" className={notoSansJP.className}>
      <body>{children}</body>
    </html>
  );
}
```

### スクリプト最適化

```typescript
import Script from 'next/script';

export default function Page() {
  return (
    <>
      {/* インラインスクリプト */}
      <Script id="gtm" strategy="afterInteractive">
        {`
          (function(w,d,s,l,i){w[l]=w[l]||[];w[l].push({'gtm.start':
          new Date().getTime(),event:'gtm.js'});var f=d.getElementsByTagName(s)[0],
          j=d.createElement(s),dl=l!='dataLayer'?'&l='+l:'';j.async=true;j.src=
          'https://www.googletagmanager.com/gtm.js?id='+i+dl;f.parentNode.insertBefore(j,f);
          })(window,document,'script','dataLayer','GTM-XXXX');
        `}
      </Script>
      
      {/* 外部スクリプト */}
      <Script
        src="https://example.com/script.js"
        strategy="lazyOnload"
        onLoad={() => console.log('Script loaded')}
      />
    </>
  );
}
```

### メタデータ最適化

```typescript
// app/layout.tsx
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: {
    default: 'My App',
    template: '%s | My App',
  },
  description: 'My Next.js application',
  keywords: ['Next.js', 'React', 'TypeScript'],
  authors: [{ name: 'Your Name' }],
  openGraph: {
    title: 'My App',
    description: 'My Next.js application',
    url: 'https://example.com',
    siteName: 'My App',
    images: [
      {
        url: 'https://example.com/og-image.jpg',
        width: 1200,
        height: 630,
      },
    ],
    locale: 'ja_JP',
    type: 'website',
  },
  twitter: {
    card: 'summary_large_image',
    title: 'My App',
    description: 'My Next.js application',
    images: ['https://example.com/twitter-image.jpg'],
  },
  robots: {
    index: true,
    follow: true,
  },
};
```

```typescript
// app/blog/[slug]/page.tsx
import type { Metadata } from 'next';

export async function generateMetadata({ 
  params 
}: { 
  params: { slug: string } 
}): Promise<Metadata> {
  const post = await fetch(`https://api.example.com/posts/${params.slug}`)
    .then(res => res.json());

  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: [post.coverImage],
    },
  };
}
```

---

## API Routes

### Route Handlers

```typescript
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';

// GET /api/users
export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const page = searchParams.get('page') || '1';
  
  const users = await fetchUsers(parseInt(page));
  
  return NextResponse.json(users);
}

// POST /api/users
export async function POST(request: NextRequest) {
  const body = await request.json();
  
  // バリデーション
  if (!body.email || !body.name) {
    return NextResponse.json(
      { error: 'Email and name are required' },
      { status: 400 }
    );
  }
  
  const user = await createUser(body);
  
  return NextResponse.json(user, { status: 201 });
}
```

### 動的ルートハンドラー

```typescript
// app/api/users/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const user = await fetchUser(params.id);
  
  if (!user) {
    return NextResponse.json(
      { error: 'User not found' },
      { status: 404 }
    );
  }
  
  return NextResponse.json(user);
}

export async function PUT(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const body = await request.json();
  const user = await updateUser(params.id, body);
  
  return NextResponse.json(user);
}

export async function DELETE(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  await deleteUser(params.id);
  
  return NextResponse.json({ success: true });
}
```

### ヘッダーとCookies

```typescript
// app/api/auth/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { cookies, headers } from 'next/headers';

export async function POST(request: NextRequest) {
  const body = await request.json();
  
  // ヘッダーの読み取り
  const headersList = headers();
  const userAgent = headersList.get('user-agent');
  
  // 認証処理
  const token = await authenticate(body.email, body.password);
  
  // Cookieの設定
  const response = NextResponse.json({ success: true });
  response.cookies.set('token', token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 60 * 60 * 24 * 7, // 7日間
  });
  
  return response;
}
```

### ファイルアップロード

```typescript
// app/api/upload/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { writeFile } from 'fs/promises';
import { join } from 'path';

export async function POST(request: NextRequest) {
  const formData = await request.formData();
  const file = formData.get('file') as File;
  
  if (!file) {
    return NextResponse.json(
      { error: 'No file uploaded' },
      { status: 400 }
    );
  }
  
  const bytes = await file.arrayBuffer();
  const buffer = Buffer.from(bytes);
  
  const path = join(process.cwd(), 'public/uploads', file.name);
  await writeFile(path, buffer);
  
  return NextResponse.json({ 
    success: true,
    url: `/uploads/${file.name}` 
  });
}
```

---

## 認証とミドルウェア

### ミドルウェア

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  // 認証チェック
  const token = request.cookies.get('token');
  
  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }
  
  // ヘッダーの追加
  const response = NextResponse.next();
  response.headers.set('x-custom-header', 'value');
  
  return response;
}

export const config = {
  matcher: [
    '/dashboard/:path*',
    '/api/:path*',
  ],
};
```

### 認証の実装例

```typescript
// app/lib/auth.ts
import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';
import jwt from 'jsonwebtoken';

export async function getSession() {
  const token = cookies().get('token')?.value;
  
  if (!token) {
    return null;
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!);
    return decoded;
  } catch (error) {
    return null;
  }
}

export async function requireAuth() {
  const session = await getSession();
  
  if (!session) {
    redirect('/login');
  }
  
  return session;
}
```

```typescript
// app/dashboard/page.tsx
import { requireAuth } from '@/app/lib/auth';

export default async function DashboardPage() {
  const user = await requireAuth();
  
  return (
    <div>
      <h1>ダッシュボード</h1>
      <p>こんにちは、{user.name}さん</p>
    </div>
  );
}
```

### ログイン機能

```typescript
// app/actions/auth-actions.ts
'use server';

import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';
import jwt from 'jsonwebtoken';

export async function login(formData: FormData) {
  const email = formData.get('email') as string;
  const password = formData.get('password') as string;
  
  // 認証処理
  const user = await authenticateUser(email, password);
  
  if (!user) {
    return { error: 'メールアドレスまたはパスワードが正しくありません' };
  }
  
  // JWTトークンを生成
  const token = jwt.sign(
    { userId: user.id, email: user.email },
    process.env.JWT_SECRET!,
    { expiresIn: '7d' }
  );
  
  // Cookieに保存
  cookies().set('token', token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 60 * 60 * 24 * 7,
  });
  
  redirect('/dashboard');
}

export async function logout() {
  cookies().delete('token');
  redirect('/login');
}
```

```typescript
// app/login/page.tsx
import { login } from '@/app/actions/auth-actions';

export default function LoginPage() {
  return (
    <form action={login}>
      <input 
        name="email" 
        type="email" 
        placeholder="メールアドレス" 
        required 
      />
      <input 
        name="password" 
        type="password" 
        placeholder="パスワード" 
        required 
      />
      <button type="submit">ログイン</button>
    </form>
  );
}
```

---

## デプロイメント

### Vercelへのデプロイ

```bash
# Vercel CLIのインストール
npm i -g vercel

# デプロイ
vercel

# 本番デプロイ
vercel --prod
```

### 環境変数の設定

```bash
# .env.local
DATABASE_URL=postgresql://...
JWT_SECRET=your-secret-key
NEXT_PUBLIC_API_URL=https://api.example.com
```

```typescript
// next.config.js
module.exports = {
  env: {
    CUSTOM_KEY: process.env.CUSTOM_KEY,
  },
};
```

### パフォーマンス最適化

```typescript
// next.config.js
module.exports = {
  // 圧縮を有効化
  compress: true,
  
  // Strict Mode
  reactStrictMode: true,
  
  // SWC Minification
  swcMinify: true,
  
  // 画像最適化
  images: {
    formats: ['image/avif', 'image/webp'],
  },
  
  // リダイレクト
  async redirects() {
    return [
      {
        source: '/old-path',
        destination: '/new-path',
        permanent: true,
      },
    ];
  },
  
  // ヘッダー
  async headers() {
    return [
      {
        source: '/:path*',
        headers: [
          {
            key: 'X-Frame-Options',
            value: 'DENY',
          },
          {
            key: 'X-Content-Type-Options',
            value: 'nosniff',
          },
        ],
      },
    ];
  },
};
```

---

## まとめ

### レンダリング戦略の選択

| 戦略 | 使用ケース |
|------|-----------|
| **SSR** | ユーザー固有データ、常に最新 |
| **SSG** | 静的コンテンツ、最速 |
| **ISR** | 定期更新コンテンツ |
| **CSR** | 高度にインタラクティブ |

### ベストプラクティス

✅ **Server Components優先** - デフォルトでServer Componentsを使用  
✅ **データフェッチの最適化** - 並列フェッチとSuspenseの活用  
✅ **キャッシング戦略** - 適切なキャッシング設定  
✅ **Server Actions** - フォーム処理の簡素化  
✅ **画像最適化** - next/imageの使用  
✅ **TypeScript** - 型安全性の確保  
✅ **セキュリティ** - 認証とミドルウェアの実装

---