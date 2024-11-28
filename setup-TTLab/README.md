# setup-TTLab
- 玉城研のUbuntuのパソコンの環境構築を行う
  - sudo を使えるようにする

# 前提条件
- Ansibleの動作環境が構築済み
  1. gitのインストール
  2. pythonのインストール
  3. ansibleのインストール

# 動作確認環境
- a

# 使い方
## vault.ymlの作成
を以下のようなvault.ymlを作成し、暗号化する
```text
# vault.yml
become_pass: "root_password"
```

```bash
# 暗号化（パスワード：pass）
ansible-vault encrypt vault.yml
```

```bash
# 復号（パスワード：pass）
ansible-vault decrypt vault.yml
```

## 実行
```bash
ansible-playbook ---ask-vault-pass vault.yml
```