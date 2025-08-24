# 設計書

## 概要

WEBリンククローラーは、uvを使用した依存関係管理と実行環境でPython 3.13ベースのコマンドラインツールとして設計されています。システムは指定されたドメイン内のWEBページを幅優先探索で巡回し、キューベースのアプローチでクロール対象URLを管理し、訪問済みURLのセットを維持して重複を防ぎ、堅牢な動作のための設定可能な制限とエラーハンドリングを実装します。

## アーキテクチャ

システムは関心の分離を明確にしたモジュラーアーキテクチャに従います：

```
WebLinkCrawler
├── URLManager (訪問済みURLとキューを管理)
├── HTTPClient (リトライロジック付きWEBリクエストを処理)
├── LinkExtractor (HTMLを解析してリンクを抽出)
├── DomainValidator (同一ドメイン制約を検証)
├── OutputFormatter (結果をフォーマットして表示)
└── ConfigManager (コマンドライン引数と設定を処理)
```

## コンポーネントとインターフェース

### 1. WebLinkCrawler (メインコントローラー)
**目的:** クロールプロセスを統制し、コンポーネント間を調整する。

**主要メソッド:**
- `crawl(start_url: str) -> CrawlResult`
- `process_url(url: str, depth: int) -> List[str]`
- `run() -> None`

**責任:**
- 設定でコンポーネントを初期化
- メインクロールループを管理
- URL管理と処理間の調整
- 適切なシャットダウンと結果編集

### 2. URLManager
**目的:** クロール対象URLのキューを管理し、訪問済みURLを追跡する。

**主要メソッド:**
- `add_url(url: str, depth: int) -> bool`
- `get_next_url() -> Tuple[str, int]`
- `is_visited(url: str) -> bool`
- `mark_visited(url: str) -> None`
- `has_pending_urls() -> bool`

**データ構造:**
- `visited_urls: Set[str]` - 処理済みの正規化されたURL
- `url_queue: Queue[Tuple[str, int]]` - 深度付きの処理対象URL
- `url_depth_map: Dict[str, int]` - URLと発見深度のマッピング

### 3. HTTPClient
**目的:** 適切なエラーハンドリングとリトライロジック付きHTTPリクエストを処理する。

**主要メソッド:**
- `fetch_page(url: str) -> Optional[str]`
- `is_valid_response(response: requests.Response) -> bool`

**機能:**
- 設定可能なタイムアウト設定
- 指数バックオフリトライメカニズム
- ユーザーエージェントヘッダー設定
- 設定可能な遅延付きレート制限
- HTTPステータスコードの適切な処理

### 4. LinkExtractor
**目的:** HTMLコンテンツを解析してすべてのhrefリンクを抽出する。

**主要メソッド:**
- `extract_links(html_content: str, base_url: str) -> List[str]`
- `normalize_url(url: str, base_url: str) -> str`
- `is_valid_link(url: str) -> bool`

**機能:**
- BeautifulSoupベースのHTML解析
- 相対URL解決
- URL正規化（フラグメント除去、パス正規化）
- 非HTTP(S)スキーム（mailto:、javascript:等）のフィルタリング

### 5. DomainValidator
**目的:** URLが開始URLと同じドメインに属するかを検証する。

**主要メソッド:**
- `is_same_domain(url: str, base_domain: str) -> bool`
- `extract_domain(url: str) -> str`
- `should_crawl(url: str) -> bool`

**機能:**
- サブドメイン処理設定
- プロトコル正規化
- ポート番号処理

### 6. OutputFormatter
**目的:** クロール結果を様々な形式でフォーマットして表示する。

**主要メソッド:**
- `format_results(results: CrawlResult, format_type: str) -> str`
- `print_progress(current_url: str, depth: int, total_found: int) -> None`
- `print_summary(results: CrawlResult) -> None`

**サポート形式:**
- テキスト（階層ツリー構造）
- JSON（構造化データ）
- CSV（表形式）

### 7. ConfigManager
**目的:** コマンドライン引数と設定を処理する。

**設定オプション:**
- `max_depth: int` (デフォルト: 3)
- `delay: float` (デフォルト: 1.0秒)
- `timeout: int` (デフォルト: 10秒)
- `output_format: str` (デフォルト: "text")
- `include_external: bool` (デフォルト: False)
- `max_urls: int` (デフォルト: 1000)
- `verbose: bool` (デフォルト: False)

## データモデル

### CrawlResult
```python
@dataclass
class CrawlResult:
    start_url: str
    internal_urls: List[UrlInfo]
    external_urls: List[UrlInfo]
    failed_urls: List[FailedUrl]
    total_processed: int
    total_found: int
    crawl_duration: float
```

### UrlInfo
```python
@dataclass
class UrlInfo:
    url: str
    depth: int
    parent_url: Optional[str]
    status_code: Optional[int]
    discovered_at: datetime
```

### FailedUrl
```python
@dataclass
class FailedUrl:
    url: str
    error_message: str
    attempted_at: datetime
    parent_url: Optional[str]
```

### CrawlerConfig
```python
@dataclass
class CrawlerConfig:
    max_depth: int = 3
    delay: float = 1.0
    timeout: int = 10
    output_format: str = "text"
    include_external: bool = False
    max_urls: int = 1000
    verbose: bool = False
    user_agent: str = "WebLinkCrawler/1.0"
```

## エラーハンドリング

### HTTPエラー
- **接続エラー:** ログに記録して次のURLで継続
- **タイムアウトエラー:** 指数バックオフで再試行（最大3回）
- **4xxエラー:** 失敗URLとしてログに記録、クロール継続
- **5xxエラー:** バックオフで再試行後、失敗としてログ記録

### 解析エラー
- **不正なHTML:** BeautifulSoupの寛容な解析を使用
- **無効なURL:** 警告をログに記録してスキップ
- **エンコーディング問題:** 複数のエンコーディング検出方法を試行

### リソース制限
- **メモリ制限:** URLキューサイズ制限を実装
- **時間制限:** オプションの最大クロール時間
- **URL制限:** 処理するURLの最大数を設定可能

## テスト戦略

### ユニットテスト
- **URLManager:** キュー操作、重複検出、深度追跡をテスト
- **LinkExtractor:** HTML解析、URL正規化、相対URL解決をテスト
- **DomainValidator:** ドメインマッチングロジック、サブドメイン処理をテスト
- **HTTPClient:** リトライロジック、タイムアウト処理、レスポンス検証をテスト
- **OutputFormatter:** 異なる出力形式、結果フォーマットをテスト

### 統合テスト
- **エンドツーエンドクロール:** モックHTTPレスポンスで完全なクロールワークフローをテスト
- **設定処理:** コマンドライン引数解析と検証をテスト
- **エラーシナリオ:** 様々なエラー条件での動作をテスト

### テストデータ
- **モックHTMLページ:** 様々なリンク構造のテストHTMLを作成
- **ドメインテストケース:** 内部/外部リンク分類をテスト
- **エラーレスポンスモック:** 様々なHTTPエラー条件をシミュレート

## パフォーマンス考慮事項

### メモリ管理
- メモリフットプリントを削減するため可能な限りジェネレーターを使用
- メモリ枯渇を防ぐためURLキューサイズ制限を実装
- 長時間実行クロールでは処理済みデータを定期的にクリア

### ネットワーク効率
- HTTPリクエスト用の接続プーリングを実装
- 接続を再利用するためセッションオブジェクトを使用
- robots.txtを尊重（オプション機能）
- 適切なレート制限を実装

### スケーラビリティ
- 将来のasync/await実装のための設計
- モジュラーアーキテクチャにより簡単なコンポーネント置換が可能
- 異なる用途のための設定駆動動作

## 開発環境

### Pythonバージョンとパッケージ管理
- **Pythonバージョン:** 3.13
- **パッケージマネージャー:** 高速な依存関係解決と仮想環境管理のためのuv
- **プロジェクト構造:** pyproject.toml設定を持つ標準Pythonパッケージ
- **依存関係:** 再現可能なビルドのためのロックファイル付きuvで管理

### uv設定
- プロジェクト初期化に`uv init`を使用
- `pyproject.toml`で依存関係を管理
- uvによる仮想環境の自動管理
- 依存関係の高速インストールと解決