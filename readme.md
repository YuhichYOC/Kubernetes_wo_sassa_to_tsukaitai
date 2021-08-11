# 前提
- ハードウェア : Raspberry PI 4B, 3 機
- OS : 3 機とも Ubuntu 20.04
- master 機 ( 1gouki ) & ノード機 ( 2gouki, 3gouki )
- 以下の手順はハードウェアや OS により変動します。特に手順 4-2 は Raspberry PI 以外では必要ないかもしれない
    - ぶっちゃけ半年後には使えなくなってそう。まとめれば 18 行になっちゃう手順を公式ドキュメントが説明できてない。長々クドクド書いてるけど、試行錯誤の途中なんでしょう

# Kubernetes をとりあえずサッサと使いたい場合の手順

1. sudo apt install -y docker.io
1. docker 実行時 cgroup の変更
1. 参照先リポジトリの追加
    1. sudo apt install -y apt-transport-https
    1. sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
    1. echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
    1. sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    1. sudo apt-get update ( 404 が何回か返ってくる / 404 が無くなるまで繰り返したほうがいい )
1. kubernetes インストール
    1. sudo apt-get install -y kubectl kubelet kubeadm
    1. CGROUP_MEMORY の有効化
1. 再起動
1. master 側での初期化
    1. sudo kubeadm init --pod-network-cidr=xxx.xxx.0.0/16 ( master 機の NIC と被らないセグメントにする )
    1. 6-1 でコマンド戻り値に記載されている手順に従い kubectl コマンド実行環境を整える
    1. sudo kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
1. 2gouki & 3gouki で手順 1 - 手順 4 を行う
1. 2gouki & 3gouki を 6-1 でコマンド戻り値に記載されている手順に従いクラスターへ追加する

---
## 手順 2. docker が使う cgroup について
- デフォルトでは docker は cgroupfs という cgroup を使用している / 一方 kubernetes は systemd の cgroup を使用しようとする
- docker が使用する cgroup を systemd へ片寄せしないと kubernetes がうまく動作しない
- 仕方がないのでファイル /etc/docker/daemon.json を作成・以下の内容を記入
```json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
```
※ docker が cgroupfs を使用しているのには正当な理由がある

---
## 手順 4. CGROUP_MEMORY
- Raspberry PI 4B, Ubuntu 20.04 の場合はこの手順が必要  
起動オプションを追加・ファイル /boot/firmware/cmdline.txt を以下のように変更
```bash
net.ifnames=0 dwc_otg.lpm_enable=0 console=serial0,115200 console=tty1 root=LABEL=writable rootfstype=ext4 elevator=deadline rootwait fixrtc
```
↓↓↓
```bash
net.ifnames=0 dwc_otg.lpm_enable=0 console=serial0,115200 console=tty1 root=LABEL=writable rootfstype=ext4 elevator=deadline rootwait fixrtc cgroup_enable=memory cgroup_memory=1
```
- 再起動後に /proc/cgroups の内容が以下のように変わる
```bash
ubuntu@1gouki:~$ cat /proc/cgroups
#subsys_name    hierarchy       num_cgroups     enabled
cpuset  3       1       1
cpu     4       93      1
cpuacct 4       93      1
blkio   7       93      1
memory  0       101     0
devices 8       93      1
freezer 9       2       1
net_cls 6       1       1
perf_event      10      1       1
net_prio        6       1       1
pids    2       99      1
rdma    5       1       1
```
↓↓↓
```bash
ubuntu@1gouki:~$ cat /proc/cgroups
#subsys_name    hierarchy       num_cgroups     enabled
cpuset  9       1       1
cpu     6       42      1
cpuacct 6       42      1
blkio   11      42      1
memory  10      97      1
devices 2       42      1
freezer 4       2       1
net_cls 7       1       1
perf_event      8       1       1
net_prio        7       1       1
pids    3       47      1
rdma    5       1       1
```
---
## 手順 5. master 側での初期化
- --pod-network-cidr 指定なしで kubeadm init を実行すると、flannel がうまく動作しない。なんでだろう
```bash
ubuntu@1gouki:~$ kubectl logs -n kube-system kube-flannel-ds-7rxss
I0811 04:57:23.861633       1 main.go:520] Determining IP address of default interface
I0811 04:57:23.956550       1 main.go:533] Using interface with name eth0 and address yyy.yyy.0.1
I0811 04:57:23.956667       1 main.go:550] Defaulting external address to interface address (yyy.yyy.0.1)
W0811 04:57:23.956778       1 client_config.go:608] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
I0811 04:57:25.157913       1 kube.go:116] Waiting 10m0s for node controller to sync
I0811 04:57:25.158133       1 kube.go:299] Starting kube subnet manager
I0811 04:57:26.158661       1 kube.go:123] Node controller sync successful
I0811 04:57:26.158768       1 main.go:254] Created subnet manager: Kubernetes Subnet Manager - 1gouki.localdomain.com
I0811 04:57:26.158808       1 main.go:257] Installing signal handlers
I0811 04:57:26.159638       1 main.go:392] Found network config - Backend type: vxlan
I0811 04:57:26.159902       1 vxlan.go:123] VXLAN config: VNI=1 Port=0 GBP=false Learning=false DirectRouting=false
E0811 04:57:26.161406       1 main.go:293] Error registering network: failed to acquire lease: node "1gouki.localdomain.com" pod cidr not assigned
I0811 04:57:26.161625       1 main.go:372] Stopping shutdownHandler...
W0811 04:57:26.162153       1 reflector.go:424] github.com/flannel-io/flannel/subnet/kube/kube.go:300: watch of *v1.Node ended with: an error on the server ("unable to decode an event from the watch stream: context canceled") has prevented the 
request from succeeding

==== START logs for container kube-flannel of pod kube-system/kube-flannel-ds-7rxss ====
I0811 04:57:23.861633       1 main.go:520] Determining IP address of default interface
I0811 04:57:23.956550       1 main.go:533] Using interface with name eth0 and address yyy.yyy.0.1
I0811 04:57:23.956667       1 main.go:550] Defaulting external address to interface address (yyy.yyy.0.1)
W0811 04:57:23.956778       1 client_config.go:608] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
I0811 04:57:25.157913       1 kube.go:116] Waiting 10m0s for node controller to sync
I0811 04:57:25.158133       1 kube.go:299] Starting kube subnet manager
I0811 04:57:26.158661       1 kube.go:123] Node controller sync successful
I0811 04:57:26.158768       1 main.go:254] Created subnet manager: Kubernetes Subnet Manager - 1gouki.localdomain.com
I0811 04:57:26.158808       1 main.go:257] Installing signal handlers
I0811 04:57:26.159638       1 main.go:392] Found network config - Backend type: vxlan
I0811 04:57:26.159902       1 vxlan.go:123] VXLAN config: VNI=1 Port=0 GBP=false Learning=false DirectRouting=false
E0811 04:57:26.161406       1 main.go:293] Error registering network: failed to acquire lease: node "1gouki.localdomain.com" pod cidr not assigned
I0811 04:57:26.161625       1 main.go:372] Stopping shutdownHandler...
W0811 04:57:26.162153       1 reflector.go:424] github.com/flannel-io/flannel/subnet/kube/kube.go:300: watch of *v1.Node ended with: an error on the server ("unable to decode an event from the watch stream: context canceled") has prevented the 
request from succeeding
==== END logs for container kube-flannel of pod kube-system/kube-flannel-ds-7rxss ====
```
- 「cidr not assigned」を手掛かりに kubeadm init に --pod-network-cidr オプションを付けたら上手くいった。要するにただのまぐれ
- --kubeconfig は指定している ( 5-2 で作成 ) はずだし、--master はググっても情報がない。こんなのどう対処すればいいの
