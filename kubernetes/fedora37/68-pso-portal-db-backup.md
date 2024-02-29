# PSO Portal DB Backup/Restore

PSO Portal の DB を Backup / Restore する手順を以下に記載する。 \
作業は全て管理クライアントで実施すること。

## Backup

- PostgreSQL の Pod 名を確認する。
  ```bash
  kubectl get pod -n vmw-pso-portal | grep -e NAME -e postgres
  ```

  - 確認観点：PostgreSQL の Pod 名を確認する
    ```text
    <例>
    NAME                        READY   STATUS      RESTARTS   AGE
    postgres-69d9b696b6-lhzg4   1/1     Running     0          33d
    ```

- Pod にログインする
  ```bash
  kubectl exec -it <pod名> -n vmw-pso-portal -- bash
  ```

  - 確認観点：プロンプトが `root@<pod名>:/#` に変わること
    ```text
    <例>
    root@postgres-69d9b696b6-lhzg4:/#
    ```

- 作業ディレクトリを作成する
  ```bash
  mkdir -p /backup/$(date +%Y%m%d)/
  cd /backup/$(date +%Y%m%d)/
  pwd
  ```

  - 確認観点：カレントディレクトリが `/backup/YYYYMMDD` であること

- バックアップ取得 
  ```bash
  export PASSWORD=********
  psql -U postgres -l
  ```

  - 確認観点：テーブル一覧が出力されること
    ```text
    <出力例>
                                      List of databases
        Name     |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
    -------------+----------+----------+------------+------------+-----------------------
     history     | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
     inventory   | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
     notice      | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
     portal_auth | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
     postgres    | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
     template0   | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
                 |          |          |            |            | postgres=CTc/postgres
     template1   | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
                 |          |          |            |            | postgres=CTc/postgres
     vcenter_vm  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
    (8 rows)
    ```

  ```bash
  dbs=("history" "inventory" "notice" "portal_auth" "vcenter_vm")
  
  for db in ${dbs[@]}; do
    echo "${db}"
    pg_dump -U postgres -Fc -d ${db} > ${db}
  done
  
  ls -l
  ```

  - 確認観点：dumpファイルが存在すること
    ```text
    <例>
    -rw-r--r--. 1 root root  3539 Feb  8 14:09 history
    -rw-r--r--. 1 root root  7471 Feb  8 14:09 inventory
    -rw-r--r--. 1 root root  2844 Feb  8 14:09 notice
    -rw-r--r--. 1 root root 13121 Feb  8 14:09 portal_auth
    -rw-r--r--. 1 root root  6989 Feb  8 14:09 vcenter_vm
    ```

  ```bash
  cd ..
  ls -ld $(date +%Y%m%d)
  tar -zcvf $(date +%Y%m%d).tar.gz $(date +%Y%m%d)
  ls -l $(date +%Y%m%d).tar.gz
  exit
  ```

- pod から tar.gz ファイルを取得する
  ```bash
  cd /tmp
  kubectl -n vmw-pso-portal cp <pod名>:backup/$(date +%Y%m%d).tar.gz $(date +%Y%m%d).tar.gz 
  ll $(date +%Y%m%d).tar.gz 
    # -> ファイルが存在すること
  ```

- 上記で取得した tar.gz ファイルをサーバから取得し、データロストしない場所で保管して下さい。


## Restore

- PostgreSQL の Pod 名を確認する。
  ```bash
  kubectl get pod -n vmw-pso-portal | grep -e NAME -e postgres
  ```

  - 確認観点：PostgreSQL の Pod 名を確認する
    ```text
    <例>
    NAME                        READY   STATUS      RESTARTS   AGE
    postgres-69d9b696b6-lhzg4   1/1     Running     0          33d
    ```

- Backup ファイルを　PostgreSQL の Pod の /tmp に配置する。
  ```bash
  kubectl -n vmw-pso-portal cp <backupファイル> <pod名>:tmp/<backupファイル名>
  ```

- Pod にログインする
  ```bash
  kubectl exec -it <pod名> -n vmw-pso-portal -- bash
  ```

  - 確認観点：プロンプトが `root@<pod名>:/#` に変わること
    ```text
    <例>
    root@postgres-69d9b696b6-lhzg4:/#
    ```

- Restore
  ```bash
  cd /tmp/
  ls -l <backupファイル>
    # -> backupファイルが存在すること

  tar -zxvf <backupファイル>
  ls -l
    # -> 展開されたディレクトリが存在すること
  
  cd <展開されたディレクトリ>
  ls -l
    # -> dump ファイルが存在すること

  for db in $(ls); do
    echo "${db}"
    pg_restore --clean -U postgres -d ${db} ${db}
  done
  ```

