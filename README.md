# eccube-specification

EC-CUBE開発用のドキュメントリポジトリです。

## 現在提供されているもの

- 機能一覧
- ER図/テーブル定義書
- 結合試験項目書
- 負荷試験項目書

## ER図の出力手順

schemaspyを利用して出力しています。

```
// ec-cubeの起動
$ git clone https://github.com/EC-CUBE/ec-cube.git
$ cd ec-cube
$ docker-compose -f docker-compose.yml -f docker-compose.pgsql.yml up -d
$ docker-compose -f docker-compose.yml -f docker-compose.pgsql.yml exec ec-cube composer run-script compile

// schemaspyで出力
$ docker run -v "$PWD/schema:/output" --net="host" schemaspy/schemaspy:6.1.0 -t pgsql -host 127.0.0.1:15432 -db eccubedb -u dbuser -p secret -connprops useSSL\\=false -all
```
