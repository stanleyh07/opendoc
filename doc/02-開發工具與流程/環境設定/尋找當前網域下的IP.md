- 使用 nmap 尋找 IP
```bash
sudo apt-get install nmap  # 如果沒有 nmap，請先安裝
sudo nmap -sn 192.168.1.0/24 # 尋找 192.168.1.1 ~ 192.168.1.255
sudo nmap -sn 192.168.0.0/16 # 尋找 192.168.x.x 下的 IP
sudo nmap -sn 192.0.0.0/8    # 尋找 192.x.x.x 下的 IP
```

- 如果需要更詳細的資訊，可以選擇 arp-scan
```bash
sudo apt-get install arp-scan  # 先安装 arp-scan

# arp-scan --interface="要scan的網路介面" --localnet
sudo arp-scan --interface="eth0" --localnet
sudo arp-scan --interface="wlx74da385d7a60" --localnet
```
