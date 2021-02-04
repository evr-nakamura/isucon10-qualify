# やったことリスト
## インメモリキャッシュ化
椅子検索・物件検索の結果をメモリにキャッシュさせて使い回すように

## コネクションプール設定
```
// コネクションプールの最大接続数を設定。
db.SetMaxIdleConns(10)
// 接続の最大数を設定。 nに0以下の値を設定で、接続数は無制限。
db.SetMaxOpenConns(13)
```

## echoのデバッグモードをオフに
```
e.Debug = false
```
debugモードだと整形されたJSONがログに出力されてしまうのでoffにする

## botからのリクエストを弾く
これは当日マニュアルに記載されている

## indexをはる
order byのasc,desc両方つかわれているところだと機能しないので、
descの値（popularity）を別カラムとしてマイナスを使って保持

## 空間インデックスを利用
なぞって検索で指定した位置の中に物件が存在するかを空間インデックスを利用した検索方法に変更

## DB分割
estateとchairでそれぞれサーバーを建てる
csvのとこバルクインサートにして上手く動いた

## mysqlチューニング
```
[mysqld]
# commit毎のログへの書き込み量
innodb_flush_log_at_trx_commit = 0
# デフォルトだとInnoDB にすべてのデータが2回書き込まれる
innodb_doublewrite = 0
# メモリにロードされるデータとインデックスのために使うメモリ量
innodb_buffer_pool_size = 1280M
# 各テーブルのデータおよびインデックスがシステムテーブルスペースではなく、個別の .ibd ファイルに格納される
innodb_file_per_table = ON
```

## estateテーブルをMyISAMに
空間インデックス上手く効かないっぽいから

## nginxチューニング
キャッシュ対応させようとしたけど上手くいっていないっぽい・・

# 導入したツール
## pprof
ソースにpprofを仕込む
```
go func() {
    e.Logger.Debug(http.ListenAndServe("localhost:6060", nil))
}()
```

## pt-query-digest
mysqlのスローログ集計出力ツール
```
// ツール取得
wget https://github.com/percona/percona-toolkit/archive/3.0.5-test.tar.gz

tar zxvf 3.0.5-test.tar.gz

sudo mv ./percona-toolkit-3.0.5-test/bin/pt-query-digest /usr/local/bin/pt-query-digest

sudo pt-query-digest /var/log/mysql/mysql-slow.log > query.log
```


# ベンチマーク起動までの手順
```
cd /home/isucon/isuumo/webapp/go
// ソース修正後
make
// goサーバー再起動
sudo systemctl restart isuumo.go.service
// ベンチ起動
cd /home/isucon/bench
./bench -target-url http://127.0.0.1
```


# vscode remote ssh接続
```
// windowsだからなのか（発生しない人もいる）こちらの現象が発生
https://www.ei.fukui-nct.ac.jp/2020/05/21/remote-ssh-could-not-to-establish/
```
鍵を作成してローカルと接続先においてあげて（キーの権限700にする必要あり）、
ssh configを用意して、
remote platformの設定をすればつながるようになる


# vagrant
## 複数台たてるように
【前提知識】
```
・vmネットワーク設定
https://qiita.com/centipede/items/64e8f7360d2086f4764f
・vagrantコマンド
https://qiita.com/oreo3@github/items/4054a4120ccc249676d9
```

```
box = "ubuntu/bionic64"
Vagrant.configure("2") do |config|
  config.vm.box = box
  config.vm.network "private_network", type: "dhcp"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
    vb.cpus = 1
  end

  config.vm.provision "shell", inline: <<-SHELL
    set -e
    export DEBIAN_FRONTEND=noninteractive
    apt-get update
    apt-get install -y --no-install-recommends ansible git

    GITDIR="${HOME}/isucon10-qualify"
    rm -rf ${GITDIR}
    git clone https://github.com/evr-nakamura/isucon10-qualify.git ${GITDIR}
    (
      cd ${GITDIR}/provisioning/ansible
      PYTHONUNBUFFERED=1 ANSIBLE_FORCE_COLOR=true ansible-playbook -i allinone, --connection=local allinone.yaml
    )
    rm -rf ${GITDIR}
  SHELL

  # テスト
  config.vm.define :test1 do |test|
    test.vm.network :private_network, ip: "192.168.1.11"
  end

  # ベンチマーカー
  config.vm.define :bench do |bench|
    bench.vm.network :private_network, ip: "192.168.1.12", virtualbox__intnet: "isucon10-network"
  end

  # アプリ１
  config.vm.define :app1 do |app|
    app.vm.network :private_network, ip: "192.168.1.13", virtualbox__intnet: "isucon10-network"
  end

  # アプリ２
  config.vm.define :app2 do |app|
    app.vm.network :private_network, ip: "192.168.1.14", virtualbox__intnet: "isucon10-network"
  end

  # アプリ3
  config.vm.define :app3 do |app|
    app.vm.network :private_network, ip: "192.168.1.15", virtualbox__intnet: "isucon10-network"
  end
end
```