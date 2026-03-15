# Chain-of-Event 障害検出能力調査

日付: 2026-03-15

## 概要
Chain-of-Event が Rook Ceph OSD 障害（CrashLoopBackoff による PV マウント失敗）を検出できるか、および Prometheus・SigNoz への対応状況を調査した。

## 作業内容

### 1. リポジトリ全体の構造調査
- `deep_rule_evaluation.py`（エントリポイント）、`graph/` 配下の各モジュール、`conf/` を調査
- システムの目的・アーキテクチャ・データフローを把握

**判明した構成:**
- eBay 社内サービスプール間の因果連鎖を解析する RCA（根本原因分析）システム
- 入力: ローカル JSON インシデントファイル（`dataset/<catalog>/` 以下）
- アルゴリズム: BERT + GAT による深層学習 + 修正 PageRank

### 2. Rook Ceph OSD 障害の検出可否を評価
- 監視対象データソース・イベント種別・依存グラフの内容を確認
- 障害チェーン（OSD 不整合 → CrashLoopBackoff → PV 不可 → アプリ Pod FailedMount）と照合

**結論: 現状の実装では検出不可**

| 観点 | Chain-of-Event の現実装 | Rook Ceph 障害に必要なもの |
|------|------------------------|--------------------------|
| データソース | eBay サービスプールイベント | Kubernetes Events、Ceph OSD ヘルス |
| イベント種別 | `InternalErrorSpike`, `DBMarkdown` 等 | `CrashLoopBackoff`, `FailedMount` 等 |
| 依存グラフ | eBay アプリサービス間 | OSD Pod → PV → PVC → アプリ Pod |

対応には以下の追加実装が必要:
- Kubernetes Events を `poolEvents` として取り込むアダプター
- Ceph 固有の因果ルール定義
- Rook/Ceph トポロジーを `subGraph` として定義

### 3. Prometheus・SigNoz 対応状況を調査
- リポジトリ全体に対して `prometheus`, `signoz`, `grafana`, `opentelemetry`, `otel`, `metrics`, `http`, `api`, `webhook` 等のキーワードで検索

**結論: 未対応**

- 外部システムとの連携コードは一切存在しない
- 入力は `open(filePath)` によるローカル JSON ファイル読み込みのみ
- REST API・メッセージキュー・ストリーム連携なし

| 外部システム | 対応状況 |
|-------------|---------|
| Prometheus | 未対応 |
| SigNoz | 未対応 |
| Grafana | 未対応 |
| OpenTelemetry | 未対応 |
| Kafka / ストリーム | 未対応 |

## 変更ファイル一覧
なし（読み取り専用の調査作業）

## 結果
Chain-of-Event は eBay 社内サービス専用の RCA システムであり、Kubernetes/Ceph インフラ層の障害検出や外部監視ツール（Prometheus・SigNoz）との連携には現状対応していない。これらに対応するには、データ取り込みアダプターと Kubernetes/Ceph 固有の因果ルールの実装が必要。

## 備考
- リポジトリの Python ファイル: `deep_rule_evaluation.py`, `graph/deep_rule.py`, `graph/rca_algo.py`, `graph/groot_algo.py`, `graph/event_graph_search.py`, `util/cfg.py`, `util/log.py`, `conf/constants.py`
- 設定ファイル: `conf/groot.conf`（eBay サービスプール定義）
- 依存ライブラリ: PyTorch 2.0.1, PyTorch Geometric 2.3.1, NetworkX 2.8.8, Transformers (BERT)
