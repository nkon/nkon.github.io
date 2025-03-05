---
layout: post
title: Tauri - Todoアプリ(SQLプラグイン版)
category: blog
tags: rust tauri react javascript
---

Tauriの練習問題として簡単なToDoアプリを作る。

* [セットアップ](../Tauri-Setup/)
* [簡単なアプリ(MP3プレイヤー)](../Tauri-Player1/)
* [Todo(React版)](../Tauri-Todo/)
* [Todo(rusqlite版)](../Tauri-Todo-Rusqlite/)
* Todo(SQLプラグイン版)(本記事)

## Tauri SQL plugin

[rusqlite版](Tauri-Todo-Rusqlite.md)はRust側でデータハンドリングを実装できて、GUIと分離されたアーキテクチャが美しい。しかしモバイル環境(iOS, Android)では動かない。[Tauri SQL plugin](https://tauri.app/ja/plugin/sql/)は、公式からもモバイル環境がサポートされている。

一方、Tauri SQL pluginを使う場合は、SQLの呼び出しや、そのデータのハンドリングはReact側で行うことになる。バックエンドにRustがある意味は？と思うのだが、クロスプラットフォームのアプリ基盤としては十分に役に立つものだ。

### プラグインのインストール

[公式のガイドにしたがってSQLプラグインをインストール](https://v2.tauri.app/plugin/sql/)。

その後に、Rustディレクトリ(src-tauri)で、`tauri-plugin-sql`をSQLiteフィーチャー付きで`cargo add`しておく。

```
❯ pnpm tauri add sql
❯ cd src-tauri
❯ cargo add tauri-plugin-sql --features sqlite
```

また、`src-tauri/tauri.conf.json`にデータベースのパスをセットしておく。

```
  "plugins": {
    "sql": {
      "default": {
        "type": "SQLite",
        "path": "./todoP.db"
      }
    }
  }
```

この場合、データベースファイルは、ソースツリーの中ではなく、OS標準の設定ディレクトリの中に作られる。MacOSの場合は、`~/Library/Application Support/com.tauri-app.app/todoP.db`となる。`~/Library/Application Support/`がOS標準の設定ディレクトリ。Rustの場合は[`dirs`](https://docs.rs/dirs/latest/dirs/)クレートがOSの違いを吸収してくれる。

| Platform | Value                               | Example                                    |
| -------- | ----------------------------------- | ------------------------------------------ |
| Linux    | `$XDG_CONFIG_HOME or $HOME/.config` | `/home/alice/.config`                      |
| macOS    | `$HOME/Library/Application Support` | `/Users/Alice/Library/Application Support` |
| Windows  | `{FOLDERID_RoamingAppData}`         | C:\Users\Alice\AppData\Roaming             |

### SQLの呼び出し

SQLプラグインを使う場合は、React側でテーブル定義、SQL生成、結果のハンドリングを行う必要がある。

注意: Jykellのレンダリングのバグを回避するために二重波括弧は間にスペースが入っている。実際のコードでは省くこと。

```javascript
import React, { useState, useEffect } from "react";
import Database from '@tauri-apps/plugin-sql';

const ToDoSqliteP = () => {
    const [db, setDB] = useState(null);
    const [todos, setTodos] = useState([]);
    const [input, setInput] = useState('');

    useEffect(() => {
        const initDatabase = async () => {
            // database is created in ~/Library/Application Support/com.tauri-app.app/todoP.db
            const database = await Database.load("sqlite:todoP.db");
            setDB(database);
            await createTable(database);
            await loadTodos(database);
        };
        initDatabase();
    }, []);

    const createTable = async (database) => {
        await database.execute(`
            CREATE TABLE IF NOT EXISTS todos(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            task TEXT NOT NULL,
            completed BOOLEAN NOT NULL DEFAULT 0
        )`);
    }

    const loadTodos = async (database = db) => {
        if (!database) return;
        const todos = await database.select("SELECT * from todos");
        setTodos(todos);
    }

    const addTodo = async () => {
        if (!db) return;
        if (input.trim() === "") return;
        await db.execute("INSERT INTO todos (task, completed) VALUES (?, ?)", [input, 0]);
        setInput("");
        loadTodos();
    }

    const toggleTodo = async (id) => {
        if (!db) return;
        await db.execute("UPDATE todos SET completed = NOT completed WHERE id = ?", [id]);
        loadTodos();
    }

    const removeTodo = async (id) => {
        if (!db) return;
        await db.execute("DELETE FROM todos WHERE id = ?", [id]);
        loadTodos();
    }

    return (
        <div style={ {textAlign: 'center', padding: '2rem' } }>
            <h1>TODO App SQlite Plugin</h1>
            <input
                type="text"
                value={input}
                onChange={(e) => setInput(e.target.value)}
                placeholder="Enter a todo task"
            />
            <button onClick={addTodo}>Add</button>
            <ul>
                {todos.map((todo) => (
                    <li key={todo.id}>
                        <span
                            style={ { textDecoration: todo.completed ? "line-through" : "none", } }
                            onClick={() => toggleTodo(todo.id)}>{todo.task}</span> <button onClick={() => removeTodo(todo.id)}>Delete</button>
                    </li>
                ))}
            </ul>
        </div>
    );
}

export default ToDoSqliteP;
```

* `CREATE TABLE`は、既存のものがあればそれが使われ、存在しなえれば新規作成される。
* `async (databases)`、`await database.execute()`のような非同期呼び出しとなる。
* SQLプラグイン版は、iOS、Androidでも正常に動く。シミュレータを再起動すれば、以前のデータが残っていることが確認できる。


MP3プレイヤーと同様に、これをApp.jsxの中から呼び出してやれば良い。

見た目、機能はRusqlite版と同様。


* [セットアップ](../Tauri-Setup/)
* [簡単なアプリ(MP3プレイヤー)](../Tauri-Player1/)
* [Todo(React版)](../Tauri-Todo/)
* [Todo(rusqlite版)](../Tauri-Todo-Rusqlite/)
* Todo(SQLプラグイン版)(本記事)