# Work-Ansible

# Ansible
Ansibleは、Red Hat社が開発・管理しているオープンソースの構成管理ツールで、ITインフラストラクチャのプロビジョニング、構成管理、アプリケーションデプロイメントを自動化するためのオープンソースソフトウェアです。これは、YAML形式で記述されたPlaybookと呼ばれる設定ファイルを用いて、複数のホストに対して一貫性のある環境構築を行います。

# Ansibleのセットアップ
## WSLを使う場合
1. Ubuntu内でAnsibleをインストール
    ```sh
    sudo apt update
    sudo apt install ansible -y
    ```
2. **インベントリファイルを作成**
    `inventory.yaml` というファイルを作成する
3. 接続確認
    ```sh
    ansible -i inventory.yaml conoha -m ping
    ```
    成功すると、以下のようなレスポンスが返ってくる

    ```javascript
    your-vps-ip | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
    ```
4. Playbookを使った自動化
   例えば、VPSに nginx をインストールするPlaybookを作成：

   1. `playbook.yml` を作成： 
        ```yaml
        - hosts: conoha
        become: yes
        tasks:
            - name: Install nginx
            apt:
                name: nginx
                state: present
        ```
   2. Playbookを実行：
        ```sh
        ansible-playbook -i inventory.ini playbook.yml
        ```