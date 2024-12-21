---
title: "bolt.new でフルスタックtodoアプリ"
emoji: "⚡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [bolt]
published: true
---

# はじめに

https://bolt.new を使って、TODO アプリを作りました。
![](/images/bolt.gif)

# プロンプト

以下のようなプロンプトで生成しました。

```
バックエンドにsqliteを使ったAPIを提供し、フロントエンドでそのAPIにアクセスして表示するtodo アプリ
```

sqlite 使ってくれなかったので、さらに伝えてみました。

```
server/index.jsを、sqliteを使ってtodoデータを保存できるようにして
```

# bolt.new の画面

こんな感じでチャット画面とプレビュー画面が見れます。
![](/images/2024-11-15-15-06-28.png)

コードも見れます。
今回、バックエンドとフロントエンドを別で動くようにして Web API で通信できるようにしました。

バックエンドコードの実行は bolt がやってくれてました。
![](/images/2024-11-15-15-11-51.png)

フロントエンドの実行は、bolt はしてくれなかったようなので、自分でターミナルを追加して実行しました。
![](/images/2024-11-15-15-11-04.png)

バックエンドとフロントエンドを分離したので、ポートを指定することでどちらのサービスを表示するか切り替えられるようになってました。
![](/images/2024-11-15-15-10-10.png)

# コード

以下、主要なコードの抜粋です。
指定しないと、バックエンドは js で express になりました。
構造化はされてませんでした。
エラーハンドリングはしてました。

## sqlite アクセス

直接テーブル作成 SQL が書かれてました。

```js
import sqlite3 from "sqlite3";
import { open } from "sqlite";

// Initialize database connection
export async function initializeDB() {
  const db = await open({
    filename: "todos.db",
    driver: sqlite3.Database,
  });

  // Create todos table if it doesn't exist
  await db.exec(`
    CREATE TABLE IF NOT EXISTS todos (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      title TEXT NOT NULL,
      completed BOOLEAN NOT NULL DEFAULT 0,
      created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )
  `);

  return db;
}
```

## API

update が改善の余地アリかもです。

```js
import express from "express";
import cors from "cors";
import { initializeDB } from "./db.js";

const app = express();
app.use(cors());
app.use(express.json());

let db;

// Initialize database connection
initializeDB()
  .then((database) => {
    db = database;
    console.log("Database connected successfully");
  })
  .catch((err) => {
    console.error("Failed to initialize database:", err);
    process.exit(1);
  });

// Get all todos
app.get("/api/todos", async (req, res) => {
  try {
    const todos = await db.all("SELECT * FROM todos ORDER BY created_at DESC");
    res.json(todos);
  } catch (err) {
    console.error("Error fetching todos:", err);
    res.status(500).json({ error: "Failed to fetch todos" });
  }
});

// Create todo
app.post("/api/todos", async (req, res) => {
  const { title } = req.body;
  try {
    const result = await db.run(
      "INSERT INTO todos (title, completed) VALUES (?, ?)",
      [title, false]
    );
    const todo = await db.get(
      "SELECT * FROM todos WHERE id = ?",
      result.lastID
    );
    res.json(todo);
  } catch (err) {
    console.error("Error creating todo:", err);
    res.status(500).json({ error: "Failed to create todo" });
  }
});

// Toggle todo completion
app.patch("/api/todos/:id", async (req, res) => {
  const { id } = req.params;
  try {
    const todo = await db.get("SELECT completed FROM todos WHERE id = ?", id);
    if (!todo) {
      return res.status(404).json({ error: "Todo not found" });
    }

    await db.run("UPDATE todos SET completed = ? WHERE id = ?", [
      !todo.completed,
      id,
    ]);

    const updatedTodo = await db.get("SELECT * FROM todos WHERE id = ?", id);
    res.json(updatedTodo);
  } catch (err) {
    console.error("Error updating todo:", err);
    res.status(500).json({ error: "Failed to update todo" });
  }
});

// Delete todo
app.delete("/api/todos/:id", async (req, res) => {
  const { id } = req.params;
  try {
    const result = await db.run("DELETE FROM todos WHERE id = ?", id);
    if (result.changes === 0) {
      return res.status(404).json({ error: "Todo not found" });
    }
    res.json({ success: true });
  } catch (err) {
    console.error("Error deleting todo:", err);
    res.status(500).json({ error: "Failed to delete todo" });
  }
});

app.listen(3000, () => {
  console.log("Server running on http://localhost:3000");
});
```

## API call

エラー等も考慮されてます。

```tsx
import React, { useState, useEffect } from "react";
import { TodoItem } from "./components/TodoItem";
import { TodoForm } from "./components/TodoForm";
import { Loader2 } from "lucide-react";

interface Todo {
  id: number;
  title: string;
  completed: boolean;
  created_at: string;
}

function App() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTodos();
  }, []);

  const fetchTodos = async () => {
    try {
      setLoading(true);
      const response = await fetch("http://localhost:3000/api/todos");
      if (!response.ok) throw new Error("Failed to fetch todos");
      const data = await response.json();
      setTodos(data);
      setError(null);
    } catch (err) {
      setError("Failed to load todos. Please ensure the server is running.");
    } finally {
      setLoading(false);
    }
  };

  const addTodo = async (title: string) => {
    try {
      const response = await fetch("http://localhost:3000/api/todos", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ title }),
      });
      if (!response.ok) throw new Error("Failed to add todo");
      const todo = await response.json();
      setTodos([todo, ...todos]);
      setError(null);
    } catch (err) {
      setError("Failed to add todo. Please try again.");
    }
  };

  const toggleTodo = async (id: number) => {
    try {
      const response = await fetch(`http://localhost:3000/api/todos/${id}`, {
        method: "PATCH",
      });
      if (!response.ok) throw new Error("Failed to update todo");
      const updatedTodo = await response.json();
      setTodos(todos.map((todo) => (todo.id === id ? updatedTodo : todo)));
      setError(null);
    } catch (err) {
      setError("Failed to update todo. Please try again.");
    }
  };

  const deleteTodo = async (id: number) => {
    try {
      const response = await fetch(`http://localhost:3000/api/todos/${id}`, {
        method: "DELETE",
      });
      if (!response.ok) throw new Error("Failed to delete todo");
      setTodos(todos.filter((todo) => todo.id !== id));
      setError(null);
    } catch (err) {
      setError("Failed to delete todo. Please try again.");
    }
  };

  if (loading) {
    return (
      <div className="min-h-screen bg-gradient-to-br from-indigo-500 to-purple-600 flex items-center justify-center">
        <div className="bg-white p-8 rounded-lg shadow-xl">
          <Loader2 className="w-8 h-8 animate-spin text-indigo-600" />
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gradient-to-br from-indigo-500 to-purple-600 py-12 px-4">
      <div className="max-w-3xl mx-auto">
        <div className="bg-white rounded-lg shadow-xl overflow-hidden">
          <div className="p-8">
            <h1 className="text-4xl font-bold text-gray-800 mb-8">Todo App</h1>

            {error && (
              <div className="mb-4 p-4 bg-red-50 border border-red-100 text-red-600 rounded-lg flex items-center gap-2">
                <span className="flex-1">{error}</span>
                <button
                  onClick={fetchTodos}
                  className="px-3 py-1 text-sm bg-red-100 hover:bg-red-200 rounded-md transition-colors duration-200"
                >
                  Retry
                </button>
              </div>
            )}

            <TodoForm onSubmit={addTodo} />

            <div className="space-y-4">
              {todos.map((todo) => (
                <TodoItem
                  key={todo.id}
                  {...todo}
                  onToggle={toggleTodo}
                  onDelete={deleteTodo}
                />
              ))}
              {todos.length === 0 && !error && (
                <p className="text-center text-gray-500 py-4">
                  No todos yet. Add one above!
                </p>
              )}
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}

export default App;
```

# デプロイ（できなかった）

デプロイしようとしたら下記のように言われて無理でした。バックエンドはデプロイできないみたい。bolt.new はフルスタック！みたいな記事が多いけど、このコードを Netlify へのデプロイするのはできないみたいです。ブラウザ上でやってる分には、WebContainer API 上でコンテナ立てて、node が動いてるから、バックエンドも開発できるけど、Netlify にデプロイしようと思うと、ブラウザに配信される html, javascript, css になって node は動かせないって感じかと思います。

もしかしたら、そのうち行けるようになるんかな。スピード早すぎるから記事書いてる間にもういけてる可能性も。。。

以下、チャットが返してくれたメッセージ。

```
I apologize, but I cannot deploy this application as is. This is a full-stack application with a Node.js backend and SQLite database, which cannot be deployed to static hosting providers like Netlify.

For deployment, you have a few options:

Split the application:

Deploy only the frontend to Netlify
Deploy the backend separately to a platform that supports Node.js (like Heroku, Railway, or Render)
Convert to a static application:

Remove the backend
Use local storage or a cloud database service instead of SQLite
Would you like me to help you with either of these approaches? I can:

Modify the app to be frontend-only using local storage for persistence
Keep the current architecture and deploy only the frontend, and I can guide you on deploying the backend separately
Please let me know which approach you'd prefer to take.
```

# おわりに

そのうち、ちゃんと Next.js で試してみたい。試そうとしたら、トークン制限が来た。。。

![](/images/2024-11-15-15-24-37.png)
