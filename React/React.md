# React Hooks 完全ガイド（React 19.2対応）

React 16.8から19.2までの全Hooksの包括的なリファレンス

## 目次

- [基本Hooks](#基本hooks)
- [追加Hooks](#追加hooks)
- [React 18で追加されたHooks](#react-18で追加されたhooks)
- [React 19で追加されたHooks](#react-19で追加されたhooks)
- [ライブラリHooks](#ライブラリhooks)
- [使用上のルール](#使用上のルール)

---

## 基本Hooks

React 16.8で導入された基本的なHooks

### 1. `useState()`

コンポーネントに状態を追加します。

#### 基本的な使い方

```javascript
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

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

#### 関数による更新

```javascript
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    // 前の状態に基づいて更新
    setCount(prevCount => prevCount + 1);
    setCount(prevCount => prevCount + 1);
    // countは+2される
  }

  return <button onClick={handleClick}>カウント: {count}</button>;
}
```

#### 遅延初期化

```javascript
function ExpensiveComponent() {
  // 初回レンダリング時のみ実行される
  const [state, setState] = useState(() => {
    const initialState = computeExpensiveValue();
    return initialState;
  });

  return <div>{state}</div>;
}
```

#### オブジェクトと配列の状態

```javascript
function Form() {
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    age: 0
  });

  function updateField(field, value) {
    setFormData(prev => ({
      ...prev,
      [field]: value
    }));
  }

  return (
    <form>
      <input
        value={formData.name}
        onChange={e => updateField('name', e.target.value)}
      />
      <input
        value={formData.email}
        onChange={e => updateField('email', e.target.value)}
      />
    </form>
  );
}
```

---

### 2. `useEffect()`

副作用を実行します（データ取得、購読、DOM操作など）。

#### 基本的な使い方

```javascript
import { useEffect, useState } from 'react';

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    // 副作用の実行
    fetchUser(userId).then(data => setUser(data));
  }, [userId]); // 依存配列

  return user ? <div>{user.name}</div> : <div>読み込み中...</div>;
}
```

#### クリーンアップ関数

```javascript
function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();

    // クリーンアップ関数
    return () => {
      connection.disconnect();
    };
  }, [roomId]);

  return <div>チャットルーム: {roomId}</div>;
}
```

#### 実行タイミングの制御

```javascript
function Component() {
  // 毎回実行（依存配列なし）
  useEffect(() => {
    console.log('毎回実行される');
  });

  // 初回のみ実行（空の依存配列）
  useEffect(() => {
    console.log('マウント時のみ実行');
    return () => console.log('アンマウント時のみ実行');
  }, []);

  // 特定の値が変更時のみ実行
  useEffect(() => {
    console.log('propが変更された時に実行');
  }, [prop]);
}
```

#### 非同期処理

```javascript
function DataFetcher({ id }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;

    async function fetchData() {
      try {
        setLoading(true);
        const result = await fetch(`/api/data/${id}`);
        const json = await result.json();
        
        if (!cancelled) {
          setData(json);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err);
        }
      } finally {
        if (!cancelled) {
          setLoading(false);
        }
      }
    }

    fetchData();

    return () => {
      cancelled = true;
    };
  }, [id]);

  if (loading) return <div>読み込み中...</div>;
  if (error) return <div>エラー: {error.message}</div>;
  return <div>{JSON.stringify(data)}</div>;
}
```

---

### 3. `useContext()`

Contextの値を読み取ります。

#### 基本的な使い方

```javascript
import { createContext, useContext } from 'react';

const ThemeContext = createContext('light');

function ThemedButton() {
  const theme = useContext(ThemeContext);
  
  return (
    <button className={`button-${theme}`}>
      テーマ: {theme}
    </button>
  );
}

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <ThemedButton />
    </ThemeContext.Provider>
  );
}
```

#### 複数のContextの使用

```javascript
const ThemeContext = createContext('light');
const UserContext = createContext(null);
const LanguageContext = createContext('ja');

function ProfilePage() {
  const theme = useContext(ThemeContext);
  const user = useContext(UserContext);
  const language = useContext(LanguageContext);

  return (
    <div className={theme}>
      <h1>{language === 'ja' ? 'プロフィール' : 'Profile'}</h1>
      <p>{user.name}</p>
    </div>
  );
}
```

#### カスタムフックでContextをラップ

```javascript
const AuthContext = createContext(null);

// カスタムフック
function useAuth() {
  const context = useContext(AuthContext);
  if (context === null) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}

function AuthProvider({ children }) {
  const [user, setUser] = useState(null);

  const login = async (credentials) => {
    const user = await loginAPI(credentials);
    setUser(user);
  };

  const logout = () => setUser(null);

  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

// 使用例
function ProfileButton() {
  const { user, logout } = useAuth();
  return user ? <button onClick={logout}>ログアウト</button> : null;
}
```

---

## 追加Hooks

React 16.8で基本Hooksと共に導入された追加のHooks

### 4. `useReducer()`

複雑な状態ロジックを管理します。

#### 基本的な使い方

```javascript
import { useReducer } from 'react';

const initialState = { count: 0 };

function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 };
    case 'decrement':
      return { count: state.count - 1 };
    case 'reset':
      return initialState;
    default:
      throw new Error('Unknown action type');
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <div>
      <p>カウント: {state.count}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <button onClick={() => dispatch({ type: 'reset' })}>リセット</button>
    </div>
  );
}
```

#### 複雑な状態管理

```javascript
const initialState = {
  loading: false,
  data: null,
  error: null
};

function dataReducer(state, action) {
  switch (action.type) {
    case 'FETCH_START':
      return { ...state, loading: true, error: null };
    case 'FETCH_SUCCESS':
      return { loading: false, data: action.payload, error: null };
    case 'FETCH_ERROR':
      return { loading: false, data: null, error: action.payload };
    default:
      return state;
  }
}

function DataComponent() {
  const [state, dispatch] = useReducer(dataReducer, initialState);

  useEffect(() => {
    dispatch({ type: 'FETCH_START' });
    
    fetchData()
      .then(data => dispatch({ type: 'FETCH_SUCCESS', payload: data }))
      .catch(error => dispatch({ type: 'FETCH_ERROR', payload: error }));
  }, []);

  if (state.loading) return <div>読み込み中...</div>;
  if (state.error) return <div>エラー: {state.error.message}</div>;
  return <div>{JSON.stringify(state.data)}</div>;
}
```

#### 初期化関数

```javascript
function init(initialCount) {
  return { count: initialCount };
}

function Counter({ initialCount }) {
  const [state, dispatch] = useReducer(reducer, initialCount, init);
  
  return <div>カウント: {state.count}</div>;
}
```

---

### 5. `useCallback()`

メモ化されたコールバック関数を返します。

#### 基本的な使い方

```javascript
import { useCallback, useState } from 'react';

function SearchComponent() {
  const [query, setQuery] = useState('');

  // queryが変更されない限り、同じ関数インスタンスを返す
  const handleSearch = useCallback(() => {
    performSearch(query);
  }, [query]);

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <SearchResults onSearch={handleSearch} />
    </div>
  );
}

// 子コンポーネントの不要な再レンダリングを防ぐ
const SearchResults = memo(({ onSearch }) => {
  // onSearchが変更されない限り再レンダリングされない
  return <button onClick={onSearch}>検索</button>;
});
```

#### イベントハンドラーの最適化

```javascript
function TodoList({ todos }) {
  const [filter, setFilter] = useState('all');

  const handleToggle = useCallback((id) => {
    toggleTodo(id);
  }, []); // 依存配列が空なので、一度だけ作成される

  const handleDelete = useCallback((id) => {
    deleteTodo(id);
  }, []);

  const filteredTodos = useMemo(() => {
    return todos.filter(todo => {
      if (filter === 'completed') return todo.completed;
      if (filter === 'active') return !todo.completed;
      return true;
    });
  }, [todos, filter]);

  return (
    <div>
      <FilterButtons onFilterChange={setFilter} />
      {filteredTodos.map(todo => (
        <TodoItem
          key={todo.id}
          todo={todo}
          onToggle={handleToggle}
          onDelete={handleDelete}
        />
      ))}
    </div>
  );
}
```

---

### 6. `useMemo()`

計算結果をメモ化します。

#### 基本的な使い方

```javascript
import { useMemo, useState } from 'react';

function ExpensiveComponent({ items, filter }) {
  // filterまたはitemsが変更された時のみ再計算
  const filteredItems = useMemo(() => {
    console.log('計算実行');
    return items.filter(item => item.category === filter);
  }, [items, filter]);

  return (
    <ul>
      {filteredItems.map(item => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

#### 複雑な計算の最適化

```javascript
function DataVisualization({ data }) {
  const statistics = useMemo(() => {
    // 重い計算処理
    const sum = data.reduce((acc, val) => acc + val, 0);
    const average = sum / data.length;
    const max = Math.max(...data);
    const min = Math.min(...data);
    
    return { sum, average, max, min };
  }, [data]);

  return (
    <div>
      <p>合計: {statistics.sum}</p>
      <p>平均: {statistics.average}</p>
      <p>最大: {statistics.max}</p>
      <p>最小: {statistics.min}</p>
    </div>
  );
}
```

#### オブジェクトの参照の安定化

```javascript
function ParentComponent() {
  const [count, setCount] = useState(0);

  // countが変更されない限り、同じオブジェクト参照を保持
  const config = useMemo(() => ({
    theme: 'dark',
    fontSize: 14,
    count: count
  }), [count]);

  return <ChildComponent config={config} />;
}
```

---

### 7. `useRef()`

ミュータブルな値や DOM 要素への参照を保持します。

#### DOM要素への参照

```javascript
import { useRef } from 'react';

function TextInput() {
  const inputRef = useRef(null);

  function handleClick() {
    // DOM要素に直接アクセス
    inputRef.current.focus();
  }

  return (
    <div>
      <input ref={inputRef} type="text" />
      <button onClick={handleClick}>フォーカス</button>
    </div>
  );
}
```

#### ミュータブルな値の保持

```javascript
function Timer() {
  const [count, setCount] = useState(0);
  const intervalRef = useRef(null);

  function start() {
    if (intervalRef.current !== null) return;
    
    intervalRef.current = setInterval(() => {
      setCount(c => c + 1);
    }, 1000);
  }

  function stop() {
    if (intervalRef.current !== null) {
      clearInterval(intervalRef.current);
      intervalRef.current = null;
    }
  }

  useEffect(() => {
    return () => stop(); // クリーンアップ
  }, []);

  return (
    <div>
      <p>カウント: {count}</p>
      <button onClick={start}>開始</button>
      <button onClick={stop}>停止</button>
    </div>
  );
}
```

#### 前の値の保持

```javascript
function usePrevious(value) {
  const ref = useRef();
  
  useEffect(() => {
    ref.current = value;
  }, [value]);
  
  return ref.current;
}

function Counter() {
  const [count, setCount] = useState(0);
  const prevCount = usePrevious(count);

  return (
    <div>
      <p>現在: {count}</p>
      <p>前回: {prevCount}</p>
      <button onClick={() => setCount(count + 1)}>増やす</button>
    </div>
  );
}
```

#### React 19.2: ref コールバックのクリーンアップ

```javascript
function VideoPlayer({ src }) {
  const refCallback = (element) => {
    if (element) {
      const observer = new IntersectionObserver(entries => {
        if (entries[0].isIntersecting) {
          element.play();
        } else {
          element.pause();
        }
      });
      
      observer.observe(element);
      
      // クリーンアップ関数を返せる（React 19.2+）
      return () => {
        observer.disconnect();
      };
    }
  };

  return <video ref={refCallback} src={src} />;
}
```

---

### 8. `useImperativeHandle()`

親コンポーネントに公開するインスタンス値をカスタマイズします。

#### 基本的な使い方

```javascript
import { forwardRef, useImperativeHandle, useRef } from 'react';

const FancyInput = forwardRef((props, ref) => {
  const inputRef = useRef();

  useImperativeHandle(ref, () => ({
    focus: () => {
      inputRef.current.focus();
    },
    scrollIntoView: () => {
      inputRef.current.scrollIntoView();
    },
    // カスタムメソッドを公開
    getValue: () => {
      return inputRef.current.value;
    }
  }));

  return <input ref={inputRef} {...props} />;
});

// 使用例
function Parent() {
  const inputRef = useRef();

  function handleClick() {
    inputRef.current.focus();
    console.log(inputRef.current.getValue());
  }

  return (
    <div>
      <FancyInput ref={inputRef} />
      <button onClick={handleClick}>操作</button>
    </div>
  );
}
```

---

### 9. `useLayoutEffect()`

DOM 変更後、ブラウザの描画前に同期的に実行されます。

#### 基本的な使い方

```javascript
import { useLayoutEffect, useRef, useState } from 'react';

function Tooltip() {
  const [height, setHeight] = useState(0);
  const ref = useRef(null);

  // DOM更新後、画面描画前に実行
  useLayoutEffect(() => {
    const height = ref.current.getBoundingClientRect().height;
    setHeight(height);
  }, []);

  return (
    <div ref={ref}>
      <p>要素の高さ: {height}px</p>
    </div>
  );
}
```

#### DOM測定とレイアウト調整

```javascript
function ResizablePanel({ children }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  const panelRef = useRef(null);

  useLayoutEffect(() => {
    const panel = panelRef.current;
    const rect = panel.getBoundingClientRect();
    
    // ビューポートからはみ出る場合は調整
    if (rect.right > window.innerWidth) {
      setPosition(prev => ({ 
        ...prev, 
        x: window.innerWidth - rect.width 
      }));
    }
  }, [children]);

  return (
    <div 
      ref={panelRef}
      style={{ 
        position: 'absolute',
        left: position.x,
        top: position.y 
      }}
    >
      {children}
    </div>
  );
}
```

---

### 10. `useDebugValue()`

React DevTools でカスタムフックのラベルを表示します。

#### 基本的な使い方

```javascript
import { useDebugValue, useState, useEffect } from 'react';

function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  useEffect(() => {
    function handleOnline() {
      setIsOnline(true);
    }
    function handleOffline() {
      setIsOnline(false);
    }

    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);

    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);

  // DevToolsに "OnlineStatus: Online" または "OnlineStatus: Offline" と表示
  useDebugValue(isOnline ? 'Online' : 'Offline');

  return isOnline;
}
```

#### フォーマット関数の使用

```javascript
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(data => {
        setData(data);
        setLoading(false);
      });
  }, [url]);

  // 第2引数でフォーマット関数を指定（重い処理の場合に有用）
  useDebugValue(data, data => 
    data ? `Loaded: ${Object.keys(data).length} keys` : 'Loading...'
  );

  return { data, loading };
}
```

---

## React 18で追加されたHooks

### 11. `useId()`

ユニークなIDを生成します（アクセシビリティ対応）。

#### 基本的な使い方

```javascript
import { useId } from 'react';

function PasswordField() {
  const passwordHintId = useId();

  return (
    <div>
      <label>
        パスワード:
        <input
          type="password"
          aria-describedby={passwordHintId}
        />
      </label>
      <p id={passwordHintId}>
        パスワードは8文字以上である必要があります
      </p>
    </div>
  );
}
```

#### 複数のIDの生成

```javascript
function NameFields() {
  const id = useId();

  return (
    <div>
      <label htmlFor={`${id}-firstName`}>名:</label>
      <input id={`${id}-firstName`} type="text" />
      
      <label htmlFor={`${id}-lastName`}>姓:</label>
      <input id={`${id}-lastName`} type="text" />
    </div>
  );
}
```

#### リスト内での使用

```javascript
function FormFields({ fields }) {
  return (
    <>
      {fields.map(field => (
        <Field key={field.id} field={field} />
      ))}
    </>
  );
}

function Field({ field }) {
  const id = useId();
  
  return (
    <div>
      <label htmlFor={id}>{field.label}</label>
      <input id={id} type={field.type} />
    </div>
  );
}
```

---

### 12. `useTransition()`

優先度の低い状態更新をマークします。

#### 基本的な使い方

```javascript
import { useState, useTransition } from 'react';

function SearchResults() {
  const [isPending, startTransition] = useTransition();
  const [input, setInput] = useState('');
  const [results, setResults] = useState([]);

  function handleChange(e) {
    // 即座に更新（高優先度）
    setInput(e.target.value);

    // 遅延可能な更新（低優先度）
    startTransition(() => {
      const filtered = performExpensiveSearch(e.target.value);
      setResults(filtered);
    });
  }

  return (
    <div>
      <input value={input} onChange={handleChange} />
      {isPending && <Spinner />}
      <ResultsList results={results} />
    </div>
  );
}
```

#### React 19での非同期サポート

```javascript
function AsyncSearchResults() {
  const [isPending, startTransition] = useTransition();
  const [results, setResults] = useState([]);

  function handleSearch(query) {
    startTransition(async () => {
      // 非同期処理も直接サポート（React 19+）
      const data = await fetchSearchResults(query);
      setResults(data);
    });
  }

  return (
    <div>
      <SearchInput onSearch={handleSearch} />
      {isPending ? <Spinner /> : <ResultsList results={results} />}
    </div>
  );
}
```

---

### 13. `useDeferredValue()`

値の更新を遅延させます。

#### 基本的な使い方

```javascript
import { useState, useDeferredValue } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  // queryの遅延バージョン
  const deferredQuery = useDeferredValue(query);

  return (
    <div>
      <input 
        value={query} 
        onChange={e => setQuery(e.target.value)} 
      />
      {/* 入力は即座に反応 */}
      <p>入力中: {query}</p>
      
      {/* 検索結果は遅延更新される */}
      <SearchResults query={deferredQuery} />
    </div>
  );
}

function SearchResults({ query }) {
  // queryが古い値の場合、透明度を下げる
  const isStale = query !== query;
  
  const results = useMemo(() => {
    return performExpensiveSearch(query);
  }, [query]);

  return (
    <div style={{ opacity: isStale ? 0.5 : 1 }}>
      {results.map(result => (
        <div key={result.id}>{result.title}</div>
      ))}
    </div>
  );
}
```

#### useTransition との比較

```javascript
function ComparisonExample() {
  const [input, setInput] = useState('');
  
  // 方法1: useDeferredValue
  const deferredInput = useDeferredValue(input);
  
  // 方法2: useTransition
  const [isPending, startTransition] = useTransition();
  const [transitionalInput, setTransitionalInput] = useState('');
  
  function handleChangeWithTransition(e) {
    setInput(e.target.value);
    startTransition(() => {
      setTransitionalInput(e.target.value);
    });
  }

  return (
    <div>
      {/* useDeferredValueを使用 */}
      <div>
        <input value={input} onChange={e => setInput(e.target.value)} />
        <Results query={deferredInput} />
      </div>

      {/* useTransitionを使用 */}
      <div>
        <input value={input} onChange={handleChangeWithTransition} />
        {isPending && <Spinner />}
        <Results query={transitionalInput} />
      </div>
    </div>
  );
}
```

---

### 14. `useSyncExternalStore()`

外部ストアを購読します。

#### 基本的な使い方

```javascript
import { useSyncExternalStore } from 'react';

// 外部ストアの例
const store = {
  state: { count: 0 },
  listeners: new Set(),
  
  getState() {
    return this.state;
  },
  
  subscribe(listener) {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  },
  
  setState(newState) {
    this.state = { ...this.state, ...newState };
    this.listeners.forEach(listener => listener());
  }
};

function useStore() {
  const state = useSyncExternalStore(
    store.subscribe.bind(store),
    store.getState.bind(store)
  );
  
  return state;
}

function Counter() {
  const state = useStore();
  
  return (
    <div>
      <p>カウント: {state.count}</p>
      <button onClick={() => store.setState({ count: state.count + 1 })}>
        増やす
      </button>
    </div>
  );
}
```

#### ブラウザAPIとの連携

```javascript
function useOnlineStatus() {
  const isOnline = useSyncExternalStore(
    subscribe,
    getSnapshot,
    getServerSnapshot
  );
  
  return isOnline;
}

function subscribe(callback) {
  window.addEventListener('online', callback);
  window.addEventListener('offline', callback);
  return () => {
    window.removeEventListener('online', callback);
    window.removeEventListener('offline', callback);
  };
}

function getSnapshot() {
  return navigator.onLine;
}

function getServerSnapshot() {
  return true; // サーバーサイドでは常にオンラインと仮定
}

// 使用例
function StatusIndicator() {
  const isOnline = useOnlineStatus();
  return <div>{isOnline ? '🟢 オンライン' : '🔴 オフライン'}</div>;
}
```

#### Redux風のストアとの連携

```javascript
function createStore(initialState) {
  let state = initialState;
  const listeners = new Set();

  return {
    getState: () => state,
    
    setState: (newState) => {
      state = { ...state, ...newState };
      listeners.forEach(listener => listener());
    },
    
    subscribe: (listener) => {
      listeners.add(listener);
      return () => listeners.delete(listener);
    }
  };
}

const userStore = createStore({ name: '', email: '' });

function useUserStore(selector) {
  return useSyncExternalStore(
    userStore.subscribe,
    () => selector(userStore.getState())
  );
}

function UserProfile() {
  const userName = useUserStore(state => state.name);
  const userEmail = useUserStore(state => state.email);

  return (
    <div>
      <p>名前: {userName}</p>
      <p>Email: {userEmail}</p>
    </div>
  );
}
```

---

### 15. `useInsertionEffect()`

CSS-in-JSライブラリ用の特殊なEffect Hook。

#### 基本的な使い方

```javascript
import { useInsertionEffect } from 'react';

function useCSS(rule) {
  useInsertionEffect(() => {
    // DOMが変更される前にスタイルを挿入
    const styleElement = document.createElement('style');
    styleElement.textContent = rule;
    document.head.appendChild(styleElement);
    
    return () => {
      document.head.removeChild(styleElement);
    };
  }, [rule]);
}

function Component() {
  useCSS(`
    .my-component {
      color: blue;
      font-size: 16px;
    }
  `);
  
  return <div className="my-component">スタイル適用</div>;
}
```

#### CSS-in-JSライブラリでの使用例

```javascript
function useDynamicStyle(className, styles) {
  useInsertionEffect(() => {
    const styleSheet = document.createElement('style');
    const rules = Object.entries(styles)
      .map(([key, value]) => `${key}: ${value};`)
      .join(' ');
    
    styleSheet.textContent = `.${className} { ${rules} }`;
    document.head.appendChild(styleSheet);
    
    return () => {
      document.head.removeChild(styleSheet);
    };
  }, [className, styles]);
}

function ThemedButton({ theme }) {
  const className = `button-${theme}`;
  
  useDynamicStyle(className, {
    'background-color': theme === 'dark' ? '#333' : '#fff',
    'color': theme === 'dark' ? '#fff' : '#333',
    'padding': '10px 20px',
    'border-radius': '4px'
  });

  return <button className={className}>ボタン</button>;
}
```

---

## React 19で追加されたHooks

### 16. `use()` ⭐ NEW

PromiseやContextを読み取る統合Hook。

#### Promiseの読み取り

```javascript
import { use, Suspense } from 'react';

function UserProfile({ userPromise }) {
  // Promiseを直接読み取る
  const user = use(userPromise);
  
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}

function App() {
  const userPromise = fetchUser(123);
  
  return (
    <Suspense fallback={<Loading />}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}
```

#### Contextの読み取り

```javascript
import { use, createContext } from 'react';

const ThemeContext = createContext('light');

function ThemedComponent() {
  // useContextの代わりに使用可能
  const theme = use(ThemeContext);
  return <div className={theme}>テーマ: {theme}</div>;
}
```

#### 条件付き使用（他のHooksとの最大の違い）

```javascript
function ConditionalData({ shouldFetch, dataPromise }) {
  let data = null;
  
  // 条件分岐内でも使用可能！
  if (shouldFetch) {
    data = use(dataPromise);
  }
  
  return <div>{data ? data.content : 'データなし'}</div>;
}

function UserOrGuest({ isLoggedIn, userPromise }) {
  if (!isLoggedIn) {
    return <GuestView />;
  }
  
  // 早期リターン後でも使用可能
  const user = use(userPromise);
  return <UserView user={user} />;
}
```

#### ループ内での使用

```javascript
function MultipleUsers({ userPromises }) {
  return (
    <div>
      {userPromises.map((promise, index) => {
        // ループ内でも使用可能
        const user = use(promise);
        return <UserCard key={index} user={user} />;
      })}
    </div>
  );
}
```

---

### 17. `useActionState()`

フォームアクションの状態を管理します。

#### 基本的な使い方

```javascript
import { useActionState } from 'react';

function ContactForm() {
  const [state, action, isPending] = useActionState(
    async (previousState, formData) => {
      const name = formData.get('name');
      const email = formData.get('email');
      const message = formData.get('message');
      
      try {
        await sendContactMessage({ name, email, message });
        return { 
          success: true, 
          message: 'メッセージを送信しました！' 
        };
      } catch (error) {
        return { 
          success: false, 
          message: `エラー: ${error.message}` 
        };
      }
    },
    { success: null, message: '' } // 初期状態
  );

  return (
    <form action={action}>
      <input name="name" placeholder="お名前" required />
      <input name="email" type="email" placeholder="メール" required />
      <textarea name="message" placeholder="メッセージ" required />
      
      <button type="submit" disabled={isPending}>
        {isPending ? '送信中...' : '送信'}
      </button>
      
      {state.message && (
        <p style={{ color: state.success ? 'green' : 'red' }}>
          {state.message}
        </p>
      )}
    </form>
  );
}
```

#### 前の状態を使用した累積処理

```javascript
function CommentSection({ postId }) {
  const [state, action, isPending] = useActionState(
    async (previousState, formData) => {
      const comment = formData.get('comment');
      
      const newComment = await postComment(postId, comment);
      
      // 前の状態を利用して累積
      return {
        comments: [...previousState.comments, newComment],
        totalCount: previousState.totalCount + 1
      };
    },
    { comments: [], totalCount: 0 }
  );

  return (
    <div>
      <h3>コメント ({state.totalCount})</h3>
      
      {state.comments.map(comment => (
        <div key={comment.id}>{comment.text}</div>
      ))}
      
      <form action={action}>
        <textarea name="comment" required />
        <button type="submit" disabled={isPending}>
          {isPending ? '投稿中...' : 'コメント'}
        </button>
      </form>
    </div>
  );
}
```

---

### 18. `useOptimistic()`

楽観的更新を実装します。

#### 基本的な使い方

```javascript
import { useOptimistic, useState } from 'react';

function TodoList() {
  const [todos, setTodos] = useState([]);
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    todos,
    (state, newTodo) => [...state, { ...newTodo, sending: true }]
  );

  async function handleSubmit(formData) {
    const title = formData.get('title');
    const newTodo = { id: Date.now(), title, completed: false };
    
    // UIを即座に更新
    addOptimisticTodo(newTodo);
    
    try {
      const savedTodo = await saveTodoToServer(newTodo);
      setTodos([...todos, savedTodo]);
    } catch (error) {
      // エラー時は元に戻る
      console.error('保存失敗', error);
    }
  }

  return (
    <div>
      <ul>
        {optimisticTodos.map(todo => (
          <li 
            key={todo.id}
            style={{ opacity: todo.sending ? 0.5 : 1 }}
          >
            {todo.title}
            {todo.sending && ' (送信中...)'}
          </li>
        ))}
      </ul>
      
      <form action={handleSubmit}>
        <input name="title" required />
        <button type="submit">追加</button>
      </form>
    </div>
  );
}
```

#### いいね機能の実装

```javascript
function LikeButton({ postId, initialLikes, initialLiked }) {
  const [likes, setLikes] = useState(initialLikes);
  const [liked, setLiked] = useState(initialLiked);
  
  const [optimisticState, setOptimisticState] = useOptimistic(
    { likes, liked },
    (state, newLiked) => ({
      likes: state.likes + (newLiked ? 1 : -1),
      liked: newLiked
    })
  );

  async function handleClick() {
    const newLiked = !liked;
    
    // 楽観的更新
    setOptimisticState(newLiked);
    
    try {
      const result = await toggleLike(postId, newLiked);
      setLikes(result.likes);
      setLiked(result.liked);
    } catch (error) {
      // エラー時は元の状態に戻る
      console.error('いいね失敗', error);
    }
  }

  return (
    <button onClick={handleClick}>
      {optimisticState.liked ? '❤️' : '🤍'} {optimisticState.likes}
    </button>
  );
}
```

---

## ライブラリHooks

### React DOM Hooks

#### `useFormStatus()`

フォームの送信状態を取得します。

```javascript
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending, data, method, action } = useFormStatus();
  
  return (
    <button type="submit" disabled={pending}>
      {pending ? '送信中...' : '送信'}
    </button>
  );
}

function MyForm() {
  async function handleSubmit(formData) {
    await saveData(formData);
  }

  return (
    <form action={handleSubmit}>
      <input name="username" />
      <SubmitButton />
    </form>
  );
}
```

#### `useFormState()` (旧名)

> 注: React 19では `useActionState()` に改名されました

---

## 使用上のルール

### Hooksのルール

1. **トップレベルでのみ呼び出す**

```javascript
// ❌ NG: 条件分岐内
function Component({ condition }) {
  if (condition) {
    const [state, setState] = useState(0); // エラー！
  }
}

// ✅ OK
function Component({ condition }) {
  const [state, setState] = useState(0);
  
  if (condition) {
    // stateを使用
  }
}
```

2. **React関数内でのみ呼び出す**

```javascript
// ❌ NG: 通常の関数
function regularFunction() {
  const [state, setState] = useState(0); // エラー！
}

// ✅ OK: コンポーネント内
function Component() {
  const [state, setState] = useState(0);
}

// ✅ OK: カスタムフック内
function useCustomHook() {
  const [state, setState] = useState(0);
  return state;
}
```

3. **use() の例外**

```javascript
// ✅ OK: use()は条件分岐内でも使用可能
function Component({ shouldLoad, promise }) {
  if (shouldLoad) {
    const data = use(promise); // OK!
    return <div>{data}</div>;
  }
  return <div>Not loading</div>;
}
```

### カスタムHooksの作成

```javascript
// カスタムフックは "use" で始める
function useWindowSize() {
  const [size, setSize] = useState({
    width: window.innerWidth,
    height: window.innerHeight
  });

  useEffect(() => {
    function handleResize() {
      setSize({
        width: window.innerWidth,
        height: window.innerHeight
      });
    }

    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return size;
}

// 使用例
function Component() {
  const { width, height } = useWindowSize();
  return <div>画面サイズ: {width} x {height}</div>;
}
```

---

## まとめ

### Hooks一覧表

| Hook | 用途 | 導入バージョン |
|------|------|---------------|
| `useState` | 状態管理 | React 16.8 |
| `useEffect` | 副作用の実行 | React 16.8 |
| `useContext` | Context値の読み取り | React 16.8 |
| `useReducer` | 複雑な状態管理 | React 16.8 |
| `useCallback` | コールバックのメモ化 | React 16.8 |
| `useMemo` | 計算結果のメモ化 | React 16.8 |
| `useRef` | ミュータブルな参照 | React 16.8 |
| `useImperativeHandle` | ref のカスタマイズ | React 16.8 |
| `useLayoutEffect` | 同期的な副作用 | React 16.8 |
| `useDebugValue` | デバッグラベル | React 16.8 |
| `useId` | ユニークID生成 | React 18 |
| `useTransition` | 低優先度更新 | React 18 |
| `useDeferredValue` | 値の遅延更新 | React 18 |
| `useSyncExternalStore` | 外部ストア購読 | React 18 |
| `useInsertionEffect` | CSS-in-JS用 | React 18 |
| `use` | Promise/Context読み取り | React 19 |
| `useActionState` | フォームアクション | React 19 |
| `useOptimistic` | 楽観的更新 | React 19 |
| `useFormStatus` | フォーム状態 | React DOM |

### 選択ガイド

**状態管理:**
- シンプルな状態 → `useState`
- 複雑な状態ロジック → `useReducer`
- グローバル状態 → `useContext`

**パフォーマンス最適化:**
- 計算結果のキャッシュ → `useMemo`
- 関数のキャッシュ → `useCallback`
- 低優先度の更新 → `useTransition` / `useDeferredValue`

**副作用:**
- 非同期処理、購読 → `useEffect`
- DOM測定 → `useLayoutEffect`
- 外部ストア → `useSyncExternalStore`

**フォーム:**
- アクション管理 → `useActionState`
- 送信状態 → `useFormStatus`
- 楽観的更新 → `useOptimistic`

**その他:**
- DOM参照 → `useRef`
- アクセシビリティID → `useId`
- Promise/Context → `use`

---

## ベストプラクティス

### 1. 依存配列を正しく指定する

```javascript
// ❌ NG: 依存配列の指定漏れ
useEffect(() => {
  doSomething(prop);
}, []); // propが変更されても実行されない

// ✅ OK
useEffect(() => {
  doSomething(prop);
}, [prop]);
```

### 2. クリーンアップを忘れない

```javascript
// ✅ OK
useEffect(() => {
  const subscription = subscribe();
  
  return () => {
    subscription.unsubscribe();
  };
}, []);
```

### 3. カスタムフックで再利用性を高める

```javascript
// ✅ OK: ロジックを抽出
function useAuth() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, setUser);
    setLoading(false);
    return unsubscribe;
  }, []);
  
  return { user, loading };
}
```

### 4. 過度な最適化を避ける

```javascript
// ❌ NG: 不要なメモ化
const value = useMemo(() => x + y, [x, y]); // シンプルな計算には不要

// ✅ OK: 重い計算のみメモ化
const expensiveValue = useMemo(() => {
  return heavyComputation(data);
}, [data]);
```

---