---
title: PVE 常用脚本
date: "2024-09-06"
weight: 4
---

```shell
#!/bin/bash
# capacity.sh

file_path="/sys/class/power_supply/BAT1/capacity"

file_content=$(cat "$file_path")

echo "Battery capacity is ${file_content}%."

```

```shell
#!/bin/bash
# clone_vms.sh

for i in {211..215}
do
  qm clone 1102 $i --name "node$i"
done
```

```shell
#!/bin/bash
# restore_cluster.sh

for i in {211..215}
do
  qm rollback $i initial_state
done
```

```shell
#!/bin/bash
# start_cluster.sh

for i in {211..215}
do
  qm start $i
  echo "VM $i started."
done
```

```shell
#!/bin/bash
# stop_cluster.sh

for i in {211..215}
do
  qm start $i
  echo "VM $i stopped."
done
```
