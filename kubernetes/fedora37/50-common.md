# 共通設定作業

## hostname, IP, Gateway, DNS 設定

- Fedora の hostname, IP アドレス, Default Gateway, DNS を設定する。
  ```bash
  # コマンド例
  hostnamectl set-hostname k8s-cp01
  ip a
  nmcli connection modify ens192 ipv4.addresses 192.168.14.11/24
  nmcli connection modify ens192 ipv4.gateway 192.168.14.1
  nmcli connection modify ens192 ipv4.dns 192.168.13.2
  nmcli connection modify ens192 ipv4.method manual
  nmcli connection down ens192
  nmcli connection up ens192
  ip a
  ```

## ディスク拡張

- サーバに搭載しているディスク容量の全てが `fedora-root` に割り当てられていない場合は以下を実施して割当てサイズを拡張する。

  ```bash
  df -h | grep fedora-root
    # -> <出力例>
    #    /dev/mapper/fedora-root    15G  1.6G   14G   11% /
    # 15 GB しか割り当てられていないことが確認できる
  
  lvextend -An --extents +100%FREE /dev/mapper/fedora-root
    # -> <出力例>
    #    Size of logical volume fedora/root changed from 15.00 GiB (3840 extents) to <79.00 GiB (20223 extents).
    #    WARNING: This metadata update is NOT backed up.
    #    Logical volume fedora/root successfully resized.
  
  xfs_growfs /dev/mapper/fedora-root
    # -> data blocks changed from 3932160 to 20708352
  
  df -h | grep fedora-root
    # -> <出力例>
    #    /dev/mapper/fedora-root    79G  2.1G   77G    3% /
    # 79 GB まで拡張された
  ```

## dnfリポジトリキャッシュ更新無効化

- dnf の自動アップデート確認のタイマーを無効化する。

  ```bash
  systemctl stop dnf-makecache.timer
  systemctl disable dnf-makecache.timer
  systemctl status dnf-makecache.timer --no-pager
  ```

## 変数定義

- 構築作業を効率化するため、以下値をbashの環境変数に設定する。

  | 変数名                      | 設定値                                        | 設定例                    | 
  | :---                        | :---                                          | :---                      | 
  | k8s_management_hn           | 管理クライアントの hostname                   | k8s-management            | 
  | k8s_management_ip           | 管理クライアントの IP アドレス                | 192.168.14.30             | 
  | harbor_hn                   | Harbor の hostname                            | harbor2                   | 
  | harbor_ip                   | Harbor の IP アドレス                         | 192.168.14.40             | 
  | harbor_fqdn                 | Harbor の FQDN                                | harbor2.home.ndeguchi.com | 
  | k8s_vip_hn                  | Kubernetes クラスタ API サーバの hostname     | vip-k8s-master            | 
  | k8s_vip_ip                  | Kubernetes クラスタ API サーバの IP アドレス  | 192.168.14.10             | 
  | k8s_cp\<XX\>_hn             | Kubernetes Control Plnae XX号機の hoatname    | k8s-cp01                  | 
  | k8s_cp\<XX\>_ip             | Kubernetes Control Plnae XX号機の IP アドレス | 192.168.14.11             | 
  | k8s_worker\<XX\>_hn         | Kubernetes Worker Node XX号機の hoatname      | k8s-worker01              | 
  | k8s_worker\<XX\>_ip         | Kubernetes Worker Node XX号機の IP アドレス   | 192.168.14.21             | 
  | proxy_ip_port               | Proxy Server の IP アドレス:ポート番号        | 192.168.13.2:8080         | 
  | management_nw               | 管理ネットワークのネットワークアドレス        | 192.168.14.0/24           |

  ```bash
  # 設定例
  cat <<EOF >> ~/.bashrc

  # K8s Management
  k8s_management_hn=k8s-management
  k8s_management_ip=192.168.14.30
  
  # Harbor
  harbor_hn=harbor2
  harbor_ip=192.168.14.40
  harbor_fqdn=harbor2.home.ndeguchi.com
  
  # Kubernetes API Server VIP
  k8s_vip_hn=vip-k8s-master
  k8s_vip_ip=192.168.14.10
  
  # Kubernetes ControlPlane #01-03
  k8s_cp01_hn=k8s-cp01
  k8s_cp01_ip=192.168.14.11
  k8s_cp02_hn=k8s-cp02
  k8s_cp02_ip=192.168.14.12
  k8s_cp03_hn=k8s-cp03
  k8s_cp03_ip=192.168.14.13
  
  # Kubernetes Worker #01-02
  k8s_worker01_hn=k8s-worker01
  k8s_worker01_ip=192.168.14.21
  k8s_worker02_hn=k8s-worker02
  k8s_worker02_ip=192.168.14.22
  
  # Proxy Server
  proxy_ip_port=192.168.13.2:8080
  
  # Management Network
  management_nw=192.168.14.0/24
  EOF
  ```

- 変数読み込み
  ```bash
  source ~/.bashrc
  ```

## /etc/hosts 追記

- /etc/hosts に Harbor と Kubernetes API Server VIP を追記する。
  ```bash
  cat <<EOF >> /etc/hosts
  ${harbor_ip} ${harbor_fqdn}
  ${k8s_vip_ip} ${k8s_vip_hn}
  EOF
  
  cat /etc/hosts
  ```

- 名前解決に成功することを確認する
  ```bash
  nslookup ${harbor_fqdn}
  ```

  - 確認観点：名前解決に成功すること

    ```text
    Server:         127.0.0.53
    Address:        127.0.0.53#53
    
    Name:   harbor2.home.ndeguchi.com
    Address: 192.168.14.40
    ```

  ```bash
  nslookup ${k8s_vip_hn}
  ```

  - 確認観点：名前解決に成功すること

    ```text
    Server:         127.0.0.53
    Address:        127.0.0.53#53
    
    Name:   vip-k8s-master
    Address: 192.168.14.10
    ```

## Fedora の Proxy 設定

- Proxy サーバを設定する
  ```bash
  cat <<EOF >> /etc/environment
  export http_proxy=http://${proxy_ip_port}/
  export HTTP_PROXY=http://${proxy_ip_port}/
  
  export https_proxy=http://${proxy_ip_port}/
  export HTTPS_PROXY=http://${proxy_ip_port}/
  
  export no_proxy=localhost,127.0.0.1,${k8s_vip_ip},${k8s_vip_hn},${k8s_cp01_ip},${k8s_cp02_ip},${k8s_cp03_ip},${k8s_worker01_ip},${k8s_worker02_ip},${harbor_ip},${harbor_fqdn},${management_nw},10.96.0.0/12,10.20.0.0/16,${k8s_vip_hs},*.svc
  export NO_PROXY=localhost,127.0.0.1,${k8s_vip_ip},${k8s_vip_hn},${k8s_cp01_ip},${k8s_cp02_ip},${k8s_cp03_ip},${k8s_worker01_ip},${k8s_worker02_ip},${harbor_ip},${harbor_fqdn},${management_nw},10.96.0.0/12,10.20.0.0/16,${k8s_vip_hs},*.svc
  EOF
  
  cat /etc/environment
  source /etc/environment
  ```

## dnf(yum) のリポジトリ指定

- Proxy Server の宛先許可リストを固定するため、dnf のリポジトリ参照先を mirror リストから `riken.jp` に変更する。Proxy Server の宛先許可リストを固定する必要が無いのであれば実施不要。

  ```bash
  cd /etc/
  
  # backup
  ls -ld yum.repos.d*
    # -> yum.repos.d/ が存在すること
  
  cp -pr yum.repos.d yum.repos.d.bak
  ls -ld yum.repos.d*
    # -> yum.repos.d/ と yum.repos.d.bak/ が存在すること
  
  # 変更
  cd yum.repos.d
  sed -i -e "s/^metalink=/#metalink=/g" ./*
  sed -i -e "s,^#baseurl=http://download.example/pub/fedora/linux,baseurl=https://ftp.riken.jp/Linux/fedora,g" ./*
  sed -i -e "s/^enabled=1/enabled=0/g" fedora-cisco-openh264.repo
  
  # 差分出力（TerminalLogに記録するだけ。中身を読む必要はない）
  diff -u ../yum.repos.d.bak/ .
  
  # 動作確認
  dnf check-update
  ```

  - 確認観点：実行結果の冒頭に以下と同様の内容が出力され、リポジトリから情報を取得できていることを確認する。

    ```text
    Fedora 37 - x86_64                        32 MB/s |  82 MB     00:02
    Fedora Modular 37 - x86_64               7.1 MB/s | 3.8 MB     00:00
    Fedora 37 - x86_64 - Updates              18 MB/s |  40 MB     00:02
    Fedora Modular 37 - x86_64 - Updates     1.7 MB/s | 2.9 MB     00:01
    ```

## package update

- パッケージを更新

  ```
  dnf update -y
  ```

## (OPTIONAL) tmux,vim,bash インストール/設定

- tmux のインストール及び tmux, vim, bash の設定ファイルを作成する。 \
  構築作業には影響しないのでOptional.

  ```bash
  # tmux インストール
  dnf install -y tmux
  
  # .tmux.conf
  cat <<EOF > ~/.tmux.conf
  set -g prefix C-q
  unbind C-b
  bind h select-pane -L
  bind j select-pane -D
  bind k select-pane -U
  bind l select-pane -R
  setw -g mode-keys vi
  bind -T copy-mode-vi v send -X begin-selection
  bind | split-window -h
  bind - split-window -v
  bind -r H resize-pane -L 1
  bind -r J resize-pane -D 1
  bind -r K resize-pane -U 1
  bind -r L resize-pane -R 1
  bind B setw synchronize-panes on
  bind b setw synchronize-panes off
  EOF
  
  # .vimrc
  cat <<EOF > ~/.vimrc
  set nocompatible
  set number
  set tabstop=2
  set showmatch
  set incsearch
  set hlsearch
  set nowrapscan
  set ignorecase
  set fileencodings=utf-8,utf-16le,cp932,iso-2022-jp,euc-jp,default,latin
  set foldmethod=marker
  set nf=""
  nnoremap <ESC><ESC> :nohlsearch<CR>
  set laststatus=2
  set statusline=%t%m%r%h%w\%=[POS=%p%%/%LLINES]\[TYPE=%Y][FORMAT=%{&ff}]\%{'[ENC='.(&fenc!=''?&fenc:&enc).']'}
  syntax enable
  set directory=/tmp
  set backupdir=/tmp
  set undodir=/tmp
  set paste
  EOF
  
  # .bashrc 追記
  cat <<EOF >> ~/.bashrc
  alias k=kubectl
  set -o vi
  EOF
  ```


## Firewall 停止

- Firewall を停止する

  ```bash
  systemctl stop firewalld
  systemctl disable firewalld
  systemctl status firewalld --no-pager
  ```

  - 確認観点：`Active: inactive (dead)` が出力されること

## SELinux 無効化

- SELinux を無効化する
  ```bash
  cat /etc/selinux/config
  sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
  cat /etc/selinux/config
  ```

## NTP 設定

- chrony, ntpstat インストール

  ```bash
  # インストール
  dnf install -y chrony ntpstat
  
  # 設定変更
  cp -p /etc/chrony.conf /etc/chrony.conf.org
  ll /etc/chrony.conf*
  vim /etc/chrony.conf
    # 以下 diff 結果例を参考に NTP 同期先を指定する。
    # 構築している Fedora から参照可能な NTP サーバを指定すること。
  
  # 差分確認
  diff -u /etc/chrony.conf.org /etc/chrony.conf
  ```

  - diff 結果例

    ```diff
    -pool 2.fedora.pool.ntp.org iburst
    +pool 192.168.14.1 iburst
    ```

  ```bash
  # chronyd 起動
  systemctl enable --now chronyd
  
  # 確認
  chronyc sources
  ntpstat
  ```

  - 確認観点：NTP サーバと時刻を同期できていること

    ```
    synchronised to NTP server (192.168.14.1) at stratum 3
       time correct to within 17 ms
       polling server every 64 s
    ```

## 再起動・確認

- 上記設定を反映・確認するためサーバを再起動する

  ```bash
  shutdown -r now
  ```

- Proxy 設定確認

  ```bash
  env | grep -i proxy | sort
  ```

  - 確認観点：/etc/environment に設定した Proxy の設定が反映されていること

- SELinux 確認

  ```bash
  getenforce
  ```

  - 確認観点：`Permissive` が出力されること

- NTP 同期確認
  ```bash
  ntpstat
  ```
  - 確認観点：NTP サーバと同期できていること

## Docker Engine インストール・設定

- インストール

  ```bash
  # 古いバージョンのDocker パッケージを削除
  dnf remove -y docker docker-client docker-client-latest docker-common \
                docker-latest docker-latest-logrotate docker-logrotate \
                docker-selinux docker-engine-selinux docker-engine
  
  # リポジトリを追加
  dnf -y install dnf-plugins-core
  dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
  
  # Docker Engine をインストール
  dnf install -y docker-ce docker-ce-cli containerd.io \
                 docker-buildx-plugin docker-compose-plugin
  
  # Docker Engine を起動
  systemctl start docker
  systemctl enable docker
  systemctl status docker --no-pager
  ```

  - 確認観点： `active (running)` が出力されること

    ```text
    ● docker.service - Docker Application Container Engine
         Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: disabled)
         Active: active (running) since Mon 2024-01-01 21:40:56 JST; 15h ago
    TriggeredBy: ● docker.socket
           Docs: https://docs.docker.com
       Main PID: 1036 (dockerd)
          Tasks: 11
         Memory: 571.6M
            CPU: 7min 6.442s
         CGroup: /system.slice/docker.service
                 └─1036 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
    ```


- Proxy 設定前確認 \
  docker の proxy を設定する前に dockerhub からのコンテナイメージ取得に失敗することを確認する。

  ```bash
  docker run --rm hello-world
  ```

  - 確認観点:エラーメッセージが出力されること

- Proxy 設定 \
  docker の proxy を設定する。

  ```bash
  mkdir -p /etc/systemd/system/docker.service.d

  cat <<EOF > /etc/systemd/system/docker.service.d/http-proxy.conf
  [Service]
  Environment="HTTP_PROXY=${HTTP_PROXY}"
  Environment="HTTPS_PROXY=${HTTPS_PROXY}"
  Environment="NO_PROXY=${NO_PROXY}"
  EOF
  
  cat /etc/systemd/system/docker.service.d/http-proxy.conf
  systemctl daemon-reload
  systemctl restart docker
  systemctl status docker --no-pager
  systemctl show --property=Environment docker --no-pager
  ```

  - 確認観点：設定した Proxy の設定が出力されること
    ```text
    <出力例>
    Environment=HTTP_PROXY=http://192.168.13.2:8080 HTTPS_PROXY=http://192.168.13.2:8080 "NO_PROXY=localhost,127.0.0.1,192.168.14.10,192.168.14.11,192.168.14.12,192.168.14.13,192.168.14.21,192.168.14.22,192.168.14.0/24,10.96.0.0/12,10.20.0.0/16,vip-k8s-master,*.svc"
    ```

- 動作確認 \
  Proxy 経由で Docker Hub からコンテナイメージを取得できる環境では以下が実行出来ることを確認する。

  ```bash
  docker run --rm hello-world
  ```

  - 確認観点：`Hello from Docker!` のメッセージが出力されること

    ```text
    Unable to find image 'hello-world:latest' locally
    latest: Pulling from library/hello-world
    719385e32844: Pull complete
    Digest: sha256:88ec0acaa3ec199d3b7eaf73588f4518c25f9d34f58ce9a0df68429c5af48e8d
    Status: Downloaded newer image for hello-world:latest
    
    Hello from Docker!
    This message shows that your installation appears to be working correctly.
    
    To generate this message, Docker took the following steps:
     1. The Docker client contacted the Docker daemon.
     2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
        (amd64)
     3. The Docker daemon created a new container from that image which runs the
        executable that produces the output you are currently reading.
     4. The Docker daemon streamed that output to the Docker client, which sent it
        to your terminal.
    
    To try something more ambitious, you can run an Ubuntu container with:
     $ docker run -it ubuntu bash
    
    Share images, automate workflows, and more with a free Docker ID:
     https://hub.docker.com/
    
    For more examples and ideas, visit:
     https://docs.docker.com/get-started/
    ```

