# チュートリアル: ASP.NET Core を使用して最小限の API を作成する

Minimal API は、依存関係が最小限の HTTP API を作成するために設計されています。
ASP.NET Core での最小限のファイル、機能、依存関係のみを含むマイクロサービスやアプリに最適です。

このチュートリアルでは、ASP.NET Core を使用した最小限の API の構築の基本について説明します。

## Overview

| API                     | Description                           | リクエストの本文   | 応答本文             |
| ----------------------- | ------------------------------------- | ------------------ | -------------------- |
| GET /todoitems          | すべての To Do アイテムを取得します。 | None               | To Do アイテムの配列 |
| GET /todoitems/complete | 完了した To Do 項目を取得します。     | None               | To Do アイテムの配列 |
| GET /todoitems/{id}     | ID でアイテムを取得します。           | None               | やることリスト項目   |
| POST /todoitems         | 新しいアイテムを追加します。          | やることリスト項目 | やることリスト項目   |
| PUT /todoitems/{id}     | 既存のアイテムを更新します。          | やることリスト項目 | None                 |
| PATCH /todoitems/{id}   | アイテムを部分的に更新する            | 部分的なタスク項目 | None                 |
| DELETE /todoitems/{id}  | アイテムを削除します。                | None               | None                 |
