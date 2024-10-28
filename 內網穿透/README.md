# 使用ZeroTier 實現內網穿透

## 基礎安裝
1. 先去官網建立帳號創建ID(https://www.zerotier.com/)
2. Windows電腦先下載(https://www.zerotier.com/download/)
3. OMV NAS 安裝  curl -s https://install.zerotier.com | sudo bash

## 使用
1. 先加入防火牆 
```
sudo iptables -A INPUT -p tcp --dport 9993 -j ACCEPT

sudo iptables-save | sudo tee /etc/iptables/rules.v4

sudo iptables-save

sudo iptables-save
```
2. 啟動ZeroTier服務，設定開機自動啟動
```
sudo systemctl start zerotier-one

sudo systemctl enable zerotier-one
```
3. 加入ZeroTier網路
```
sudo zerotier-cli join "自己的ID"
#應會回傳200 join OK

```
### 加完ID 要先去你的ID網頁打勾

4. 確認ZeroTier網路狀態
```
sudo zerotier-cli status
```
5. 確認ZeroTier有無P2P成功，如果有的話Peers上面會顯示LEAF，這樣網路會順暢些。
```
sudo zerotier-cli peers
```

### 詳細他人教學
1. (https://ivonblog.com/posts/install-zerotier-on-linux/)

