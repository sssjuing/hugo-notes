---
title: PVE 常用脚本
date: "2024-09-06"
---

---

### 查看电池电量

```bash
#!/bin/bash
# capacity.sh

file_path="/sys/class/power_supply/BAT1/capacity"

file_content=$(cat "$file_path")

echo "Battery capacity is ${file_content}%."

```

### 克隆虚拟机

```bash
#!/bin/bash
# clone_vms.sh

for i in {211..215}
do
  qm clone 1102 $i --name "node$i"
done
```

### 批量回滚虚拟机

```bash
#!/bin/bash
# restore_cluster.sh

for i in {211..215}
do
  qm rollback $i initial_state
done
```

### 启动虚拟机集群

```bash
#!/bin/bash
# start_cluster.sh

for i in {211..215}
do
  qm start $i
  echo "VM $i started."
done
```

### 关闭虚拟机集群

```bash
#!/bin/bash
# stop_cluster.sh

for i in {211..215}
do
  qm start $i
  echo "VM $i stopped."
done
```

### 修改虚拟机集群的网关

```bash
#!/bin/bash
# batch_change_gateway.sh

for i in {221..225}
do
  ssh 192.168.7.$i 'sed -i "14c\gateway 192.168.7.1" /etc/network/interfaces'
  echo "VM 192.168.7.$i has been set."
done
```
