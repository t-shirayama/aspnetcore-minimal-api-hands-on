# チュートリアル: ASP.NET Core Minimal API で TODO API を作る

このメモは、Microsoft Learn の「ASP.NET Core を使用して最小限の API を作成する」を参考に、このリポジトリの `TodoApi` プロジェクト向けに要点を整理したものです。

参考: https://learn.microsoft.com/ja-jp/aspnet/core/tutorials/min-web-api?view=aspnetcore-10.0&tabs=visual-studio-code

## 何を作るか

TODO アイテムを登録、取得、更新、削除できる小さな Web API を作ります。
ASP.NET Core の Minimal API を使うため、コントローラークラスを作らず、`Program.cs` にエンドポイントを直接定義します。

このチュートリアルで扱う主な要素は次の 5 つです。

- `Todo`: TODO アイテムを表すモデル
- `TodoItemDTO`: API の入出力で使う DTO モデル
- `TodoDb`: TODO アイテムを保存する EF Core のデータベースコンテキスト
- `Program.cs`: API の起動設定とエンドポイント定義
- Swagger UI: ブラウザから API を試すための画面

## この API の一覧

| API                     | 役割                              | リクエスト本文 | 主なレスポンス |
| ----------------------- | --------------------------------- | -------------- | -------------- |
| `GET /todoitems`        | すべての TODO を取得する          | なし           | TODO DTO の配列 |
| `GET /todoitems/complete` | 完了済みの TODO だけ取得する    | なし           | TODO DTO の配列 |
| `GET /todoitems/{id}`   | 指定した ID の TODO を取得する    | なし           | TODO DTO または 404 |
| `POST /todoitems`       | 新しい TODO を追加する            | TODO DTO       | 作成された TODO DTO |
| `PUT /todoitems/{id}`   | 指定した ID の TODO を更新する    | TODO DTO       | 204 または 404 |
| `DELETE /todoitems/{id}` | 指定した ID の TODO を削除する   | なし           | 204 または 404 |

この README では、最初に Minimal API の基本形を確認し、その後で `MapGroup`、`TypedResults`、DTO を使った最終版の形に整理します。
先に素朴な書き方を見てから改善版を見る流れにすると、最終版のコードがなぜその形になっているのか追いやすくなります。

## 前提条件

- Visual Studio Code
- C# Dev Kit 拡張機能
- .NET 10 SDK

SDK が入っているか確認します。

```sh
dotnet --version
```

## プロジェクトを作成する

Minimal API の空に近い Web プロジェクトを作ります。

```sh
dotnet new web -o TodoApi
cd TodoApi
code -r .
```

`dotnet new web` で作られる最初の `Program.cs` は、とても小さい構成です。

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.Run();
```

ここで重要なのは、Minimal API では `app.MapGet(...)` のように HTTP メソッドと URL を直接対応付けることです。

## 必要な NuGet パッケージ

TODO データの保存と API テスト画面のために、次のパッケージを使います。

```sh
dotnet add package Microsoft.EntityFrameworkCore.InMemory
dotnet add package Microsoft.AspNetCore.Diagnostics.EntityFrameworkCore
dotnet add package NSwag.AspNetCore
```

各パッケージの役割は次のとおりです。

- `Microsoft.EntityFrameworkCore.InMemory`: メモリ上にデータベースを作るための EF Core プロバイダー
- `Microsoft.AspNetCore.Diagnostics.EntityFrameworkCore`: 開発中に EF Core 関連のエラーを見やすくする診断機能
- `NSwag.AspNetCore`: OpenAPI 定義と Swagger UI を生成するためのライブラリ

インメモリデータベースは学習用には便利ですが、アプリを終了するとデータは消えます。
本番用途では SQL Server、PostgreSQL、SQLite などの永続化できるデータベースを使います。

## Todo モデル

`Todo.cs` は、API で扱う TODO アイテムの形を表します。

```csharp
public class Todo
{
    public int Id { get; set; }
    public string? Name { get; set; }
    public bool IsComplete { get; set; }
}
```

各プロパティの意味は次のとおりです。

- `Id`: TODO を一意に識別する番号
- `Name`: TODO の内容
- `IsComplete`: 完了しているかどうか

`string?` は、`Name` が `null` になる可能性もあることを表しています。

## TodoDb クラス

`TodoDb.cs` は、EF Core が TODO データを扱うための入口です。

```csharp
using Microsoft.EntityFrameworkCore;

class TodoDb : DbContext
{
    public TodoDb(DbContextOptions<TodoDb> options)
        : base(options) { }

    public DbSet<Todo> Todos => Set<Todo>();
}
```

`DbContext` は、データベースとのやり取りをまとめるクラスです。
`DbSet<Todo>` は、データベース内の TODO テーブルのようなものだと考えると分かりやすいです。

このクラスの中で特に分かりにくいのは、次の 2 か所です。

```csharp
public TodoDb(DbContextOptions<TodoDb> options)
    : base(options) { }
```

これは `TodoDb` のコンストラクターです。
`TodoDb` が作られるときに、EF Core の設定情報を受け取っています。

`DbContextOptions<TodoDb>` には、たとえば次のような情報が入ります。

- どのデータベースを使うか
- 接続先はどこか
- EF Core がどのように `TodoDb` を動かすか

このプロジェクトでは、`Program.cs` の次の行で設定しています。

```csharp
builder.Services.AddDbContext<TodoDb>(opt => opt.UseInMemoryDatabase("TodoList"));
```

ここで「`TodoDb` ではインメモリデータベースの `TodoList` を使う」と指定しています。
その設定が `DbContextOptions<TodoDb> options` として `TodoDb` のコンストラクターに渡されます。

次の `: base(options)` は、受け取った設定を親クラスである `DbContext` に渡す処理です。

```csharp
: base(options)
```

`TodoDb` は `DbContext` を継承しています。
実際にデータベース接続や変更追跡などの基本機能を持っているのは親クラスの `DbContext` です。
そのため、`TodoDb` が受け取った設定を `DbContext` に渡して、「この設定で動いてください」と伝えています。

コンストラクターの本体が空になっているのは、追加で書く処理がないからです。

```csharp
{ }
```

つまり、このコンストラクター全体は次のような意味です。

```text
TodoDb が作られるときに EF Core の設定を受け取り、
その設定を親クラスの DbContext に渡す。
```

もう 1 つは、次のプロパティです。

```csharp
public DbSet<Todo> Todos => Set<Todo>();
```

`DbSet<Todo>` は、`Todo` データの集まりを表します。
SQL データベースで考えるなら、`Todos` テーブルのようなものです。

このプロパティがあることで、`Program.cs` から次のように TODO データを操作できます。

```csharp
db.Todos.ToListAsync();
db.Todos.FindAsync(id);
db.Todos.Add(todo);
db.Todos.Remove(todo);
```

右側の `Set<Todo>()` は、親クラス `DbContext` が持っているメソッドです。
`Todo` 型に対応する `DbSet<Todo>` を取得しています。

つまり、次の 1 行は、

```csharp
public DbSet<Todo> Todos => Set<Todo>();
```

次のような意味です。

```text
Todo 型のデータを操作するための入口を Todos という名前で用意する。
```

`=>` は式形式のプロパティです。
短く書くための構文で、次のように書くのと近い意味です。

```csharp
public DbSet<Todo> Todos
{
    get { return Set<Todo>(); }
}
```

このプロジェクトでは実際の SQL データベースではなく、メモリ上の `TodoList` というデータベースを使います。

## Program.cs の全体像

`Program.cs` では、アプリの設定、Swagger UI の設定、API エンドポイントの定義を行います。

最初に、EF Core と Swagger UI を使うためのサービスを登録します。

```csharp
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<TodoDb>(opt => opt.UseInMemoryDatabase("TodoList"));
builder.Services.AddDatabaseDeveloperPageExceptionFilter();

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddOpenApiDocument(config =>
{
    config.DocumentName = "TodoAPI";
    config.Title = "TodoAPI v1";
    config.Version = "v1";
});

var app = builder.Build();
```

ここでのポイントは、`AddDbContext<TodoDb>` によって `TodoDb` を API の処理内で受け取れるようにしていることです。
後で出てくる `async (TodoDb db) => ...` の `db` は、ASP.NET Core が DI から自動で渡してくれます。

開発環境では Swagger UI を有効にします。

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseOpenApi();
    app.UseSwaggerUi(config =>
    {
        config.DocumentTitle = "TodoAPI";
        config.Path = "/swagger";
        config.DocumentPath = "/swagger/{documentName}/swagger.json";
        config.DocExpansion = "list";
    });
}
```

これにより、アプリ起動後に `/swagger` を開くと API をブラウザ上で試せます。

## Map&lt;HttpVerb&gt; でエンドポイントを定義する

Minimal API では、`MapGet`、`MapPost`、`MapPut`、`MapDelete` のようなメソッドを使って API のエンドポイントを定義します。
Microsoft Learn ではこれらをまとめて `Map<HttpVerb>` と呼んでいます。

`HttpVerb` は HTTP メソッドのことです。
たとえば、`GET`、`POST`、`PUT`、`DELETE` などが HTTP メソッドです。

基本形は次のようになります。

```csharp
app.MapGet("/path", handler);
```

これは「`GET /path` にリクエストが来たら、`handler` の処理を実行する」という意味です。

TODO API では、次のように HTTP メソッドごとに役割を分けています。

| 書き方 | HTTP メソッド | 主な用途 | この API の例 |
| ------ | ------------- | -------- | ------------- |
| `MapGet` | `GET` | データを取得する | TODO 一覧や 1 件取得 |
| `MapPost` | `POST` | 新しいデータを作成する | TODO の追加 |
| `MapPut` | `PUT` | 既存データを置き換える、更新する | TODO の更新 |
| `MapDelete` | `DELETE` | データを削除する | TODO の削除 |
| `MapPatch` | `PATCH` | 既存データの一部だけ更新する | このリポジトリでは未実装 |

たとえば、すべての TODO を取得する API は次のように書きます。

```csharp
app.MapGet("/todoitems", async (TodoDb db) =>
    await db.Todos.ToListAsync());
```

この 1 行は、次のように読めます。

- `app.MapGet`: HTTP GET の API を作る
- `"/todoitems"`: URL は `/todoitems`
- `async (TodoDb db) => ...`: リクエストが来たときに実行する処理
- `TodoDb db`: DI から受け取るデータベースコンテキスト
- `db.Todos.ToListAsync()`: TODO を一覧で取得して返す

URL に値を含めたい場合は、`{id}` のように書きます。

```csharp
app.MapGet("/todoitems/{id}", async (int id, TodoDb db) =>
    await db.Todos.FindAsync(id)
        is Todo todo
            ? Results.Ok(todo)
            : Results.NotFound());
```

この場合、`/todoitems/1` にアクセスすると、URL の `1` が `int id` に自動で入ります。
この仕組みをルートパラメーターのバインドと呼びます。

`POST` や `PUT` のようにリクエスト本文を受け取る場合は、引数にモデルを追加します。

```csharp
app.MapPost("/todoitems", async (Todo todo, TodoDb db) =>
{
    db.Todos.Add(todo);
    await db.SaveChangesAsync();

    return Results.Created($"/todoitems/{todo.Id}", todo);
});
```

この場合、送られてきた JSON が `Todo todo` に自動で変換されます。
つまり `Map<HttpVerb>` は、URL、HTTP メソッド、受け取る値、実行する処理を 1 か所で定義するための API です。

## GET: TODO を取得する

まずは、ラムダ式と `Results` を使う素朴な形で CRUD の流れを確認します。
すべての TODO を取得するエンドポイントです。

```csharp
app.MapGet("/todoitems", async (TodoDb db) =>
    await db.Todos.ToListAsync());
```

`MapGet` は HTTP GET のリクエストを処理します。
`db.Todos.ToListAsync()` によって、保存されている TODO を配列として返します。

完了済みの TODO だけ取得する場合は、`Where` で条件を付けます。

```csharp
app.MapGet("/todoitems/complete", async (TodoDb db) =>
    await db.Todos.Where(t => t.IsComplete).ToListAsync());
```

`t => t.IsComplete` は「`IsComplete` が `true` の TODO だけ」という意味です。

ID を指定して 1 件だけ取得するエンドポイントです。

```csharp
app.MapGet("/todoitems/{id}", async (int id, TodoDb db) =>
    await db.Todos.FindAsync(id)
        is Todo todo
            ? Results.Ok(todo)
            : Results.NotFound());
```

`/todoitems/{id}` の `{id}` は URL の一部をパラメーターとして受け取る書き方です。
該当する TODO があれば `200 OK`、なければ `404 Not Found` を返します。

## POST: TODO を追加する

素朴な形で、新しい TODO を追加するエンドポイントです。

```csharp
app.MapPost("/todoitems", async (Todo todo, TodoDb db) =>
{
    db.Todos.Add(todo);
    await db.SaveChangesAsync();

    return Results.Created($"/todoitems/{todo.Id}", todo);
});
```

このコードの流れは次のとおりです。

1. リクエスト本文の JSON が `Todo todo` に変換される
2. `db.Todos.Add(todo)` で追加対象として登録する
3. `await db.SaveChangesAsync()` でデータベースに保存する
4. `Results.Created(...)` で `201 Created` を返す

たとえば、次のような JSON を送ると TODO が作成されます。

```json
{
  "name": "ASP.NET Core を学ぶ",
  "isComplete": false
}
```

`Results.Created($"/todoitems/{todo.Id}", todo)` は、作成されたリソースの URL と作成後の TODO を返します。
REST API では、作成成功時に `201 Created` を返すのが一般的です。

## PUT: TODO を更新する

素朴な形で、既存の TODO を更新するエンドポイントです。

```csharp
app.MapPut("/todoitems/{id}", async (int id, Todo inputTodo, TodoDb db) =>
{
    var todo = await db.Todos.FindAsync(id);

    if (todo is null) return Results.NotFound();

    todo.Name = inputTodo.Name;
    todo.IsComplete = inputTodo.IsComplete;

    await db.SaveChangesAsync();

    return Results.NoContent();
});
```

まず `FindAsync(id)` で更新対象を探します。
対象がなければ `404 Not Found` を返します。

対象が見つかったら、リクエスト本文で受け取った `inputTodo` の値を既存の `todo` にコピーし、`SaveChangesAsync()` で保存します。
更新成功時は本文なしの `204 No Content` を返します。

## DELETE: TODO を削除する

素朴な形で、指定した ID の TODO を削除するエンドポイントです。

```csharp
app.MapDelete("/todoitems/{id}", async (int id, TodoDb db) =>
{
    if (await db.Todos.FindAsync(id) is Todo todo)
    {
        db.Todos.Remove(todo);
        await db.SaveChangesAsync();
        return Results.NoContent();
    }

    return Results.NotFound();
});
```

削除対象が見つかった場合だけ `Remove` して保存します。
見つからない場合は `404 Not Found` を返します。

## ラムダ式ではなくメソッドを呼び出す

ここまでの例では、処理をラムダ式で直接書いていました。
しかし `Map<HttpVerb>` の handler には、別に定義したメソッドを渡すこともできます。

```csharp
app.MapGet("/todoitems", GetAllTodos);

static async Task<List<Todo>> GetAllTodos(TodoDb db)
{
    return await db.Todos.ToListAsync();
}
```

この書き方でも、`GET /todoitems` にリクエストが来ると `GetAllTodos` メソッドが呼び出されます。
`TodoDb db` もラムダ式のときと同じように DI から自動で渡されます。

`POST` のようにリクエスト本文を受け取るメソッドも分けられます。

```csharp
app.MapPost("/todoitems", CreateTodo);

static async Task<IResult> CreateTodo(Todo todo, TodoDb db)
{
    db.Todos.Add(todo);
    await db.SaveChangesAsync();

    return Results.Created($"/todoitems/{todo.Id}", todo);
}
```

ラムダ式で書く方法と、メソッドに分ける方法のどちらも正しい書き方です。
小さい処理ならラムダ式のままでも読みやすいです。
処理が長くなったり、エンドポイントが増えたりしたら、メソッドに分けると `Program.cs` の見通しがよくなります。

## MapGroup API を使う

ここまでの素朴な例では、各エンドポイントに毎回 `/todoitems` を書いています。

```csharp
app.MapGet("/todoitems", ...);
app.MapGet("/todoitems/complete", ...);
app.MapGet("/todoitems/{id}", ...);
app.MapPost("/todoitems", ...);
app.MapPut("/todoitems/{id}", ...);
app.MapDelete("/todoitems/{id}", ...);
```

エンドポイントが増えてくると、同じ URL の先頭部分を何度も書くことになります。
このようなときに `MapGroup` を使うと、共通の URL プレフィックスをまとめられます。

```csharp
var todoItems = app.MapGroup("/todoitems");
```

この `todoItems` に対して `MapGet` や `MapPost` を定義すると、すべて `/todoitems` 配下の API になります。

たとえば、TODO API は次のように書き換えられます。

```csharp
RouteGroupBuilder todoItems = app.MapGroup("/todoitems");

todoItems.MapGet("/", GetAllTodos);
todoItems.MapGet("/complete", GetCompleteTodos);
todoItems.MapGet("/{id}", GetTodo);
todoItems.MapPost("/", CreateTodo);
todoItems.MapPut("/{id}", UpdateTodo);
todoItems.MapDelete("/{id}", DeleteTodo);
```

`todoItems.MapGet("/", GetAllTodos)` は、実際には `GET /todoitems` になります。
`todoItems.MapGet("/{id}", GetTodo)` は、実際には `GET /todoitems/{id}` になります。

この形にすると、`Program.cs` の上側では「どの URL がどのメソッドにつながるか」だけを一覧できます。
実際の処理は `GetAllTodos`、`CreateTodo`、`UpdateTodo` などのメソッド側に分けて書けます。

`MapGroup` を使う利点は次のとおりです。

- 共通の URL プレフィックスを 1 か所にまとめられる
- エンドポイントが増えても URL の構造が分かりやすい
- グループ単位で認証、タグ、フィルターなどをまとめて設定できる

たとえば Swagger UI 上でグループ名を付けたい場合は、次のように書けます。

```csharp
var todoItems = app.MapGroup("/todoitems")
    .WithTags("TodoItems");
```

このようにすると、`/todoitems` 配下の API をひとまとまりとして扱いやすくなります。

## TypedResults API を使う

これまでのコードでは、`Results.Ok(...)` や `Results.NotFound()` のように `Results` を使ってレスポンスを返していました。

```csharp
static async Task<IResult> GetTodo(int id, TodoDb db)
{
    return await db.Todos.FindAsync(id)
        is Todo todo
            ? Results.Ok(todo)
            : Results.NotFound();
}
```

`Results` は便利ですが、このメソッドの戻り値は `IResult` として扱われます。
一方、`TypedResults` を使うと、`Ok<Todo>` や `NotFound` のような具体的な結果型を返せます。

ざっくり言うと、どちらも HTTP レスポンスを作るための API です。
違いは「返す結果の型を、どれくらい C# のコード上で明確にするか」です。

| 書き方 | 返すもの | 向いている場面 |
| ------ | -------- | -------------- |
| `Results.Ok(todo)` | `IResult` として扱われる | まず動かしたい、シンプルに書きたい |
| `TypedResults.Ok(todo)` | `Ok<Todo>` として扱われる | 戻り値の種類を明確にしたい、OpenAPI 情報を分かりやすくしたい |

たとえば、`GetTodo` メソッドは「TODO があれば `200 OK`、なければ `404 Not Found`」を返します。
`Results` を使う素朴な形では、次のように `Task<IResult>` として書けます。

```csharp
static async Task<IResult> GetTodo(int id, TodoDb db)
{
    return await db.Todos.FindAsync(id)
        is Todo todo
            ? Results.Ok(todo)
            : Results.NotFound();
}
```

このコードは短くて読みやすいです。
ただし C# から見ると、`Results.Ok(todo)` も `Results.NotFound()` も大きくは `IResult` として扱われます。
つまり「このメソッドが `Ok<Todo>` と `NotFound` を返す」という情報は、コードの型としてはあまり強く残りません。

同じ処理を `TypedResults` で書くと、次のようになります。

```csharp
static async Task<Results<Ok<Todo>, NotFound>> GetTodo(int id, TodoDb db)
{
    return await db.Todos.FindAsync(id)
        is Todo todo
            ? TypedResults.Ok(todo)
            : TypedResults.NotFound();
}
```

この例では、メソッドが返す可能性のある結果を `Results<Ok<Todo>, NotFound>` として明示しています。
つまり、この API は次のどちらかを返すとコード上で分かります。

- `Ok<Todo>`: TODO が見つかった場合の `200 OK`
- `NotFound`: TODO が見つからなかった場合の `404 Not Found`

`TypedResults` を使うには、必要に応じて次の `using` を追加します。

```csharp
using Microsoft.AspNetCore.Http.HttpResults;
```

最終版のソースコードでは、DTO を返すため `Todo` ではなく `TodoItemDTO` を使って、次のような戻り値型にしています。

```csharp
static async Task<Results<Ok<TodoItemDTO>, NotFound>> GetTodo(int id, TodoDb db)
{
    return await db.Todos.FindAsync(id)
        is Todo todo
            ? TypedResults.Ok(new TodoItemDTO(todo))
            : TypedResults.NotFound();
}
```

`POST` の `CreateTodo` メソッドも、`TypedResults` を使って書けます。

```csharp
static async Task<Created<TodoItemDTO>> CreateTodo(TodoItemDTO todoItemDTO, TodoDb db)
{
    var todoItem = new Todo
    {
        IsComplete = todoItemDTO.IsComplete,
        Name = todoItemDTO.Name
    };

    db.Todos.Add(todoItem);
    await db.SaveChangesAsync();

    todoItemDTO = new TodoItemDTO(todoItem);

    return TypedResults.Created($"/todoitems/{todoItem.Id}", todoItemDTO);
}
```

`DELETE` のように、成功時は `204 No Content`、対象がない場合は `404 Not Found` を返すメソッドでは、次のように書けます。

```csharp
static async Task<Results<NoContent, NotFound>> DeleteTodo(int id, TodoDb db)
{
    if (await db.Todos.FindAsync(id) is Todo todo)
    {
        db.Todos.Remove(todo);
        await db.SaveChangesAsync();
        return TypedResults.NoContent();
    }

    return TypedResults.NotFound();
}
```

`MapGroup` 側は、これまでと同じようにメソッド名を渡すだけです。

```csharp
todoItems.MapGet("/{id}", GetTodo);
todoItems.MapPost("/", CreateTodo);
todoItems.MapDelete("/{id}", DeleteTodo);
```

`TypedResults` を使う利点は次のとおりです。

- エンドポイントが返すレスポンスの種類をコードから読み取りやすい
- コンパイル時に戻り値の型を確認しやすい
- OpenAPI のレスポンス情報がより明確になりやすい
- 大きな API でも、成功時とエラー時の戻り値を整理しやすい

学習の最初は `Results` でも十分です。
API の戻り値をより厳密に扱いたくなったら、メソッド側の戻り値を `TypedResults` に置き換えていくと理解しやすいです。

## 過剰な投稿を防止する DTO モデル

ここまでの例では、リクエスト本文を直接 `Todo` モデルで受け取っています。

```csharp
static async Task<IResult> CreateTodo(Todo todo, TodoDb db)
{
    db.Todos.Add(todo);
    await db.SaveChangesAsync();

    return Results.Created($"/todoitems/{todo.Id}", todo);
}
```

学習用の小さな API ではこれでも動きます。
ただし、実際のアプリでは「クライアントに変更してほしくないプロパティ」まで送られてしまう可能性があります。
これを過剰な投稿、または overposting と呼びます。

たとえば、将来 `Todo` に管理用のプロパティを追加したとします。

```csharp
public class Todo
{
    public int Id { get; set; }
    public string? Name { get; set; }
    public bool IsComplete { get; set; }
    public string? OwnerUserId { get; set; }
    public bool IsAdminOnly { get; set; }
}
```

この状態で `Todo` をそのまま受け取ると、クライアントが次のような JSON を送れてしまいます。

```json
{
  "name": "管理用タスク",
  "isComplete": false,
  "ownerUserId": "other-user",
  "isAdminOnly": true
}
```

API 側が意図していない項目まで受け取って保存してしまうと、データの改ざんや権限の問題につながります。
そこで使うのが DTO です。

DTO は Data Transfer Object の略で、API の入出力用に作る専用のモデルです。
データベース用の `Todo` と、クライアントに公開する形を分けるために使います。

この TODO API なら、たとえば次のような `TodoItemDTO` を作れます。

```csharp
public class TodoItemDTO
{
    public int Id { get; set; }
    public string? Name { get; set; }
    public bool IsComplete { get; set; }

    public TodoItemDTO() { }

    public TodoItemDTO(Todo todo) =>
        (Id, Name, IsComplete) = (todo.Id, todo.Name, todo.IsComplete);
}
```

`TodoItemDTO` には、API で公開したい項目だけを書きます。
もし `Todo` に `OwnerUserId` や `IsAdminOnly` があっても、DTO に含めなければクライアントとのやり取りには出てきません。

`GET` では、データベースから取得した `Todo` を DTO に変換して返します。

```csharp
static async Task<Ok<TodoItemDTO[]>> GetAllTodos(TodoDb db)
{
    return TypedResults.Ok(await db.Todos
        .Select(todo => new TodoItemDTO(todo))
        .ToArrayAsync());
}
```

完了済みの TODO だけを返す `GetCompleteTodos` でも、同じように DTO に変換して返します。

```csharp
static async Task<Ok<List<TodoItemDTO>>> GetCompleteTodos(TodoDb db)
{
    return TypedResults.Ok(await db.Todos
        .Where(t => t.IsComplete)
        .Select(todo => new TodoItemDTO(todo))
        .ToListAsync());
}
```

`POST` では、リクエスト本文を `TodoItemDTO` で受け取り、保存用の `Todo` を API 側で作ります。

```csharp
static async Task<Created<TodoItemDTO>> CreateTodo(TodoItemDTO todoItemDTO, TodoDb db)
{
    var todoItem = new Todo
    {
        Name = todoItemDTO.Name,
        IsComplete = todoItemDTO.IsComplete
    };

    db.Todos.Add(todoItem);
    await db.SaveChangesAsync();

    todoItemDTO = new TodoItemDTO(todoItem);

    return TypedResults.Created($"/todoitems/{todoItem.Id}", todoItemDTO);
}
```

この書き方では、クライアントが `Id` を送ってきても保存用の `Todo` にはコピーしていません。
`Id` はデータベース側で決まる値として扱えます。

`PUT` でも同じように、更新してよいプロパティだけをコピーします。

```csharp
static async Task<Results<NoContent, NotFound>> UpdateTodo(int id, TodoItemDTO todoItemDTO, TodoDb db)
{
    var todo = await db.Todos.FindAsync(id);

    if (todo is null) return TypedResults.NotFound();

    todo.Name = todoItemDTO.Name;
    todo.IsComplete = todoItemDTO.IsComplete;

    await db.SaveChangesAsync();

    return TypedResults.NoContent();
}
```

DTO を使う利点は次のとおりです。

- クライアントに公開する項目を制限できる
- クライアントから変更されてよい項目だけを受け取れる
- データベース用モデルと API 用モデルを分けられる
- 将来 `Todo` に内部用プロパティが増えても、API の公開範囲を守りやすい

最初は `Todo` をそのまま使うと理解しやすいです。
ただし、実際のアプリに近づけるなら、リクエストとレスポンスには DTO を使う設計が安全です。

## アプリを実行する

ここまでの内容を `Program.cs`、`Todo.cs`、`TodoDb.cs`、`TodoItemDTO.cs` に反映したら、アプリを起動して動作確認します。

HTTPS 開発証明書を信頼します。

```sh
dotnet dev-certs https --trust
```

アプリを起動します。

```sh
dotnet run --project TodoApi
```

起動後、表示された URL にアクセスします。
Swagger UI は開発環境では次のような URL で開けます。

```text
https://localhost:<port>/swagger
```

`<port>` は `dotnet run` の出力に表示されるポート番号に置き換えます。

## curl で試す

Swagger UI の代わりに、コマンドラインからも API を確認できます。
ここではポート番号を `7042` と仮定しています。実際のポートに置き換えてください。

TODO を追加します。

```sh
curl -k -X POST https://localhost:7042/todoitems \
  -H "Content-Type: application/json" \
  -d "{\"name\":\"ASP.NET Core を学ぶ\",\"isComplete\":false}"
```

すべての TODO を取得します。

```sh
curl -k https://localhost:7042/todoitems
```

TODO を完了済みに更新します。

```sh
curl -k -X PUT https://localhost:7042/todoitems/1 \
  -H "Content-Type: application/json" \
  -d "{\"name\":\"ASP.NET Core を学ぶ\",\"isComplete\":true}"
```

完了済みの TODO だけ取得します。

```sh
curl -k https://localhost:7042/todoitems/complete
```

TODO を削除します。

```sh
curl -k -X DELETE https://localhost:7042/todoitems/1
```

## 学習ポイント

このチュートリアルの重要なポイントは次のとおりです。

- Minimal API では `MapGet`、`MapPost`、`MapPut`、`MapDelete` でエンドポイントを定義する
- `Map<HttpVerb>` は、HTTP メソッドと URL と処理を対応付ける書き方
- `MapGroup` を使うと、同じ URL プレフィックスを持つエンドポイントをまとめられる
- `TypedResults` を使うと、エンドポイントの戻り値の型をより明確にできる
- URL の `{id}` はメソッド引数の `int id` に自動でバインドされる
- リクエスト本文の JSON は `Todo todo` のようなオブジェクトに自動で変換される
- `TodoDb db` は DI によって自動で渡される
- `SaveChangesAsync()` を呼ぶまで、追加、更新、削除は確定しない
- 成功、作成、未検出などの状態は `Results.Ok`、`Results.Created`、`Results.NotFound`、`Results.NoContent` で表す
- DTO を使うと、過剰な投稿を防ぎ、API で公開する項目を制限できる
- Swagger UI を使うと、API クライアントを書かなくてもブラウザから API を試せる

## 次に学ぶとよいこと

基本の CRUD に慣れたら、Microsoft Learn の続きにある内容も確認すると理解が進みます。

- `PATCH` で一部の項目だけ更新する
- インメモリデータベースから永続化できるデータベースに変更する
