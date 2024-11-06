---
title: "Rust x WebAssembly x Yew でReactみたいに TODOアプリを作る"
emoji: "🦀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [rust, wasm, yew, todo]
published: true
---

# はじめに

最近、Rust x WebAssembly で React みたいなフロントエンドフレームワーク Yew の存在を知りました。調べてみると以下の記事が素晴らしかったので、まずは試してみて、プラスアルファで削除ボタンと API コールをやってみました。

以下の記事を試した後、API 呼び出しもしてみたいなぁと思った方に本記事も参考にしてもらえたら嬉しいです。

https://zenn.dev/azukiazusa/articles/rust-base-web-front-fremework-yew

# Yew って？

まず、Yew についてですが、Rust で React みたいにフロントエンドをかけるフレームワークです。
use_state フックがあったり、コンポーネントベースで HTML 要素（っぽいもの？）を生成できたりするので、React のような感覚で使うことができます。
特徴は以下の通りです。Rust の良いところと React の良いところを組み合わせた感じです。

- **コンポーネントベース**: Yew では UI を独立したコンポーネントとして作成し、それらを組み合わせてアプリケーションを構築します。
- **リアクティブな状態管理**: Yew では、状態が変更されると必要な部分だけが再描画されるため、効率的な UI 更新が可能です。
- **HTML マクロ**: Yew は、Rust 内で HTML 構文を記述するためのマクロを提供しており、JSX のように HTML を Rust コード内に埋め込むことができます。
- **Rust の安全性とパフォーマンス**: Yew は Rust のメリットを活かして、コンパイル時の型チェックやメモリ管理がしっかりしているため、バグの少ないコードが書けます。

:::message
「TypeScript でええやん」ってのは多分言ったらあかんやつです。
:::

# 作ったもの

こんなものを作りました。裏で API も呼び出してます。わかりにくいですが、途中でリロードもしてます。

![](/images/todo.gif)

# コード

https://github.com/optimisuke/hello-yew-todo

コードはここに置いてます。clone してぜひ実行してみてください。動かなかったら、連絡ください。

# 差分

「はじめに」で紹介した以下の記事との差分を紹介していきます。

https://zenn.dev/azukiazusa/articles/rust-base-web-front-fremework-yew

## 削除ボタン

まず、削除ボタンのみ実装しました。下記ブランチ参照です。

https://github.com/optimisuke/hello-yew-todo/tree/add-delete-button

`main.rs`で、以下のようなコールバックを準備しました。指定した id 以外の todo をセットしてます。

```rust:src/main.rs
    // 削除処理のコールバック
    let on_delete = {
        let todo_items = todo_items.clone();
        Callback::from(move |id: usize| {
            todo_items.set(
                todo_items
                    .iter()
                    .cloned()
                    .filter(|todo| todo.id != id)
                    .collect(),
            );
        })
    };
```

上記コールバックは、`todo_item.rs`の削除ボタンで呼び出してます。

```rust:src/components/todo/todo_item.rs
#[function_component(TodoItem)]
pub fn todo_item(props: &TodoItemProps) -> Html {
    let on_delete = props.on_delete.clone();

    html! {
      <li class="list-group-item d-flex justify-content-between align-items-center">
      <div>
          <input class="form-check-input me-2" type="checkbox" checked={props.completed} />
          {&props.title}
      </div>
      <button type="button" class="btn btn-danger btn-sm" onclick={Callback::from(move |_| on_delete.emit(()))}>{"削除"}</button>
      </li>
    }
}
```

## API 呼び出し

API は gloo ってのを使いました。他の API client の reqwest だとうまく動かず、こちらを使いました。Yew は Wasm ベースで、reqwest だと Wasm に対応してないみたいです。

https://github.com/rustwasm/gloo

```rust:src/api.rs
// src/api.rs

use crate::{components::todo::types::Todo, env::API_BASE_URL};
use gloo_net::http::Request;
use serde::{Deserialize, Serialize};
use std::error::Error;

fn get_api_base_url() -> &'static str {
    API_BASE_URL
}

pub async fn fetch_todos() -> Result<Vec<Todo>, Box<dyn Error>> {
    let url = format!("{}/todos", get_api_base_url());

    let response = Request::get(&url).send().await?;

    // ステータスコードが成功でない場合、エラーメッセージを返す
    if !response.ok() {
        return Err(format!("Failed to fetch todos: Status {}", response.status()).into());
    }

    let todos = response
        .json::<Vec<Todo>>()
        .await
        .map_err(|err| format!("Failed to parse todos JSON: {}", err))?;

    Ok(todos)
}

#[derive(Debug, Deserialize, Serialize)]
struct NewTodo {
    pub title: String,
    pub completed: bool,
}

pub async fn post_todo(title: &str) -> Result<Todo, Box<dyn Error>> {
    let url = format!("{}/todos", get_api_base_url());
    let new_todo = NewTodo {
        title: title.to_string(),
        completed: false,
    };

    let response = Request::post(&url).json(&new_todo)?.send().await?;

    // ステータスコードが成功でない場合、エラーメッセージを返す
    if !response.ok() {
        return Err(format!("Failed to post todo: Status {}", response.status()).into());
    }

    let todo = response
        .json::<Todo>()
        .await
        .map_err(|err| format!("Failed to parse posted todo JSON: {}", err))?;

    Ok(todo)
}

#[derive(Debug, Deserialize, Serialize)]
struct UpdateTodo {
    pub title: String,
    pub completed: bool,
}

pub async fn update_todo(id: &str, title: &str, completed: bool) -> Result<Todo, Box<dyn Error>> {
    let url = format!("{}/todos/{}", get_api_base_url(), id);
    let update_data = UpdateTodo {
        title: title.to_string(),
        completed,
    };

    let response = Request::put(&url).json(&update_data)?.send().await?;

    // ステータスコードが成功でない場合はエラーメッセージを返す
    if !response.ok() {
        return Err(format!("Failed to update todo: Status {}", response.status()).into());
    }

    let updated_todo = response
        .json::<Todo>()
        .await
        .map_err(|err| format!("Failed to parse updated todo JSON: {}", err))?;

    Ok(updated_todo)
}

pub async fn delete_todo(id: String) -> Result<(), Box<dyn Error>> {
    let url = format!("{}/todos/{}", get_api_base_url(), id);

    let response = Request::delete(&url).send().await?;

    // ステータスコードが成功でない場合、エラーメッセージを返す
    if !response.ok() {
        return Err(format!("Failed to delete todo: Status {}", response.status()).into());
    }

    Ok(())
}
```

## 画面の更新

`main.rs`で API 呼び出し関数を使い、それぞれの画面に表示するデータを更新しています。少し長いですが、以下の通りです。

```rust:main.rs
use crate::api::{delete_todo, fetch_todos, post_todo};
use api::update_todo;
use components::header::Header;
use components::todo::todo_form::TodoForm;
use components::todo::todo_list::TodoList;
use components::todo::types::Todo;
use yew::prelude::*;

mod api;
mod components;
mod env;

#[function_component(App)]
fn app() -> Html {
    let todo_items = use_state(|| Vec::<Todo>::new());

    // 初回レンダリング時に API からデータを取得
    {
        let todo_items = todo_items.clone();
        use_effect_with_deps(
            move |_| {
                wasm_bindgen_futures::spawn_local(async move {
                    match fetch_todos().await {
                        Ok(todos) => todo_items.set(todos),
                        Err(err) => log::error!("Failed to fetch todos: {}", err),
                    }
                });
                || ()
            },
            (),
        );
    }

    let on_add = {
        let todo_items = todo_items.clone();
        Callback::from(move |title: String| {
            let todo_items = todo_items.clone();
            wasm_bindgen_futures::spawn_local(async move {
                match post_todo(&title).await {
                    Ok(new_todo) => {
                        let mut current_todo_items = (*todo_items).clone();
                        current_todo_items.push(new_todo); // 追加された Todo を直接追加
                        todo_items.set(current_todo_items);
                    }
                    Err(err) => log::error!("Failed to add todo: {}", err),
                }
            });
        })
    };

    // on_toggle コールバックで completed 状態を更新し、API に送信
    let on_toggle = {
        let todos = todo_items.clone();
        Callback::from(move |id: String| {
            let todos = todos.clone();
            wasm_bindgen_futures::spawn_local(async move {
                if let Some(todo) = todos.iter().cloned().find(|todo| todo.id == id) {
                    let new_completed = !todo.completed;
                    match update_todo(&id, &todo.title, new_completed).await {
                        Ok(updated_todo) => {
                            let mut updated_todos = (*todos).clone();
                            if let Some(todo) = updated_todos.iter_mut().find(|todo| todo.id == id)
                            {
                                todo.completed = updated_todo.completed;
                            }
                            todos.set(updated_todos);
                        }
                        Err(err) => log::error!("Failed to update todo: {}", err),
                    }
                }
            });
        })
    };

    let on_delete = {
        let todo_items = todo_items.clone();
        Callback::from(move |id: String| {
            // id を String として受け取る
            let todo_items = todo_items.clone();
            wasm_bindgen_futures::spawn_local(async move {
                match delete_todo(id.clone()).await {
                    Ok(_) => {
                        todo_items.set(
                            todo_items
                                .iter()
                                .cloned()
                                .filter(|todo| todo.id != id)
                                .collect(),
                        );
                    }
                    Err(err) => log::error!("Failed to delete todo: {}", err),
                }
            });
        })
    };

    html! {
      <>
        <Header />
        <main class="container-fluid mt-2">
          <TodoForm {on_add} />
          <TodoList todo_items={(*todo_items).clone()} on_toggle={on_toggle} on_delete={on_delete} />
        </main>
      </>
    }
}

fn main() {
    yew::start_app::<App>();
    wasm_logger::init(wasm_logger::Config::default());
}
```

## その他

その他は、下記リポジトリ参照です。必要最小限にしてるつもりなので clone してコードを見たり実行したり試してみてください。Props として渡しているところや、`derive`や`function_component`の部分が注目ポイントかなと思います。

https://github.com/optimisuke/hello-yew-todo/tree/main

## API サーバー

API サーバーも Rust で書いて試しました。以下の記事参照です。よくあるインターフェースだと思うので、OpenAPI でモックサーバーをたてても良いし、他の言語で作っても良いし、自由に準備してくれたらいいかと思います。おすすめは Rust の以下の記事です。

https://zenn.dev/optimisuke/articles/947d5208db2b0e

# 動かし方

これでとりあえず動きます。

```bash
trunk serve --port 3000
```

# おわりに

何度も言いますが、以下の記事がとても良かったです。ぜひ試して欲しいです。

https://zenn.dev/azukiazusa/articles/rust-base-web-front-fremework-yew

本記事だけでは説明不十分のところもあるかと思いますので、下記リポジトリや適宜公式サイト等を参考にしながら、API 呼び出し等を試してもらえたらと思います。

https://github.com/optimisuke/hello-yew-todo/tree/main
