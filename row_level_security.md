# Postgres Row Level Security (RLS) 概要

Postgres の Row Level Security (以下 RLS) は、テーブル全体ではなく **行単位** でアクセス制御を行う機能です。<br>
SaaS や細粒度な権限制御が必要なアプリケーションで威力を発揮します。<br>
Supabase はこの RLS を **デフォルトで有効化** する設計になっています。

---

## 1. 仕組みと基本構文

| 操作 | 意味 |
| --- | --- |
| `ALTER TABLE <table> ENABLE ROW LEVEL SECURITY;` | テーブルに RLS を有効化 |
| `CREATE POLICY <name> ON <table> FOR <cmd>` | 行ポリシーを定義<br>(`cmd`: `ALL` / `SELECT` / `INSERT` / `UPDATE` / `DELETE`) |
| `USING (<boolean_expr>)` | 既存行に対して評価されるフィルタ条件<br>(`SELECT` / `UPDATE` / `DELETE`) |
| `WITH CHECK (<boolean_expr>)` | 挿入・更新される **新しい行** に対し適用<br>(`INSERT` / `UPDATE`) |

```sql
-- 例: 自分が owner の行だけ取得できるポリシー
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
CREATE POLICY "owner_can_read" ON projects
  FOR SELECT
  USING ( owner_id = auth.uid() );

-- 例: role `general_role` は自身の行のみ、`admin_role` は全行を読み取れる
CREATE POLICY "general_read_own" ON projects
  FOR SELECT
  TO general_role
  USING ( owner_id = auth.uid() );

CREATE POLICY "admin_read_all" ON projects
  FOR SELECT
  TO admin_role
  USING ( TRUE );
```

- **権限チェックの順序**: `GRANT`/`REVOKE` → RLS ポリシー
- 複数ポリシーは **OR** で結合される (いずれかを満たせば許可)
- `AS restrictive` 句を使うと AND 振る舞いに近い厳格モードにできる (Postgres 15+)

---

## 2. Supabase 固有のヘルパ関数

### 2.1 `auth.uid()`

現在実行しているユーザの UUID を返します。最もシンプルに **行に埋め込んだ `user_id` と比較** する用途で利用します。

```sql
USING ( user_id = auth.uid() )
```

### 2.2 `auth.jwt()`

ユーザの JSON Web Token を **JSON 型** として返します。`->` / `->>` 演算子でメタデータへアクセスできるため、以下のような込み入った条件を書けます。

```sql
-- app_metadata 内の teams 配列に行の team_id が存在するか
USING ( team_id = ANY (SELECT jsonb_array_elements_text(auth.jwt() -> 'app_metadata' -> 'teams')) )
```

> [!WARNING]
> `raw_user_meta_data` はユーザ自身が書き換え可能です。認可判定に使うメタデータは **`raw_app_meta_data`** に保存しましょう。

### 2.3 Service ロール & `bypassrls`

Supabase には管理用 **Service Key** があり、RLS をバイパスできます。また Postgres 独自に

```sql
ALTER ROLE admin_role WITH BYPASSRLS;
```

とすることで同等の権限を付与可能ですが、**絶対にクライアントへ公開しない** ようにします。

---

## 3. パフォーマンスの考慮

RLS では **各行ごとに条件式を評価** するため、大量データで `LIMIT / OFFSET` 等を多用するとオーバーヘッドが顕在化します。

1. **インデックスを張る**: ポリシー内で参照する列 (例: `user_id`, `team_id`) に B-tree インデックスを作成。
   ```sql
   CREATE INDEX idx_projects_owner ON projects ( owner_id );
   ```
2. **クエリプランを確認**: `EXPLAIN ANALYZE` でコストを測定し、必要に応じてポリシーを簡略化。
3. **ビューや関数による集約**: 集合演算・複雑な `JOIN` はロジックを SQL 関数に隠蔽し、最小限の行を返す。

---

## 4. 開発・デバッグ Tips

| 操作 | 用途 |
| --- | --- |
| `ALTER TABLE <table> FORCE ROW LEVEL SECURITY;` | RLS を強制し、`superuser` でさえバイパスできなくする |
| `ALTER TABLE <table> DISABLE ROW LEVEL SECURITY;` | 一時的に RLS をオフ (本番では避ける) |
| `SET ROLE <role>;` | 別ロールに切り替えてポリシーを検証 |
| `pg_show_rls(<oid>)` | ポリシー評価結果のデバッグ (`EXPLAIN` 出力) |

---
## 5. ベストプラクティスまとめ

1. **アプリ設計時に RLS を前提** とし、テーブルスキーマに `tenant_id` や `owner_id` を先に組み込む
2. 認可データは必ず **`raw_app_meta_data`** に保存し、`auth.jwt()` で参照する
3. 開発環境では `SET pgaudit.log = 'all';` など監査ログを活用し、不正パスを早期検知
4. **最小権限の原則** : クライアントに渡すキーは匿名キーか特定ロール限定キーにする。Service Key を絶対に露出しない
5. 効率化のため RLS だけに頼らず **ビューや Stored Procedure** で読み書きをカプセル化する設計も検討

---

### 参考

- [Supabase Docs](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [PostgreSQL Official Docs](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
