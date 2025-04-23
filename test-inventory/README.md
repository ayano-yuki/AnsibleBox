# test-inventory
Ansibleの環境構築と、VPSへの接続を確認するために実行

**前提条件**
- SSH用のKeyが既に配布し終わっている

## 内容
### 0. Ansibleをインストールする
下記のコマンドでAnsibleをインストールする
```sh
sudo apt update
sudo apt install ansible -y
```

### 1. `inventory.yaml` を作成する
```yaml
all:
  hosts:
    conoha:
      ansible_host: vps_ip
      ansible_user: vps_user_name
      ansible_ssh_private_key_file: "ssh_private_key_file_path"
```

### 2. 接続テスト
下記のコマンドを実行する
```sh
ansible -i inventory.yaml conoha -m ping
```

成功すると、以下のレスポンスが返ってくる
```javascript
conoha | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

## よくあるエラー
### SSHのKeyのパーミッションが適切でない
以下のコマンドで、パーミッションを調整する
```sh
chmod 600 ssh_private_key_file_path
```