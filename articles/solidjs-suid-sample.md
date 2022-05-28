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

バージョン 0.4.0 時点では 41 のコンポーネントが移植されています。

なお、[SUID](https://suid.io/) のサイト自体が SUID を使って作成・構築されています。

- **[SUID Docs](https://suid.io/getting-started/installation)**
- **[React MUI Getting started](https://mui.com/material-ui/getting-started/installation/)**

:::message
日本時間の 5/6 朝にリリースされたバージョン 0.2.0 では対応コンポーネントが 3 つ（[`Radio`](https://suid.io/components/radio-button)・[`Table`](https://suid.io/components/table)・[`Grow`](https://suid.io/components/grow)）増えました。
また、5/18 にリリースされたバージョン 0.3.0 では対応コンポーネントが 2 つ（[`Avatar`](https://suid.io/components/avatar)・[`Popover`](https://suid.io/components/popover)）増えました。
なお、5/26 にリリースされたバージョン 0.4.0 では対応コンポーネントは増えませんでした。
:::

## 試したバージョン

- SolidJS : 1.4.3
- SUID : 0.4.0
- supabase-js : 1.35.3

## 試した内容

[Supabase](https://supabase.io/) の [Quickstart: SolidJS](https://supabase.com/docs/guides/with-solidjs) を TypeScript（TSX）と Material-UI で置き換えつつ、[Card](https://suid.io/components/card) 表示も追加で試してみました。

また、0.3.0 で追加された [`Avatar`](https://suid.io/components/avatar) も試してきました。

:::message
今回は Supabase 関連部分の実装についての説明は省略します（↓の記事で説明）。
- **[SolidJS で Supabase の Row Level Security を試してみた](https://qiita.com/hmatsu47/items/b6ba2d2994e1632c13ea)**
:::

## サンプルコード（全体）

こちらに置いてあります。タイトルのとおり、2022/5/31 開催予定の **[第 33 回 PostgreSQL アンカンファレンス＠オンライン](https://pgunconf.connpass.com/event/243796/)** 向けに用意したサンプルです。

https://github.com/hmatsu47/pgunconf-sample-app

ここからはサンプルコードでの使用例と気になった点を挙げていきます（当然ですが今後のバージョンで変わる可能性があります）。

### `TextField`（テキスト入力フィールド）

#### 使用例

- [Auth.tsx](https://github.com/hmatsu47/pgunconf-sample-app/blob/main/src/Auth.tsx#L105)（105 行目〜）

```tsx:Auth.tsx（105行目〜）
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

- [setFocus.ts](https://github.com/hmatsu47/pgunconf-sample-app/blob/main/src/commons/setFocus.ts#L3)（3 行目〜）

```typescript:setFocus.ts（3行目〜）
  const element = document.getElementById(elementId);
  element?.focus();
```

- [Auth.tsx](https://github.com/hmatsu47/pgunconf-sample-app/blob/main/src/Auth.tsx#L20)（20 行目〜）

```tsx:Auth.tsx（20行目〜）
  onMount(() => {
    setFocus('email');
  })
```

#### `multiline`に対応していない

SUID では現状[`TextareaAutosize`](https://mui.com/material-ui/react-textarea-autosize/)に対応していないので代わりに`TextField`で [Multiline](https://mui.com/material-ui/react-text-field/#multiline) を使おうと思ったのですが、`multiline`・`row`などには対応していませんでした。

仕方なく通常の`textarea`タグを使いました。

- [EditItem.tsx](https://github.com/hmatsu47/pgunconf-sample-app/blob/main/src/EditItem.tsx#L195)（195 行目〜）

```tsx:EditItem.tsx（195行目〜）
  <textarea
    id="note"
    aria-label="Note"
    placeholder="本文を入力してください"
    onchange={(event) => {
      setNote(event.currentTarget.value);
    }}
  >
    {note()}
  </textarea>
```

:::message
リンク先のコードを見ると、すぐ上に`loading()`が`true`のときの`textarea`（空欄かつ`disabled`）が書かれていますが、これは`textarea`の再描画失敗対策です。
SolidJS のバージョンによってはこの「出し分け」がなくても正常動作しますが、
- `List.tsx`から`props`としてステート（`Signal`）`article`を渡す
  - `EditItem.tsx`側で新規投稿・投稿修正の各フィールド用の`Signal`にセットする
- `EditItem.tsx`内の各フィールド用の`Signal`を直接上書きする
の 2 通りの方法でカードおよびその内容を再描画しているのが原因で`textarea`などの状態がおかしくなることがあったのでこうしています。
（本来ならロード中画面に`Skelton`を使いつつカード全体を対象に再描画させるべきかもしれないが省略）
:::

### `Card`（カード表示）

#### 使用例

- [ViewItem.tsx](https://github.com/hmatsu47/pgunconf-sample-app/blob/main/src/ViewItem.tsx#L38)（38 行目〜）

```tsx:ViewItem.tsx（38行目〜）
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
        <Fade in={expand()}>
          <Box>
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
          </Box>
        </Fade>
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

React 用 MUI にある [Collapse API](https://mui.com/material-ui/api/collapse/) に対応していないため、上に記したコードでも SolidJS 自体が持つ [Show API](https://www.solidjs.com/docs/latest/api#%3Cshow%3E) を使って類似の処理をしています（Collapse API とは違い開閉そのもののアニメーション動作はしませんが、`Fade`を使って文字の表示だけ少し遅らせています）。

:::message
`For`のブロックを直接`Fade`の対象にはできないため、`For`を`Box`の中に入れています。
:::

```tsx:ViewItem.tsx（52行目〜：展開ボタン部分）
  <Switch fallback={<></>}>
    <Match when={!expand()}>
      <ExpandMoreIcon aria-label="expand more"/>
    </Match>
    <Match when={expand()}>
      <ExpandLessIcon aria-label="expand less"/>
    </Match>
  </Switch>
```

```tsx:ViewItem.tsx（93行目〜：実際に展開する部分）
  <Show
    when={expand()}
    fallback={<></>}
  >
    <Fade in={expand()}>
      <Box>
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
      </Box>
    </Fade>
  </Show>
```

## 使ってみた感想

バージョン 0.1.0 → 0.2.x → 0.3.x で徐々に対応コンポーネントが増えていますが、まともなプロダクトを作るには足りないコンポーネント・API がまだ少なくない印象です。

今後のバージョンアップに期待、ですね。

:::message
バージョン 0.4.0 ではリファクタリングが中心だった模様で、対応コンポーネント自体は増えませんでしたが、[`Grid`](https://suid.io/components/grid)・[`Stack`](https://suid.io/components/stack)・[`Typography`](https://suid.io/components/typography)に system properties が実装されました。
- [バージョン 0.4.0 リリース情報](https://github.com/swordev/suid/releases/tag/%40suid%2Fmaterial%400.4.0)
:::

## 参考：その他のサンプル画面

### プロフィール編集画面

![](/images/solidjs-suid-sample/account.png)

## おまけ：アバター（`Avatar`）表示

SUID 0.3.0 で [`Avatar`](https://suid.io/components/avatar) に対応したので、一覧表示とタイトルバーで使ってみました。

#### 使用例

- [ViewItem.tsx](https://github.com/hmatsu47/pgunconf-sample-app/blob/main/src/ViewItem.tsx#L68)（68 行目〜）

```tsx:ViewItem.tsx（68行目〜）
<Avatar
  alt={props.article.userName}
  src={props.avatar}
  sx={{
    width: 28,
    height: 28
  }}
/>
```

`src`で指定した画像のロードに失敗した場合は自動的にフォールバックしてデフォルトのアバターアイコン（Person）が表示されます。

:::message
本家 MUI では`alt`が指定されているとその頭文字が表示されるようですが、SUID では`alt`に関係なく Person のアイコンが表示されるようです。
:::

![](/images/solidjs-suid-sample/avatar.png)
