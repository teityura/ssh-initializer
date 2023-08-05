# ssh-initializer

VMに初回接続するときなどに、

1. ~/.ssh/config 書いて、
2. ssh-copy-id | authorized_keysの追記 して、
3. sshで rootログインできるようにして、
4. 一般ユーザで パスなしsudoできるようにする
5. lvmスナップショットを作成します

という、諸々の作業を一発で完了させるプレイブックです

以下のコマンド例は、
プロジェクトディレクトリ: <project_dir>
プロジェクト名: <project_name>
ホスト名: jump
をセットアップする場合の説明になります

## 概要

> roles/ssh

hosts.csv に記載した内容をもとに、  
.ssh/ 以下に config, id_rsa, rsa.pub を生成します

> roles/inventory

インベントリ(hosts) を生成します

> roles/user

authorized_keys, パスなしsudo, rootログイン を許可します

> roles/lvm

lvmスナップショットを作成します

## 使い方

- clone する

```
mkdir -p <project_dir>
cd <project_dir>/
git clone https://github.com/teityura/ssh-initializer.git <project_name>
cd <project_name>/
```

- hosts.csv を作成する

```
vim hosts.csv
```

- プレイブックを実行する

```
# 一括実行
# ansible-playbook setup.yml

# 個別実行
ansible-playbook setup.yml -t ssh,inventory
ansible -m ping jump
ssh -F .ssh/config jump
ansible-playbook setup.yml -t user,lvm
```

- プレイブックを実行すると、
  - 鍵とconfigが生成される ... roles/ssh
  - hosts が生成される ... roles/inventory
  - hosts が生成される ... roles/users
  - lvmスナップショット が作成される ... roles/lvm

### hosts.csv

ホスト名, 接続名, sshユーザ, sshパスワード, rootパスワード, グループ名 の順に記載する

| 変数 | host_name | con_name | ssh_user | ssh_pass | become_pass | group_name |
| --- | --- | --- | --- | --- | --- | --- |
| 説明 | ホスト名 | 接続名 | sshユーザ | sshパスワード | rootパスワード | グループ名 |
| 必須引数 | o | o | o | - | - | o |
| 任意引数 | - | - | - | o | o | - |

``` csv
host_name,con_name,ssh_user,ssh_pass,become_pass,group_name
jump,172.16.0.1,ubuntu,jump_ssh_pass,jump_become_pass,other
web1,172.16.1.1,ubuntu,web1_ssh_pass,web1_become_pass,web
web2,172.16.1.2,ubuntu,web2_ssh_pass,web2_become_pass,web
db1,172.16.2.1,db_user,db1_ssh_pass,db1_become_pass,db
db2,172.16.2.2,db_user,db2_ssh_pass,db2_become_pass,db
```

### roles/ssh

``` log
ls -al .ssh/
cat .ssh/config
cat hosts
```

- ホストごとに鍵が生成される

``` log
# ls -al .ssh/
total 52
drwx------ 2 root root 4096 Jul 20 02:34 .
drwxr-xr-x 6 root root 4096 Jul 20 02:34 ..
-rw------- 1 root root  663 Jul 20 02:34 config
-rw------- 1 root root 3401 Jul 20 02:34 db1.id_rsa
-rw-r--r-- 1 root root  754 Jul 20 02:34 db1.id_rsa.pub
-rw------- 1 root root 3401 Jul 20 02:34 db2.id_rsa
-rw-r--r-- 1 root root  754 Jul 20 02:34 db2.id_rsa.pub
-rw------- 1 root root 3401 Jul 20 02:34 jump.id_rsa
-rw-r--r-- 1 root root  754 Jul 20 02:34 jump.id_rsa.pub
-rw------- 1 root root 3401 Jul 20 02:34 web1.id_rsa
-rw-r--r-- 1 root root  754 Jul 20 02:34 web1.id_rsa.pub
-rw------- 1 root root 3401 Jul 20 02:34 web2.id_rsa
-rw-r--r-- 1 root root  754 Jul 20 02:34 web2.id_rsa.pub
```

- config は hosts.csv の情報を元に生成される

``` config:.ssh/config
Host *
  ServerAliveInterval 59
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null

# generated by ansible
Host jump
  Hostname 172.16.0.1
  User ubuntu
  IdentityFile .ssh/jump.id_rsa
Host web1
  Hostname 172.16.1.1
  User ubuntu
  IdentityFile .ssh/web1.id_rsa
Host web2
  Hostname 172.16.1.2
  User ubuntu
  IdentityFile .ssh/web2.id_rsa
Host db1
  Hostname 172.16.2.1
  User db_user
  IdentityFile .ssh/db1.id_rsa
Host db2
  Hostname 172.16.2.2
  User db_user
  IdentityFile .ssh/db2.id_rsa
```

### roles/inventory

- hosts は下記のように生成される
- 1ホスト は 1グループにだけ追加できる

``` ini:hosts
[all:children]
other
web
db

[other]
jump ansible_user=ubuntu ansible_password=jump_ssh_pass ansible_become_password=jump_become_pass

[web]
web1 ansible_user=ubuntu ansible_password=web1_ssh_pass ansible_become_password=web1_become_pass
web2 ansible_user=ubuntu ansible_password=web2_ssh_pass ansible_become_password=web2_become_pass

[db]
db1 ansible_user=db_user ansible_password=db1_ssh_pass ansible_become_password=db1_become_pass
db2 ansible_user=db_user ansible_password=db2_ssh_pass ansible_become_password=db2_become_pass
```

※ホストが複数のグループに所属させたい場合、手動で hosts を編集してください

```
# インベントリを作成せず、鍵だけ生成する
ansible-playbook -v setup.yml -t ssh

# 手動で hosts を 作成, 編集する
vim hosts
```

### roles/inventory

hosts の ansible_user, ansible_password, ansible_become_password をもとに、  
ssh 接続し、root ユーザに昇格して、下記を実行する

- authorized_keys を追記 (add_authorized_key.yml)
- パスなしsudo を許可 (allow_sudo.yml)
- sshの rootログインを許可 (allow_root_login.yml)

### 鍵の上書き

既に手元に秘密鍵があって、それを使いたい場合、  
鍵を生成した後に、上書きすればOK

```
cd <project_dir>/<project_name>/
ansible-playbook -v setup.yml -t ssh,inventory
\cp ~/.ssh/hoge_rsa <project_dir>/<project_name>/.ssh/jump.id_rsa
\cp ~/.ssh/hoge_rsa.pub <project_dir>/<project_name>/.ssh/jump.id_rsa.pub
ssh -F .ssh/config jump
ansible -m ping jump
ansible-playbook -v setup.yml -t user,lvm
```

### lvmスナップショット取得時の状態に戻す

lvmスナップショットを取得時の状態に戻り、  
ssh-initializer のあとに実施した `全ての作業` が消えます

```
ansible-playbook -v merge_snapshot.yml -l jump
```

### ~/.ssh/config を Include でファイル分割する

毎回 ssh -F <project_dir>/<project_name>/.ssh/config するのは面倒

~/.ssh/config で Include するようにしたら、
どこのディレクトリにいても ssh 可能になる

```
cat ~/.ssh/config | grep '^Include' || sed -i '1i Include ~/.ssh/conf.d/**/*\n' ~/.ssh/config
mkdir -p ~/.ssh/conf.d/<project_name>
ln -s <project_dir>/<project_name>/.ssh/config ~/.ssh/conf.d/<project_name>/
```
