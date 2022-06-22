---
title: "SolidJS のテストを uvu で書いてみた"
emoji: "👶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["solidjs", "typescript", "test"]
published: false
---

[SolidJS](https://www.solidjs.com/) で [uvu](https://github.com/lukeed/uvu) を使ってテストコードを実装してみましたので備忘として残しておきます。

:::message
本当は [Vitest](https://vitest.dev/) で実装したかったのですが、2022/06/22 現在、こちらの Issue にあるとおり `pnpm` 以外では正常動作しないようなので、一旦諦めました（実際に `npm` でやってみて Issue のとおりに失敗しました）。

- https://github.com/solidjs/templates/issues/47

:::

### 前回のネタ

https://zenn.dev/hmatsu47/articles/solidjs-suid-sample

### 参考資料

https://github.com/lukeed/uvu/tree/master/examples/solidjs
https://dev.to/lexlohr/testing-solidjs-code-beyond-jest-39p

## テストコード実装対象

[こちら](https://github.com/hmatsu47/create-readme/tree/main/create-readme-app) のコードです。
https://github.com/hmatsu47/create-readme/tree/main/create-readme-app

[SolidJS](https://www.solidjs.com/) と [SUID](https://suid.io/) を使った簡単なものです。

:::message
自分の GitHub Pages に公開して GitHub リポジトリの README.md からリンクするコンテンツです。
:::

## 準備

以下のパッケージをインストールします（いずれも開発用なので `npm --save-dev` や `yarn -D` などで）。

- jsdom
- solid-register
- solid-testing-library
- uvu

パッケージをインストールしたら、`package.json` に以下を追記します。

```json:package.json（関係分）
{
  "type": "module",
  "scripts": {
    "test": "uvu -r solid-register"
  },
    "solid-register": {
    "compile": {
    "solid": {
      "engine": "solid",
      "extensions": [".js", ".jsx", ".ts", ".tsx"]
    }
  }
}
```

:::message
`package.json` （ファイル全体）の例はこちら。

- https://github.com/hmatsu47/create-readme/blob/main/create-readme-app/package.json
  なお、今回試した範囲では、 `tsconfig.json` で `"module": "commonjs"` を指定する必要はありませんでした。

:::

## テストコード実装例

### 1. 単純な関数

まずは SolidJS と無関係の単純な関数の入力 → 出力チェックです。

- https://github.com/hmatsu47/create-readme/blob/main/create-readme-app/test/formatDate.test.ts

```typescript:formatDate.test.ts
import { formatDate } from '../src/formatDate';
import { suite } from 'uvu';
import * as assert from 'uvu/assert';

// formatDate のテスト（最終的には別のテストでカバーするので削除予定）
const test = suite('FormatDate');

test('Format date test', () => {
  const testCase = [
    { input: new Date('2022/01/15'), expectedOutput: '2022-01-15' },
    { input: new Date('2022/11/30'), expectedOutput: '2022-11-30' },
  ];

  let expected: string[] = [];
  let actual: string[] = [];
  testCase.forEach((item) => {
    expected.push(item.expectedOutput);
    actual.push(formatDate(item.input));
  });

  assert.equal(actual, expected, 'output differs');
});

test.run();
```

`assert.equal()` を使って比較しています。

### 2. スナップショットテスト

説明の順番が逆な気もしますが、実装例の都合でスナップショットテストを先に説明します。

- https://github.com/hmatsu47/create-readme/blob/main/create-readme-app/test/ListParts.test.tsx

```typescript:ListParts.test.tsx
import { suite } from 'uvu';
import * as assert from 'uvu/assert';
import { cleanup, render } from 'solid-testing-library';
import { ListParts } from '../src/ListParts';
import { loadSnapshot, formatSnapshot, saveSnapshot } from './common/snapStore';

const test = suite('ListParts');

test.after.each(cleanup);

test('<ListParts />', () => {
  const testName = 'ListParts';
  const load = loadSnapshot(testName);
  // Qiita の記事一覧でスナップショットテストを行う
  const { container } = render(() => (
    <ListParts
      title={'Qiita'}
      id={'qiita'}
      color={'#55c500'}
      list={[
        {
          link: 'https://qiita.com/hmatsu47/items/774a3ab9441fe8eb96c7',
          title: 'SolidJS で Supabase の Row Level Security を試してみた…の続き（補足）',
          published: '2022-06-01T00:15:37+09:00',
        },
        {
          link: 'https://qiita.com/hmatsu47/items/b6ba2d2994e1632c13ea',
          title: 'SolidJS で Supabase の Row Level Security を試してみた',
          published: '2022-05-15T18:32:24+09:00',
        },
        {
          link: 'https://qiita.com/hmatsu47/items/d3f34f39c28a4b802966',
          title: '小ネタ／Aurora MySQL v1/v2 から v3 に移行する際のユーザ権限トラブルについて',
          published: '2022-03-18T19:44:57+09:00',
        },
      ]}
      url={'https://qiita.com/hmatsu47'}
    />
  ));

  const actualHtml = formatSnapshot(container.innerHTML);
  if (load.isStored) {
    // スナップショットファイルがあった場合は比較
    assert.snapshot(actualHtml, load.snapshot!);
  } else {
    // なかった場合はスナップショットをファイルに保存
    saveSnapshot(testName, actualHtml);
  }
});

test.run();
```

最後のほうで（スナップショットファイルがあった場合に） `assert.snapshot()` しています。

uvu のスナップショットテストはスナップショットをファイルに自動保存してくれないようなので、そのあたりを追加で実装してみました。

- https://github.com/hmatsu47/create-readme/blob/main/create-readme-app/test/common/snapStore.ts

```typescript:snapStore.ts
import * as fs from 'fs';

const snapFolder = './test/__snapshots__/';

type Result = {
  snapshot?: string;
  isStored: boolean;
};
// スナップショットをファイルから読み込む
export const loadSnapshot = (testName: string) => {
  const snapFileName = `${snapFolder}${testName}.snap.txt`;
  try {
    const snapshot = fs.readFileSync(snapFileName, 'utf-8');
    return { snapshot: snapshot, isStored: true } as Result;
  } catch (e) {
    console.log('スナップショットファイルがないので保存します。');
    return { snapshot: undefined, isStored: false } as Result;
  }
};
// スナップショットを整形する（インラインスタイル名と改行）
export const formatSnapshot = (snapshot: string) => {
  return snapshot.replace(/css-\w{6}/g, 'css-xxxxxx').replace(/\r/g, ' ');
};
// スナップショットをファイルに保存する
export const saveSnapshot = (testName: string, snapshot: string) => {
  const snapFileName = `${snapFolder}${testName}.snap.txt`;
  try {
    fs.writeFileSync(snapFileName, snapshot);
  } catch (e) {
    console.log(`エラーが発生しました。: ${e.message}`);
  }
};
```

`loadSnapshot()` でスナップショットをファイルから読み込み、 `saveSnapshot()` でファイルに保存します。

なお、SolidJS でスナップショットテストを行う際の問題点として、

- **インラインスタイルを使うと、その部分が `css-xxxxxx` 形式（後ろ 6 文字がランダムかつ毎回変わる）に変換されるので、そのままスナップショット比較をすることができない**

があります。
これを吸収する（ついでに改行コードも半角スペースに変換する）ために用意したのが `formatSnapshot()` です（正規表現による置換）。

### 3. ボタンと `Signal`

3 つ目の例では、スナップショットテストに加えて、 `Signal` の値とボタンクリックを使ったテストを実装しています。

- https://github.com/hmatsu47/create-readme/blob/main/create-readme-app/test/Title.test.tsx

```typescript:Title.test.tsx
import { suite } from 'uvu';
import * as assert from 'uvu/assert';
import { cleanup, render, fireEvent, screen } from 'solid-testing-library';
import { Title } from '../src/Title';
import { route, setRoute } from '../src/signal';
import { loadSnapshot, formatSnapshot, saveSnapshot } from './common/snapStore';

const test = suite('Title');
const routeNames = ['articles', 'slides'];

test.after.each(cleanup);

routeNames.forEach((routeName) => {
  test(`<Title /> route="${routeName}"`, () => {
    const testName = `Title-${routeName}`;
    const load = loadSnapshot(testName);
    // route をセットしてスナップショットテストを行う
    setRoute(routeName);

    const { container } = render(() => <Title />);

    const actualHtml = formatSnapshot(container.innerHTML);
    if (load.isStored) {
      // スナップショットファイルがあった場合は比較
      assert.snapshot(actualHtml, load.snapshot!);
    } else {
      // なかった場合はスナップショットをファイルに保存
      saveSnapshot(testName, actualHtml);
    }
  });
});

type RoutesAndButtons = {
  routeName: string;
  buttonLabel: string;
  newRouteName: string;
  newButtonLabel: string;
};

const isInDom = (node: Node): boolean =>
  !!node.parentNode && (node.parentNode === document || isInDom(node.parentNode));

const routesAndButtons: RoutesAndButtons[] = [
  {
    routeName: 'articles',
    buttonLabel: 'Slides',
    newRouteName: 'slides',
    newButtonLabel: 'Articles',
  },
  {
    routeName: 'slides',
    buttonLabel: 'Articles',
    newRouteName: 'articles',
    newButtonLabel: 'Slides',
  },
];

routesAndButtons.forEach((routesAndButton) => {
  test(`<Title /> route="${routesAndButton.routeName}" & buttonClick="${routesAndButton.buttonLabel}"`, async () => {
    setRoute(routesAndButton.routeName);

    render(() => <Title />);
    // route に対応したボタンが表示されているか？
    const button = await screen.findByRole('button', { name: routesAndButton.buttonLabel });
    assert.ok(isInDom(button));
    // ボタンクリック後に route が切り替わるか？
    fireEvent.click(button);
    assert.equal(route(), routesAndButton.newRouteName);
    // ボタン表示も変わったか？
    const newButton = await screen.findByRole('button', { name: routesAndButton.newButtonLabel });
    assert.ok(isInDom(button));
  });
});

test.run();
```

前半は先ほどの例とほぼ同じスナップショットテストですが、後半で

- `Signal` （`route`） が特定の文字列のときに、画面に特定のラベル文字列を持つボタンが表示されているか？
  - `screen.findByRole()` でボタンを探して `assert.ok()`
- そのボタンをクリックしたときに `Signal` （`route`）の文字列が想定どおりに切り替わるか？
  - `fireEvent.click()` 後に `assert.equal()`
- 同時に表示されているボタンが切り替わるか？
  - `screen.findByRole()` でボタンを探して `assert.ok()`

を実行しています。
