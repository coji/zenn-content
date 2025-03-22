---
title: "XState と @xstate/react API リファレンス（TypeScript版）"
emoji: "🔄"
type: "tech"
topics: ["typescript", "xstate", "react", "statemachine", "frontend"]
published: false
---


# XState と @xstate/react API リファレンス（TypeScript版）

このドキュメントでは、XStateとそのReact連携ライブラリである`@xstate/react`のTypeScript向けAPIリファレンスを提供します。コア概念と関数について解説し、型安全性とTypeScript環境での使用方法に焦点を当てています。

## コア概念

APIの詳細に入る前に、XStateの基本概念を理解することが重要です：

* **マシン（Machines）:** ステートマシンは`createMachine`を使用して定義します。有限状態機械やステートチャートのロジックをカプセル化します。
* **状態（States）:** システムの取りうる状態を表します。状態はステートマシングラフのノードです。
* **イベント（Events）:** 状態遷移をトリガーする出来事を表します。イベントは`type`プロパティ（文字列）と任意のデータを持つオブジェクトです。
* **遷移（Transitions）:** イベントに応じてマシンがある状態から別の状態へ移動する方法を定義します。遷移には以下を含めることができます：
  * `target`: 次の状態
  * `actions`: 遷移中に実行する副作用（例：コンテキストの更新）
  * `guard`: 遷移が発生するために満たす必要がある条件
* **コンテキスト（Context）:** マシンの「拡張状態」- マシンの状態に関連する任意のデータ。型定義され、`assign`アクションで更新可能です。
* **初期状態（Initial State）:** マシンの開始状態。
* **最終状態（Final States）:** マシンが完了したことを示す状態。
* **階層状態（Hierarchical States）:** 他の状態内にネストされた状態（親子関係）。
* **並列状態（Parallel States）:** 同時にアクティブな状態。
* **履歴状態（History States）:** 以前に訪れた子状態を記憶する状態。
* **アクター（Actors）:** ステートマシンロジックの実行インスタンス。

## XState コアAPI (`xstate`)

### `createMachine(config, options?)`

ステートマシン定義を作成します。

```typescript
import { createMachine, assign, StateMachine, AnyEventObject, MachineContext } from 'xstate';

interface MyContext {
  count: number;
  message?: string; // オプショナルプロパティ
}

type MyEvents =
  | { type: 'INCREMENT' }
  | { type: 'DECREMENT' }
  | { type: 'SET_MESSAGE', message: string };


const myMachine = createMachine({
  types: {} as {
    context: MyContext,
    events: MyEvents
  }, // オプションだが推奨
  id: 'myMachine',
  initial: 'idle', // 初期状態ノード
  context: { // 初期コンテキスト（初期拡張状態）
    count: 0
  },
  states: {
    idle: {
      on: {
        INCREMENT: {
          target: 'active',
          actions: assign({ count: (ctx) => ctx.count + 1 }) // 型安全
        },
        DECREMENT: {
          actions: [
            assign({ count: ({context}) => context.count - 1 }),
            // `assign`を使用していないアクションも追加できます
            (ctx, event) => {
              console.log(ctx, event)
            }
          ]
        }
      }
    },
    active: {
      on: {
        RESET: {
          target: 'idle',
          actions: assign({ count: 0 }) // カウントをリセット
        },
        SET_MESSAGE: {
            actions: assign({
              message: (_, event) => event.message // イベントプロパティへの型安全なアクセス
            })
          }
      }
    }
  }
});

// `createMachine`の第二引数で、アクション、ガード等の実装を提供することもできます：
const machineWithImpl = createMachine(
  {/* ... */}, // 上記のようなマシン設定
  {
    actions: {
      myAction: (context, event) => { /* ... */ } // `myAction`の実装
    },
    guards: {
      myGuard: (context, event) => { /* ... */ } // `myGuard`の実装
    },
     // アクター、遅延等の実装も指定できます
  }
)
```

* `config`: マシン設定オブジェクト
  * `id`: （オプション）マシンの一意の文字列識別子
  * `initial`: 初期状態ノードのキー
  * `context`: 初期コンテキスト（拡張状態）オブジェクト、または初期コンテキストオブジェクトを返す関数
  * `states`: 状態ノードキーから状態ノード設定へのマッピングオブジェクト
  * `on`: マシン全体にグローバルな遷移（特定の状態に限定されない）
  * `types`: `context`や`events`のTypeScript型を定義
  * その他のプロパティ: `input`（初期入力）、`schema`（非推奨）、`predictableActionArguments`、`preserveActionOrder`、`strict`
* `options`（オプション）: 以下の実装を提供するオブジェクト：
  * `actions`: `config`で参照される名前付きアクション
  * `guards`: `config`で参照される名前付きガード
  * `actors`: 呼び出されるアクター（サービス、コールバック、プロミスなど）
  * `delays`: 遅延アクションの実装
  * `compare`: (5.4以降) コンテキスト等価性のためのカスタム比較関数

`createMachine`は`StateMachine`インスタンスを返します。マシンは自動的に開始されません。

### `createActor(logic, options?)`

指定されたアクターロジック（ステートマシン、関数など）から「アクター」を作成します。アクターはロジックを_実行_するオブジェクトです。実行を開始するには`.start()`を呼び出す必要があります。

```typescript
import { createActor, createMachine } from 'xstate';

const machine = createMachine(/* ... */);
const actor = createActor(machine, {
    input: { /* ... */ }, // オプション
    systemId: 'some-id'
});
actor.start();
```

* `logic`: `createMachine`で作成されたマシンなどのアクターロジック
* `options`:
  * `input`: アクターに提供されるデータ
  * `parent`: 作成されるアクターの親アクター
  * `id`: アクターのID
  * `systemId`: アクターが「ルート」アクターの場合に使用される不透明なID
  * `system`: このアクターをホストするアクターシステム
  * `snapshot`: このアクターが開始する状態

`ActorRef`インスタンスを返します。

### `actor.start(initialState?)`

アクターを開始します。

* `initialState`: オプション：アクターを開始する初期状態

### `actor.send(event)`

アクターにイベントを送信します。

```typescript
actor.send({ type: 'SOME_EVENT', data: '...' });
```

### `actor.getSnapshot()`

アクターの現在の状態（スナップショット）を返します。これは読み取り専用で、直接変更すべきではありません。

```typescript
const snapshot = actor.getSnapshot();
console.log(snapshot.value); // 現在の状態値
console.log(snapshot.context); // 現在のコンテキスト
console.log(snapshot.status); // 'active', 'done', 'error', または 'stopped'
```

### `actor.subscribe(listener)`

アクターからの状態更新（発行された値）を購読します。

```typescript
const subscription = actor.subscribe((state) => {
  console.log(state.value);
});

// 後で...
subscription.unsubscribe();
```

### `actor.stop()`

アクターを停止します。

### `assign(assignment)`

（アクション）マシンの`context`（拡張状態）を更新します。「アサイナー」を引数に取ります。

* **オブジェクトアサイナー:**

```typescript
assign({
  count: (context, event) => context.count + event.value,
  message: 'Count updated' // 静的な代入
});
```

* **関数アサイナー:**

```typescript
assign((context, event) => ({
  count: context.count + event.value,
  message: 'Count updated'
}));
```

`context`と`event`は型安全で、正しく推論されます。

### `raise(event)`

現在のステップ内で即座にイベントを発生させるアクション。

* `event` - 発生させるイベント

```typescript
const machine = createMachine({
  // ...
  on: {
      SOME_EVENT: {
        actions: [
            raise({ type: 'INTERNAL_EVENT' }),
        ]
      }
  }
}
```

### `log(value, label?)`

コンソールに値をログ出力するアクション（主に開発時に使用）。最初の引数として`context`と`event`を取り、オプションで`label`文字列を取ります。

* `value` - ログに出力する値（式）
* `label` - オプション：値の前にログに出力するラベル

### `state.matches(parentStateValue)`

`State`オブジェクトのメソッド。状態値が指定された`parentStateValue`のサブステートかどうかを判定します。

```typescript
const state = service.getSnapshot();

state.matches('active'); // stateがactiveまたはactiveの子孫状態の場合はtrue
state.matches({ active: 'childState' }); // stateがactive.childStateの場合はtrue
```

### `state.hasTag(tag)`

`State`オブジェクトのメソッド。状態がタグを持っているかどうかを判定します。

```typescript
const state = service.getSnapshot();

state.hasTag('loading'); // 状態ノードが'loading'タグを持っている場合はtrue
```

## `@xstate/react` フック

### `useMachine(machine, options?)`

指定された`machine`を解釈して開始し、以下のタプルを返します：

1. サービス（マシンインスタンス）の現在の`state`
2. 実行中のサービスにイベントを送信する`send`関数
3. 実行中の`service`

```typescript
import { useMachine } from '@xstate/react';
import { someMachine } from './someMachine'; // マシン定義

function MyComponent() {
  const [state, send, service] = useMachine(someMachine, {
    // オプションのマシンオプション、例えば：
    guards: {
      /* ... */
    },
    actions: {
      /* ... */
    }
  });

  // ... コンポーネント内でstate, send, serviceを使用

  return (
    <div>
      {state.matches('active') && <p>状態はアクティブです！</p>}
      <button onClick={() => send({ type: 'SOME_EVENT' })}>
        イベントを送信
      </button>
    </div>
  );
}
```

`options`にはコンテキスト値、カスタムアクションなどを含めることができます。
*注意*: `useMachine`が返す`service`は安定しています（常に同じ参照）。

### `useSelector(actor, selector, compare?)`

状態変更を購読し、選択された値を返します。コンポーネントは選択された値が変更された場合のみ再レンダリングされます。変更の判定は`compare`関数によって行われます（デフォルトは厳密等価、`===`）。

* `actor` - useMachineによって返されるサービス/アクター
* `selector` - 状態を受け取り、関連する部分を返す関数
* `compare` - オプションの関数。`previous`（前回の選択値）と`next`（次の選択値）を取り、等しい場合は`true`を返します

```typescript
import { useMachine, useSelector } from '@xstate/react';
import { someMachine } from './someMachine';

function MyComponent() {
  const [state, send] = useMachine(someMachine);

  const count = useSelector(state, (state) => state.context.count);
  const user = useSelector(
      state,
      (state) => state.context.user,
      (prevUser, nextUser) => prevUser.id === nextUser.id // カスタム比較
      );
  
  return (
    <div>
      <p>カウント: {count}</p>
      <p>ユーザー: {user.name}</p>
    </div>
  );
}
```

### `useActorRef(logic, options?)`

解釈されたマシンのアクター参照を返します。

* `logic` - `createMachine`で作成された`machine`インスタンス
* `options` - `input`などのアクターオプション

```typescript
import { useActorRef } from '@xstate/react';
import { someMachine } from './someMachine';

function MyComponent() {
  const actor = useActorRef(someMachine);

  return (
    <div>
      <button onClick={() => actor.send({ type: 'SOME_EVENT' })}>
        イベントを送信
      </button>
    </div>
  );
}
```

*注意*: `useActorRef`は同じ参照を返します。

### `useActor(actor, getSnapshot?)`

任意のアクターの現在の状態を返します。

* `actor` - アクターのようなオブジェクト（actorRef、parent、またはservice）
* `getSnapshot` - 最新の発行値を返すべきオプション関数

```typescript
import { useActor } from '@xstate/react';

function MyComponent({ someActor }) {
  const snapshot = useActor(someActor);

  return <div>現在の値: {snapshot.value}</div>;
}
```

### `createActorContext(machine, options?)`

指定されたマシン用のReact Contextを作成します。これは以下に役立ちます：

* `<ActorProvider>`の子孫にマシンを提供することでプロップドリリングを回避
* `options`に提供される共通実装の再利用

```typescript
import { createActorContext, useActorRef, useSelector } from '@xstate/react';
import { someMachine } from './someMachine';
import { createContext } from 'react';

const SomeContext = createActorContext(someMachine);

const RootComponent = () => {
  return (
    <SomeContext.Provider>
      <SomeChild />
    </SomeContext.Provider>
  )
}

const SomeChild = () => {
  const service = SomeContext.useActorRef();

  // エフェクト内で使用
  useEffect(() => {
    service.send({type: 'SOME_EVENT' });
  });

  return <div>...</div>
}


const AnotherChild = () => {
   const service = SomeContext.useActorRef();
   const isEnabled = useSelector(
    service,
    (state) => state.hasTag('enabled'));

  return <div>有効: {isEnabled ? 'はい' : 'いいえ'}</div>
}
```

## 注意点

* マシンはコンポーネントの外部、例えばモジュールレベルの`const`として作成するべきです。これにより、同じマシンの複数の使用が同じマシンを参照し、新しいマシンインスタンスを作成しないようになります。
* サービスから状態値を取得するには以下を使用できます：
  * `useSelector(service, (snapshot) => ...)` (React)
  * `$service.value` (Svelte)
  * `service.getSnapshot()` (Vanilla JS)
  * `snapshot.value` (Vue, Solid)

```typescript
  import {createMachine, assign, ActorRefFrom, fromPromise} from 'xstate';
  import { useSelector } from '@xstate/react';

  const fetchUser = (_:any, params:{userId:string}) => new Promise((resolve)=>
    setTimeout(() => {
      resolve({
        id: 42,
        name: 'David',
        date: new Date().toString()
      })
    }, 1000)
  );

  const fetchMachine = createMachine({
    initial: 'loading',
    context: {
        id: undefined as string | undefined,
        name: undefined as string | undefined,
        date: undefined as  string | undefined,
    },
    invoke: {
      src: 'getUserData',
      input: {
        userID: '42'
      },
      onDone: {
        actions: assign(({ event }) => event.output)
      }
    },
    states: {
      loading: {}
    }
  }, {
    actors: {
      fetchUser
    }
  });

  const Fetcher = () => {
      const service = useActorRef(fetchMachine);
      const snapshot = useSelector(
        service,
        (snapshot) => ({
          userId: snapshot.context.id,
          userName: snapshot.context.name,
          date: snapshot.context.date,
        })
      );

      return (
        <>
          <div>
            <span>ユーザーID: {snapshot.userId}</span>
          </div>
          <div>
            <span>ユーザー名: {snapshot.userName}</span>
          </div>
          <div>
            <span>日付: {snapshot.date}</span>
          </div>
        </>
      )
  }
```

## 注意事項

* XStateとフレームワーク固有のアダプタ（`@xstate/react`、`@xstate/vue`など）が同じ互換性のあるバージョンであることを確認してください。

## リソース

* [Vue 3 + XStateでの映画アプリの構築](https://dev.to/xstate/build-a-movie-app-with-vue-3-xstate-h91)
* [GSAPとReactでXStateマシンをアニメーション化](https://dev.to/xstate/animate-xstate-machines-with-gsap-and-react-29gc)
* [@colinhacksによるSvelte + XState統合](https://twitter.com/colinhacks/status/1242871560129191938)
* [Vue Mastery: XStateを使用した状態マシン入門](https://www.vuemastery.com/courses/advanced-components/what-are-state-machines)（Vue Masteryサブスクリプションが必要）
* [XStateとReactの使い方入門](https://www.vietnguyen.site/getting-started-with-xstate/) by Viet Nguyen
* [XStateとVueでのFSM](https://medium.com/@brockreece/fsm-with-xstate-and-vue-7e1ee65f15d9) by Brock Reece
* [React + XState - Todoリストの例](https://codesandbox.io/s/33wr94qv1) by Mateusz Burzyński

## 関連ツール

* [xstate-component-tree](https://github.com/mattpocock/xstate-component-tree) - アクターツリーをコンポーネントツリーに変換するユーティリティ。Reactでアクターを視覚化するのに役立ちます。
* [svelte-use-machine](https://github.com/bell-lab-apps/svelte-use-machine) - SvelteでXStateを使用するためのユーティリティ

## ライブ例

このチュートリアルで構築されたTodoMVCアプリは[https://xstate-todos.surge.sh](https://xstate-todos.surge.sh)で利用可能です。
