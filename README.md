# Ansible

http://docs.ansible.com/

## Ansible とは

Python で書かれた構成管理ツールのひとつ

設定ファイルに従って、ソフトウェアのインストールや、サービスの起動などを自動的に実行する

その他の構成管理ツールとして、 Chef や Puppet などがある


## Ansible の特徴

### 管理対象のマシンにエージェント（クライアントツール）を入れる必要がない

管理対象マシンに Python が入っていて SSH 接続ができればよい

主な Linux ディストリビューションには標準でインストールされているので、特別な設定を行う必要はない

### YAML 形式の設定ファイル

シンプルなフォーマットで設定が記述できる


## インストール

### 制御マシン要件
Python 2.6 or 2.7

Windows はサポート対象外

### 管理ノード要件
Python 2.4 以上

※ 管理ノードで SELinux が有効な場合は、Ansible で最初に libselinux-python をインストールする必要がある

※ Ansible は Pyhone 3 には対応していない


### yum でインストール
RHEL や CentOS では EPEL リポジトリから Ansible の RPM パッケージをインストールできる

EPEL を利用するには、 EPEL ページから自身の環境に対応した "epel-release" パッケージをダウンロードしてインストールすれば良い

例） CentOS 7 の場合
```
$ wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
$ sudo rpm -Uvh epel-release-latest-7.noarch.rpm
$ sudo yum -y install ansible
```

### apt でインストール

```
$ sudo apt-get install software-properties-common
$ sudo apt-add-repository ppa:ansible/ansible
$ sudo apt-get update
$ sudo apt-get install ansible
```

### pip でインストール
```
$ sudo easy_install pip
$ sudo pip install ansible
```
必要に応じて依存パッケージをインストールする

### インストールの確認
```
$ ansible —version
```

## 基本的な使い方

```
$ ansible <host-pattern> [options]
```

### Inventory
アクセスしたいホストの情報をインベントリファイルに記述する
```
$ vi hosts

// グループを指定できる
[web]
192.168.33.10

[local] // ローカルマシンに接続する場合
127.0.0.1 ansible_connection=local
```


デフォルトでは、 `/etc/ansible/hosts` を参照するが、 `-i` オプションで直接指定することもできる

または、 `ansible.cfg` で設定する
```
$ vi ansible.cfg

[defaults]
hostfile = ./hosts
```

### 疎通確認
```
$ ansible web -i hosts -m ping

192.168.33.10 | success >> {
    "changed": false,
    "ping": "pong"
}

$ ansible all -i hosts -m ping

192.168.33.10 | success >> {
    "changed": false,
    "ping": "pong"
}

127.0.0.1 | success >> {
    "changed": false,
    "ping": "pong"
}
```

### 任意のコマンドを実行
管理ノードで任意のコマンドを実行したい場合は、 `-a` オプションを使用する
```
$ ansible web -i hosts -a "cat /etc/redhat-release"

192.168.33.10 | success | rc=0 >>
CentOS Linux release 7.1.1503 (Core)
```

### 主なコマンドラインオプション

* `-i INVENTORY, --inventory-file=INVENTORY`<br>
    インベントリファイルを指定

* `-C, --check`<br>
    ドライラン


## PlayBook

LAMP 環境を構築する Playbook
```
---
- hosts: web # ホストグループ
  sudo: yes # SUDO 実行の有効化

  tasks:
    - name: install libselinux-python
      yum: name=libselinux-python state=present

    - name: install httpd
      yum: name=httpd state=latest
      notify:
        - restart_httpd

    - name: install php
      yum: name={{item}} state=latest # 変数を設定できる
      with_items:
        - php
        - php-pdo
        - php-mysqlnd
        - php-mbstring
      notify:
        - restart_httpd

    - name: install mariadb
      yum: name={{item}} state=latest
      with_items:
        - mariadb
        - mariadb-server
      notify:
        - restart_mariadb

  handlers: # タスク終了時に変更があれば実行
    - name: restart_httpd
      service: name=httpd state=restarted enabled=yes

    - name: restart_mariadb
      service: name=mariadb state=restarted enabled=yes

```

実行
```
$ ansible-playbook -i ./hosts ./playbook.yml
```

実行結果
```
PLAY [web] ********************************************************************

GATHERING FACTS ***************************************************************
ok: [192.168.33.10]

TASK: [install libselinux-python] *********************************************
ok: [192.168.33.10]

TASK: [install httpd] *********************************************************
changed: [192.168.33.10]

TASK: [install php] ***********************************************************
changed: [192.168.33.10] => (item=php,php-pdo,php-mysqlnd,php-mbstring)

TASK: [install mariadb] *******************************************************
changed: [192.168.33.10] => (item=mariadb,mariadb-server)

NOTIFIED: [restart_httpd] *****************************************************
changed: [192.168.33.10]

NOTIFIED: [restart_mariadb] ***************************************************
changed: [192.168.33.10]

PLAY RECAP ********************************************************************
192.168.33.10              : ok=7    changed=5    unreachable=0    failed=0
```

もう一度実行（冪等性が保たれる）
```
PLAY [web] ********************************************************************

GATHERING FACTS ***************************************************************
ok: [192.168.33.10]

TASK: [install libselinux-python] *********************************************
ok: [192.168.33.10]

TASK: [install httpd] *********************************************************
ok: [192.168.33.10]

TASK: [install php] ***********************************************************
ok: [192.168.33.10] => (item=php,php-pdo,php-mysqlnd,php-mbstring)

TASK: [install mariadb] *******************************************************
ok: [192.168.33.10] => (item=mariadb,mariadb-server)

PLAY RECAP ********************************************************************
192.168.33.10              : ok=5    changed=0    unreachable=0    failed=0
```

## Module
Ansible には多くの標準モジュールが用意されている

### よく使うモジュール

#### [shell](http://docs.ansible.com/shell_module.html)
ノード上でコマンドを実行する

#### [copy](http://docs.ansible.com/copy_module.html)
管理ノード上にファイルをコピーする

####[file](http://docs.ansible.com/file_module.html)
ファイルの属性を設定する

#### [yum](http://docs.ansible.com/yum_module.html)
yum でパッケージを管理する

#### [service](http://docs.ansible.com/service_module.html)
サービスを管理する

#### [git](http://docs.ansible.com/git_module.html)
Git でファイルやソフトウェアをデプロイする



