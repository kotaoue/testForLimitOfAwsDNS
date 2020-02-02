# 疑問
dnsmasq等がローカルキャッシュを持っていない状態で
複数プロセスからDNS問い合わせがあった場合はどのような挙動になるのか

# 予想
一般的なキャッシュの挙動と同様に、
複数プロセスから同時に問い合わせがあった場合
DNSサーバへの問い合わせが複数回行われると予想。

つまり、Apacheの場合は同時接続可能数分だけ、
nginxの場合は最大プロセス数だけDNSサーバへの
問い合わせが行われる可能性があると予想。

そのため、アクセスするドメインが1024以下であっても、
AWS DNSの名前解決制限に引っかかることがあると予想。

# 方法
- 環境
  - EC2インスタンス (AWS linux t2.micro) 
- テスト方法
  1. ローカルキャッシュを行わずEC2インスタンスから名前解決を行う。
  1. ローカルキャッシュを利用してEC2インスタンスから名前解決を行う。
  1. ローカルキャッシュの複数プロセスからRDSの名前解決を行う。
- 確認方法
  - 1と2でローカルキャッシュが利用されていることを確認。
  - 2と3で複数プロセスから同時アクセスがあった場合の挙動を確認。
- テスト詳細
  - https://aws.amazon.com/jp/premiumsupport/knowledge-center/dns-resolution-failures-ec2-linux/ の手順でインストール。
  /etc/dnsmasq.conf は以下のようにキャッシュ時間を1秒、キャッシュ数を1に変更。
  ```
  # Server Configuration
  listen-address=127.0.0.1
  port=53
  bind-interfaces
  user=dnsmasq
  group=dnsmasq
  pid-file=/var/run/dnsmasq.pid
  
  # Name resolution options
  resolv-file=/etc/resolv.dnsmasq
  cache-size=1
  min-cache-ttl=1
  max-cache-ttl=1
  neg-ttl=60
  domain-needed
  bogus-priv
  ```
  - 確認方法 1については、dnsmasqのインストールで確認。
  - 確認方法 2については、以下のテストスクリプトを実行しつつパケットキャプチャを実行して確認。
  ```
  #!/bin/bash
  while :
  do
  dig +short google.com
  done
  ```
  ```
  # 1枚目のターミナルで
  sh test.sh

  # 2枚目のターミナルで
  tcpdump -s 0 -i any port 53 -w single.pcap
  ```
  - 確認方法 3については、テストスクリプトを複数同時実行しつつパケットキャプチャを実行して確認。
  ```
  # 1枚目のターミナルで
  sh test.sh

  # 2枚目のターミナルで
  sh test.sh

  # 3枚目のターミナルで
  tcpdump -s 0 -i any port 53 -w parallel.pcap
  ```

# 結果
複数プロセスで実行した際のログには連続したDNSサーバへの問い合わせが記録されていた。

つまり、複数プロセスの同時アクセスの場合、同一のDNSに対して複数回の外部問い合わせが実行される可能性がある。

そのため、ローカルキャッシュのTTLや同時アクセス数によっては、アクセスしているドメイン数にかかわらず、ENIのDNSクエリ上限に引っかかる場合がある。

# 注意点
AWS DNSにおける制限は、1024送信パケット/秒となる。

>各 Amazon EC2 インスタンスは Amazon が提供する DNS サーバーへ送信できるパケット数をネットワークインターフェイスあたり最大 1024 パケット/秒に制限しています。
https://docs.aws.amazon.com/ja_jp/vpc/latest/userguide/vpc-dns.html

送信パケットのみの制限で、最大1024クエリ実行可能。
受信パケットには影響ないのでDNS関連のDDos等は無視できる。
TCPフォールバックが発生した場合はどのようになるかは
理解できていないために今回は考慮しない。

ENI単位の制限となっているために、ECS等を使用する場合は要注意。

# 参考資料
- EC2における単位時間あたりの名前解決制限の対応
https://devblog.thebase.in/entry/2019/12/02/123000

- EC2 DNS名前解決制限をECSでも回避する方法
http://iga-ninja.hatenablog.com/entry/2019/04/15/014914