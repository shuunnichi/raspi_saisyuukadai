
# 最終課題報告書：Docker Composeを用いたWordPress環境および監視システムの構築手順書

## 1. 目的と完成条件

本手順書では、Raspberry Pi 400（Raspberry Pi OS）を対象に、Docker Composeを用いてWebサーバ（WordPress）、データベース（MySQL）、およびリアルタイム監視ツール（Uptime-Kuma）が連携する環境を構築する 。

完成条件:

1. OSの初期設定およびセキュリティ設定（UFW）が完了していること。
    
2. DockerおよびDocker Composeが正常に動作すること。
    
3. ブラウザで `http://localhost:8080` にアクセスし、WordPressの画面が表示されること。
    
4. **発展要素**：ブラウザで `http://localhost:3001` にアクセスし、監視ダッシュボードでWebサイトの死活監視ができること 。
    
5. OS再起動後も、すべてのサービスが自動的に立ち上がること（永続化） 。
    

6. 前提条件と環境

- **ハードウェア**: Raspberry Pi 400
    
- **OS**: Raspberry Pi OS (64-bit)
    
- **操作環境**: Raspberry Pi 400上のターミナルで直接操作（SSH不使用）
    
- **選択した発展要素**: 例 B（2コンテナ構成）および **例 E（軽量監視の導入：Uptime-Kuma）** の複合構成

## 2. 全体構成図とポート設計

本システムは、Raspberry Pi 400（ホストOS）上のDocker環境で、3つの独立したコンテナを仮想ネットワークで接続して運用します。

### ポート一覧

|**コンポーネント**|**コンテナ内部ポート**|**ホスト公開ポート**|**用途**|
|---|---|---|---|
|**WordPress**|80|**8080**|Webサイト閲覧|
|**MySQL**|3306|なし（内部のみ）|データベース接続|
|**Uptime-Kuma**|3001|**3001**|サービス死活監視|

## 3. 構築手順

### 3.1.

OSのインストールと初期設定（GUI）

1. 「Raspberry Pi Imager」を使用し、SDカードに OS を書き込む。
    
2. 初回起動時のウィザードに従い、日本設定、ユーザー作成、Wi-Fi接続を行う。
    

### 3.2.

システム更新

ターミナルを開き、以下のコマンドでシステムを最新にします。

Bash

```
sudo apt update && sudo apt upgrade -y
```

### 3.3.

セキュリティ設定（UFW）

外部からのアクセスを制限します。Web用（8080）と監視用（3001）のポートを許可します。

Bash

```
sudo apt install ufw -y
sudo ufw allow 8080/tcp
sudo ufw allow 3001/tcp
sudo ufw enable
```

- **確認**: `sudo ufw status` で 8080 と 3001 が ALLOW になっていることを確認 。
    

### 3.4.

Docker環境の構築

1. **インストール**: `curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh`
    
2. **権限設定**: `sudo usermod -aG docker $USER`
    
3. **反映**: 設定を反映させるため、一度 `sudo reboot` で再起動する 。
    

### 3.5.

Docker Composeによる環境構築（発展要素）

作業ディレクトリを作成し、3つのサービス（DB、WP、監視）を定義する設定ファイルを作成します。

Bash

```
mkdir ~/wp-project && cd ~/wp-project
nano compose.yaml
```

**compose.yaml の内容**（インデントに注意）:

YAML

```
services:
  db:
    image: mysql:8.0
    container_name: wp-db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppassword
    volumes:
      - db_data:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    container_name: wp-server
    restart: always
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppassword
      WORDPRESS_DB_NAME: wordpress
    depends_on:
      - db
    volumes:
      - wp_data:/var/www/html

  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: always
    ports:
      - "3001:3001"
    volumes:
      - kuma_data:/app/data

volumes:
  db_data:
  wp_data:
  kuma_data:
```


### 3.6.

サービスの起動

Bash

```
docker compose up -d
```

- **確認**: `docker compose ps` を実行し、3つのコンテナの STATUS が `Up` であれば成功 。
    

## 4. 動作確認と検証

### 4.1. 各サービスへのアクセス確認

1. **WordPress**: `http://localhost:8080` にアクセスし、言語選択画面が出るか確認。
    
2. **Uptime-Kuma**: `http://localhost:3001` にアクセスし、監視設定画面が出るか確認。
    

### 4.2. 監視の設定（検証手順）

1. Uptime-Kumaの画面で管理者アカウントを作成する。
    
2. 「監視の追加」ボタンを押し、URLに `http://wordpress:80`（コンテナ間通信）を入力して保存する。
    
3. ダッシュボード上でステータスが緑色の「Up」になれば、正常に監視できている 。
    

### 4.3.

再起動テスト（永続化の確認）

`sudo reboot` を実行し、起動後に何も操作せずブラウザで各サイトが表示されることを確認する。これは `restart: always` 設定が機能している証明となる 。


### 4.4. 実行結果（スクリーンショット）

手順書の再現性を証明するため、以下の5つのスクリーンショットをここに添付します。


1. **コンテナ起動確認**: ![ターミナルで `docker compose ps` を実行し、3つのコンテナが `Up` になっている画面。](https://github.com/shuunnichi/raspi_saisyuukadai/blob/main/pic/20260202_14h22m18s_grim.png)
    
2. **ファイアウォール設定**: ![ターミナルで `sudo ufw status` を実行し、8080と3001が許可されている画面 。](https://github.com/shuunnichi/raspi_saisyuukadai/blob/main/pic/20260202_14h22m38s_grim.png)
    
3. **WordPress画面**: ![ブラウザで `http://localhost:8080` を開いている画面 。](https://github.com/shuunnichi/raspi_saisyuukadai/blob/main/pic/20260202_14h21m45s_grim.png)
    
4. **監視ダッシュボード**: ![ブラウザで `http://localhost:3001` を開き、WordPressのステータスが「UP（緑色）」になっている画面。](https://github.com/shuunnichi/raspi_saisyuukadai/blob/main/pic/20260202_14h21m00s_grim.png)
    
5. **永続化確認**: ![再起動直後にターミナルを開き、コマンド入力なしでコンテナが動いていることを示す `docker compose ps` の画面 。](https://github.com/shuunnichi/raspi_saisyuukadai/blob/main/pic/20260202_14h34m05s_grim.png)
    

## 5. トラブルシューティング

- **「サイトにアクセスできない」**: `docker compose ps` でコンテナが動いているか確認。止まっている場合は `docker compose logs` でエラー内容を確認する 。
    
- **「DB接続エラー」**: `compose.yaml` 内の `MYSQL_PASSWORD` と `WORDPRESS_DB_PASSWORD` が一致しているか再確認する。
    
- **「監視ツールが動かない」**: UFWの設定で 3001 番ポートが許可されているか確認する 。
    

## 6. まとめ

本構成では、Docker Composeを利用することでWeb環境の構築をコード化（IaC）し、さらにUptime-Kumaを導入することで、システムの安定稼働を視覚的に管理できる「運用」を意識した環境を実現した 。

---

## 7. セキュリティ配慮

本環境では、構築の簡便さと安全性を両立するため以下の対策を講じています。

- **ホワイトリスト方式の制限**: UFWを用い、Webサイト閲覧（8080）と監視（3001）に必要な最小限のポートのみを外部に公開し、その他の不要な通信を遮断しています 。
    
- **パスワードの分離**: DBのルートパスワードと一般ユーザーパスワードを分け、最小権限の原則に従った運用を想定しています 。
    
- **機密情報の管理**: （発展的な取り組みとして）本来は `compose.yaml` に直接記述せず、`.env` ファイルに機密情報を切り出して管理することが推奨されます 。
    
- **データベースの非公開**: MySQLのポート（3306）はホスト側に公開せず、Dockerネットワーク内のコンテナ間通信のみに限定することで、外部からの直接攻撃を防いでいます 。
    
