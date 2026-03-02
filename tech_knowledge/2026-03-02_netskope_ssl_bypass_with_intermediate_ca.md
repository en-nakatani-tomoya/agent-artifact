# netskope ssl bypass with intermediate ca

## 概要
Python 3.13のVERIFY_X509_STRICTフラグにより、Netskope環境下でHTTPSリクエストが失敗する問題を、中間証明書付きCAバンドルで解決した記録。コードレベルのワークアラウンドを排除し、環境レベルの正しい対処に置き換えた。

## 詳細
## 問題

Python 3.13で追加された `VERIFY_X509_STRICT` フラグにより、Netskope環境下のHTTPSリクエストが以下のエラーで失敗する。

```
ssl.SSLCertVerificationError: [SSL: CERTIFICATE_VERIFY_FAILED]
```

原因はNetskopeのルートCA証明書に `keyUsage` 拡張が欠如しており、RFC 5280準拠の厳格なX.509検証で拒否されること。

---

## 背景知識

### CA（Certificate Authority / 認証局）とは

「この証明書は本物だ」と保証する第三者機関。現実世界の公証役場のような役割。

- **証明書の発行**: サーバー運営者の身元を確認し、秘密鍵で署名した証明書を発行する
- **信頼の起点**: OSやブラウザには「信頼するCAのリスト」が事前に入っている

### 証明書チェーンの仕組み

TLS通信では、サーバーが「自分は本物だ」と証明するために証明書を提示する。この信頼は階層構造（PKI）で成り立つ。

```
ルートCA証明書        自分で自分を署名（自己署名）。OSやブラウザに最初から入っている。
  │
  │ 署名（「この中間CAを信頼してよい」）
  ▼
中間CA証明書          ルートCAから署名をもらっている。サーバー証明書を発行する実務担当。
  │
  │ 署名（「このサーバーは本物だ」）
  ▼
サーバー証明書        実際のWebサイトやAPIが提示する証明書。
```

### なぜ中間CAが存在するのか

ルートCAは信頼の最上位であり、秘密鍵が漏洩すると全システムが危険にさらされる。

- **ルートCAの秘密鍵はオフラインで厳重に保管**される（通常ネットワークに接続しないHSMに格納）
- 日常的な証明書発行は**中間CA**が行う
- 中間CAの秘密鍵が漏洩しても、その中間CAだけを失効させれば済む（ルートCAは無傷）

### Netskopeの通信フロー

```
通常の通信:
  クライアント  ──HTTPS──▶  API サーバー

Netskopeがいる場合:
  Python  ──HTTPS──▶  Netskope(中間者)  ──HTTPS──▶  API サーバー
                        │
                        └─ Netskopeが自前のCA証明書で
                           通信を再署名して返す
```

NetskopeはTLS通信を検査するため、通信を一度復号して再署名する。Pythonが受け取る証明書チェーンはNetskopeのCAで署名されたものになるため、NetskopeのCA証明書をシステムのトラストストアに追加する必要がある。

---

## 検証した方法と結果

### Round 1: CAバンドル系アプローチ（ルートCAのみ）

| アプローチ | 結果 |
|-----------|------|
| baseline（素のrequests） | FAILED |
| REQUESTS_CA_BUNDLE（certifi + ルートCA） | FAILED |
| SSL_CERT_FILE（certifi + ルートCA） | FAILED |
| verify=bundle（直接指定） | FAILED |
| disable_X509_STRICT（コード内でフラグ除去） | OK |

→ CAバンドル系は全滅。問題はCA信頼ではなく `VERIFY_X509_STRICT` の厳格検証。

### Round 2: モンキーパッチ系アプローチ

| アプローチ | 結果 |
|-----------|------|
| urllib3.create_urllib3_contextをパッチ | FAILED |
| ssl.create_default_contextをパッチ | FAILED |
| sitecustomize.pyでパッチ | FAILED |

→ requests/urllib3がimport時に関数参照をコピー済みのため、モジュール属性を差し替えても既存参照には影響しない。

### Round 3: 低レベルアプローチ（サブプロセスで検証）

| アプローチ | 結果 |
|-----------|------|
| baseline | FAILED |
| ssl.VERIFY_X509_STRICT = 0（sitecustomize.py） | OK |
| SSLContext.verify_flags descriptorパッチ | OK |
| 上記2つの組み合わせ | OK |

→ `sitecustomize.py` で `ssl.VERIFY_X509_STRICT = 0` にすればPython起動時にフラグが無効化され、全importより前に実行されるため有効。

### Round 4: 中間証明書ありCAバンドル

| アプローチ | 結果 |
|-----------|------|
| baseline | FAILED |
| REQUESTS_CA_BUNDLE（certifi + 中間証明書付きバンドル） | **OK** |
| SSL_CERT_FILE | FAILED |
| verify=bundle（直接指定） | **OK** |
| nullify_constant（参考） | OK |

→ **中間証明書ありなら `VERIFY_X509_STRICT` の無効化すら不要。**

---

## なぜルートCAだけだとダメだったか

旧 `nscacert.pem` にはルートCAのみが入っていた場合:

```
トラストストア: [certifi標準CAs, NetskopeルートCA]

検証:
  サーバー証明書 → 署名者は Netskope中間CA
  Netskope中間CA → ???（トラストストアにない、サーバーからも送られてこない）
  → チェーンが途切れる
  → OpenSSLがルートCAを直接使おうとする
  → VERIFY_X509_STRICT が keyUsage拡張の欠如を検出
  → 拒否
```

中間証明書付きバンドルの場合:

```
トラストストア: [certifi標準CAs, NetskopeルートCA, Netskope中間CA]

検証:
  サーバー証明書 → 署名者は Netskope中間CA ✓（トラストストアにある、keyUsageも正常）
  Netskope中間CA → 署名者は NetskopeルートCA ✓（トラストアンカー）
  → チェーン検証成功
```

中間CA証明書は実務で頻繁に使われるため適切な拡張（`keyUsage` 等）が付与されていた、というのが核心。

---

## 採用した解決策

Dockerfileが既に以下の仕組みを持っていた:

```dockerfile
ENV REQUESTS_CA_BUNDLE="/etc/ssl/certs/ca-certificates.crt"
COPY docker/nscacert.pem /usr/local/share/ca-certificates/nscacert.crt
RUN update-ca-certificates
```

`nscacert.pem` を中間証明書付きバンドルに差し替えるだけで解決。

**変更内容:**

| ファイル | 変更 |
|---------|------|
| `docker/nscacert.pem` | 中間証明書ありバンドルで上書き |
| `ranking_client_impl.py` | `_NetskopeSSLAdapter` を除去、素の `requests.Session` に |

### セキュリティ面の比較

| | 以前（_NetskopeSSLAdapter） | 今回 |
|---|---|---|
| VERIFY_X509_STRICT | **無効化** | 有効のまま |
| 証明書チェーン検証 | 厳格検証を迂回 | 正規の検証に合格 |
| 影響範囲 | ランキングAPI全リクエスト | なし（標準動作） |

以前は「検証を緩める」ワークアラウンドだったが、今回は「正しい証明書チェーンを提供して正規の検証を通す」という本来あるべき対処。セキュリティ面で新たに注意すべきことはない。

---
Generated: 2026-03-02
