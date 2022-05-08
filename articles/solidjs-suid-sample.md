---
title: "SolidJS で Material-UI（SUID）を試してみた"
emoji: "🛷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["solidjs", "typescript", "materialui", "supabase"]
published: true
---
最近話題になりつつある（？）[SolidJS](https://www.solidjs.com/) 向けの Material-UI ライブラリ **[SUID](https://suid.io/)** を少しだけ試してみました。

## SUID

https://suid.io/

React 用の [MUI](https://mui.com/) を SolidJS 向けに port（移植）するライブラリです。

バージョン 0.2.0 では 39 のコンポーネントが移植されています。

なお、[SUID](https://suid.io/) のサイト自体が SUID を使って作成・構築されています。

- **[SUID Docs](https://suid.io/getting-started/installation)**
- **[React MUI Getting started](https://mui.com/material-ui/getting-started/installation/)**

:::message
日本時間の 5/6 朝にリリースされたバージョン 0.2.0 では対応コンポーネントが 3 つ（[`Radio`](https://suid.io/components/radio-button)・[`Table`](https://suid.io/components/table)・[`Grow`](https://suid.io/components/grow)）増えました。
:::

## 試したバージョン

- SolidJS : 1.3.17
- SUID : 0.2.0
- supabase-js : 1.35.2

## 試した内容

[Supabase](https://supabase.io/) の [Quickstart: SolidJS](https://supabase.com/docs/guides/with-solidjs) を TypeScript（TSX）と Material-UI で置き換えつつ、[Card](https://suid.io/components/card) 表示も追加で試してみました。

:::message
今回は Supabase 関連部分の実装についての説明は省略します（別記事で説明予定）。
:::

## サンプルコード（全体）

こちらに置いてあります。タイトルのとおり、2022/5/31 開催予定の **[第 33 回 PostgreSQL アンカンファレンス＠オンライン](https://pgunconf.connpass.com/event/243796/)** 向けに用意したサンプルです。

https://github.com/hmatsu47/pgunconf-sample-app

ここからはサンプルコードでの使用例と気になった点を挙げていきます（当然ですが今後のバージョンで変わる可能性があります）。

### `TextField`（テキスト入力フィールド）

#### 使用例

- [Auth.tsx](https://github.com/hmatsu47/pgunconf-sample-app/blob/0f9a19d8d7e90aa6929310dbee42ee7e26afa50e/src/Auth.tsx#L77)（77 行目〜）

```tsx:Auth.tsx（77行目〜）
  <TextField
    required
    id="email"
    label="メールアドレス"
    helperText="メールアドレスを入力してください"
    value={email()}
    onChange={(event, value) => {
      setEmail(value);
    }}
  />
```

`required`指定の場合、未入力で Form を Submit しようとするとアラートが表示されブロックされます。ただし Submit を使わない場合は「*」印の表示のみでブロック動作などは生じません（自前で Validation を実装する必要あり）。

![](/images/solidjs-suid-sample/auth.png)

#### `inputRef`に対応していない

React の MUI では`TextField`で`ref`の代わりに`inputRef`が使えますが、SUID では対応していないようです。

今回は画面表示直後のフォーカスの指定に使いたかったのですが、諦めて`document.getElementById()`を使ってフォーカスを移動しました。

- [setFocus.ts](https://github.com/hmatsu47/pgunconf-sample-app/blob/0f9a19d8d7e90aa6929310dbee42ee7e26afa50e/src/commons/setFocus.ts#L3)（3 行目〜）

```typescript:setFocus.ts（3行目〜）
  const element = document.getElementById(elementId);
  element?.focus();
```

- [Auth.tsx](https://github.com/hmatsu47/pgunconf-sample-app/blob/0f9a19d8d7e90aa6929310dbee42ee7e26afa50e/src/Auth.tsx#L18)（18 行目〜）

```tsx:Auth.tsx（18行目〜）
  onMount(() => {
    setFocus('email');
  })
```

#### `multiline`に対応していない

SUID では現状[`TextareaAutosize`](https://mui.com/material-ui/react-textarea-autosize/)に対応していないので代わりに`TextField`で [Multiline](https://mui.com/material-ui/react-text-field/#multiline) を使おうと思ったのですが、`multiline`・`row`などには対応していませんでした。

仕方なく通常の`textarea`タグを使いました。

- [EditItem.tsx](https://github.com/hmatsu47/pgunconf-sample-app/blob/a5564014f98b3181132b6f707cad7627a073debd/src/EditItem.tsx#L157)（157 行目〜）

```tsx:EditItem.tsx（157行目〜）
  <textarea
    id="note"
    aria-label="Note"
    placeholder="本文を入力してください"
    onchange={(event) => {
      setNote(event.currentTarget.value);
    }}
    style="width: 100%; height: 9.0em; font-size: 1rem; line-height: 1.8em"
  >
    {note()}
  </textarea>
```

### `Card`（カード表示）

#### 使用例

- [ViewItem.tsx](https://github.com/hmatsu47/pgunconf-sample-app/blob/a5564014f98b3181132b6f707cad7627a073debd/src/ViewItem.tsx#L35)（35 行目〜）

```tsx:ViewItem.tsx（35行目〜）
  <Card
    id="itemCard"
    variant="outlined"
  >
    <CardContent>
      <Stack
        spacing={1}
        direction="row"
      >
        <CardActions sx={{ padding: 0 }}>
          <IconButton
            onClick={() => toggleExpand()}
            sx={{ padding: 0 }}
          >
            <Switch fallback={<></>}>
              <Match when={!expand()}>
                <ExpandMoreIcon aria-label="expand more"/>
              </Match>
              <Match when={expand()}>
                <ExpandLessIcon aria-label="expand less"/>
              </Match>
            </Switch>
          </IconButton>
        </CardActions>
        <Typography
          variant="h6"
          gutterBottom
        >
          {props.article.title}
        </Typography>
（中略）
      </Stack>
      <Show
        when={expand()}
        fallback={<></>}
      >
        <For
          each={props.article.note?.split('\n')}
          fallback={<></>}
        >
          {(line) =>
            <Typography
              variant="body1"
              gutterBottom
            >
              {line}
            </Typography>
          }
        </For>
      </Show>
      <CardActions sx={{ padding: 0 }}>
        <IconButton
          aria-label="edit"
          onClick={() => props.setArticle(props.article)}
          disabled={
            props.article.userId !== props.session.user!.id && props.article.noteType !== NoteType.Writable
          }
        >
          <EditIcon />
        </IconButton>
（中略）
      </CardActions>
    </CardContent>
  </Card>
```

![](/images/solidjs-suid-sample/list.png)

※ 「新規投稿」の下から続くカードです。なお「新規投稿」も`Card`で表示しています。

#### `Collapse`API に対応していない

React 用 MUI にある [Collapse API](https://mui.com/material-ui/api/collapse/) に対応していないため、上に記したコードでも SolidJS 自体が持つ [Show API](https://www.solidjs.com/docs/latest/api#%3Cshow%3E) を使って類似の処理をしています（Collapse API とは違いアニメーション動作はしません）。

```tsx:ViewItem.tsx（49行目〜：展開ボタン部分）
  <Switch fallback={<></>}>
    <Match when={!expand()}>
      <ExpandMoreIcon aria-label="expand more"/>
    </Match>
    <Match when={expand()}>
      <ExpandLessIcon aria-label="expand less"/>
    </Match>
  </Switch>
```

```tsx:ViewItem.tsx（82行目〜：実際に展開する部分）
  <Show
    when={expand()}
    fallback={<></>}
  >
    <For
      each={props.article.note?.split('\n')}
      fallback={<></>}
    >
      {(line) =>
        <Typography
          variant="body1"
          gutterBottom
        >
          {line}
        </Typography>
      }
    </For>
  </Show>
```

## 使ってみた感想

バージョン 0.2.0 ということもあり、まともなプロダクトを作るには足りないコンポーネント・API がまだ少なくない印象です。

今後のバージョンアップに期待、ですね。

## 参考：その他のサンプル画面

### プロフィール編集画面

![](/images/solidjs-suid-sample/account.png)
