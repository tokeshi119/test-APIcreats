# React開発ワークフロー

このファイルには、Reactアプリケーションを開発する際のベストプラクティスとワークフローが定義されています。

## 基本原則

### 関数コンポーネントとHooks

- **関数コンポーネントを標準として使用**: クラスコンポーネントは新規コードでは使用しない
- **Hooksを活用**: 状態管理やライフサイクルイベントにはHooksを使用する
- **カスタムHooksでロジックを再利用**: 状態を持つロジックを再利用可能な関数に抽出する

### コンポーネントの純粋性

コンポーネントとHooksは純粋であるべき:
- **冪等性**: 同じ入力に対して同じ結果を返す
- **レンダリング中の副作用なし**: 副作用はイベントハンドラやEffectsで実行する
- **非ローカル値の変更禁止**: ローカルで作成した値のみを変更する

これにより、Reactはレンダリングを最適化し、重要な更新を優先し、予期しないバグを回避できる。

## コンポーネント設計

### コンポーネント構造

- **単一責任の原則**: 各コンポーネントは1つの責任に集中（UI、状態管理、オーケストレーションのいずれか）
- **1ファイル1コンポーネント**: 1つのファイルには1つのReactコンポーネントを含める（ステートレス/純粋コンポーネントは複数可）
- **プレゼンテーショナルとコンテナの分離**: UI表示とロジックを分離する

### Props管理

- **Propsは不変として扱う**: Propsを変更しない
- **分割代入を使用**: Propsを分割代入してコードを読みやすくする
- **型定義**: TypeScriptを使用してPropsの型を定義する

```typescript
interface ButtonProps {
  label: string;
  onClick: () => void;
  disabled?: boolean;
}

function Button({ label, onClick, disabled = false }: ButtonProps) {
  return (
    <button onClick={onClick} disabled={disabled}>
      {label}
    </button>
  );
}
```

### 状態管理

- **状態は最低共通祖先に保持**: 複数の兄弟コンポーネントが同じデータを必要とする場合のみ状態をリフトアップ
- **導出可能な値は保持しない**: Propsや他の状態から導出できる値は保持せず、必要に応じて計算するかメモ化する
- **状態の最小化**: 必要な最小限の状態のみを保持する

## Hooksの使用

### 基本的なHooks

- **useState**: コンポーネントの状態を管理
- **useEffect**: 副作用（API呼び出し、購読など）を処理
- **useContext**: コンテキストから値を取得
- **useReducer**: 複雑な状態ロジックを管理

### useEffectのベストプラクティス

- **各Effectは1つの懸念事項に集中**: 複数の副作用は別々のEffectに分離
- **依存配列を適切に設定**: 依存配列に必要な値をすべて含める
- **クリーンアップ関数を実装**: 購読やタイマーなどはクリーンアップ関数で解除

```typescript
useEffect(() => {
  const subscription = subscribe();
  return () => {
    subscription.unsubscribe();
  };
}, []);
```

### カスタムHooks

- **再利用可能なロジックを抽出**: 複数のコンポーネントで使用するロジックはカスタムHookに抽出
- **プレーンな値とコールバックを公開**: JSXではなく、値とコールバックを返す
- **命名規則**: カスタムHookは`use`で始める

```typescript
function useFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    fetch(url)
      .then((res) => res.json())
      .then((data) => {
        setData(data);
        setLoading(false);
      })
      .catch((err) => {
        setError(err);
        setLoading(false);
      });
  }, [url]);

  return { data, loading, error };
}
```

## パフォーマンス最適化

### React.memo

- **子コンポーネントの不要な再レンダリングを防止**: Propsが変更された場合のみ再レンダリング
- **適切な使用**: すべてのコンポーネントをメモ化するのではなく、実際のパフォーマンス問題がある場合のみ使用

```typescript
const MemoizedComponent = React.memo(function Component({ prop1, prop2 }) {
  return <div>{prop1} {prop2}</div>;
});
```

### useMemo

- **高コストな計算をキャッシュ**: 依存配列が変更された場合のみ再計算
- **使用タイミング**: 
  - 再レンダリングで高コストな計算が発生する場合
  - 大規模なデータ処理が頻繁に行われる場合
  - 同じ計算がキーストロークや状態変更のたびに実行される場合

```typescript
const expensiveValue = useMemo(() => {
  return computeExpensiveValue(a, b);
}, [a, b]);
```

### useCallback

- **関数定義をキャッシュ**: 再レンダリング間で関数インスタンスを保持
- **使用タイミング**:
  - メモ化された子コンポーネントにコールバックを渡す場合
  - Effectフックが頻繁に発火するのを防ぐ場合
  - カスタムHooksを最適化する場合

```typescript
const memoizedCallback = useCallback(() => {
  doSomething(a, b);
}, [a, b]);
```

### 最適化の注意点

- **早期最適化を避ける**: すべてを`useCallback`や`useMemo`でラップすると、かえってパフォーマンスが悪化する可能性がある
- **プロファイリングで確認**: 実際のパフォーマンスボトルネックを特定してから最適化を適用する
- **React Compilerの検討**: React Compilerを使用すると、ビルド時に自動的に最適化される

## エラーハンドリング

### エラーバウンダリ

- **エラーバウンダリを実装**: コンポーネントツリーのエラーをキャッチして処理
- **フォールバックUIを提供**: エラー発生時にユーザーフレンドリーなUIを表示

```typescript
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    console.error('Error caught by boundary:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return <h1>Something went wrong.</h1>;
    }
    return this.props.children;
  }
}
```

## TypeScriptの使用

### 型定義

- **Propsの型定義**: すべてのコンポーネントのPropsに型を定義
- **イベントハンドラの型**: イベントハンドラの型を適切に定義

```typescript
interface FormProps {
  onSubmit: (data: FormData) => void;
}

function Form({ onSubmit }: FormProps) {
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    // ...
  };

  return <form onSubmit={handleSubmit}>...</form>;
}
```

## アクセシビリティ

### 基本的な考慮事項

- **セマンティックHTMLを使用**: 適切なHTML要素を使用する
- **ARIA属性の適切な使用**: 必要に応じてARIA属性を追加
- **キーボードナビゲーション**: キーボードで操作可能にする
- **フォーカス管理**: フォーカスの適切な管理

```typescript
<button
  aria-label="Close dialog"
  onClick={handleClose}
  onKeyDown={(e) => {
    if (e.key === 'Escape') {
      handleClose();
    }
  }}
>
  Close
</button>
```

## テスト

### テストの原則

- **ユーザー行動に焦点**: ユーザーがどのようにコンポーネントを使用するかに焦点を当てる
- **コンポーネントの動作をテスト**: 実装の詳細ではなく、コンポーネントの動作をテストする
- **統合テストを優先**: 単体テストよりも統合テストを優先する

## コードスタイル

### 命名規則

- **コンポーネント名**: PascalCase（例: `UserProfile`）
- **Hooks名**: `use`で始める（例: `useAuth`）
- **Props名**: camelCase（例: `onClick`, `isDisabled`）
- **イベントハンドラ**: `handle`で始める（例: `handleClick`）

### ファイル構造

- **コンポーネントファイル**: `ComponentName.tsx`
- **カスタムHooks**: `useHookName.ts`
- **型定義**: `types.ts`またはコンポーネントファイル内に定義
- **スタイル**: コンポーネントと同じディレクトリに配置（`ComponentName.module.css`）

## 開発時のチェックリスト

### コンポーネント設計
- [ ] 関数コンポーネントを使用しているか
- [ ] 単一責任の原則に従っているか
- [ ] Propsの型が定義されているか
- [ ] コンポーネントは純粋か（副作用がないか）

### 状態管理
- [ ] 状態は最低共通祖先に保持されているか
- [ ] 導出可能な値は保持していないか
- [ ] 状態の最小化ができているか

### Hooks
- [ ] useEffectの依存配列が適切か
- [ ] クリーンアップ関数が実装されているか
- [ ] 再利用可能なロジックはカスタムHookに抽出されているか

### パフォーマンス
- [ ] 不要な再レンダリングがないか
- [ ] 最適化が必要な箇所を特定しているか
- [ ] 早期最適化を避けているか

### アクセシビリティ
- [ ] セマンティックHTMLを使用しているか
- [ ] キーボードナビゲーションが可能か
- [ ] ARIA属性が適切に使用されているか

## 参考資料

- [React公式ドキュメント](https://react.dev/)
- [React Hooks Best Practices](https://rtcamp.com/handbook/react-best-practices/hooks/)
- [React Design Patterns](https://blog.logrocket.com/react-design-patterns/)
