---
title: Synology NVME SSD smart check
date: 2025-03-16
taxonomies:
  categories: [selfhost]
  tags: [synology]
---

群晖对于 HDD 有自带的定时 smart check 功能。但由于其本身不支持 NVME 设备作为存储池，(似乎)也没有对 NVME 设备的定时 smart check 功能。

系统自带 nvme-cli，为了实现定时检测，可以自己写一个检测脚本。

参考 [IBM - Running the Linux smart-log command to check the amount of remaining life in NVMe devices](https://www.ibm.com/docs/en/power10/9786-42H?topic=devices-running-linux-smart-log-command)

```bash
#!/bin/bash
# Usage: ./nvme-smart-check.sh /dev/nvme0

if [ "$#" -ne 1 ]; then
  echo "Usage: $0 <device_path>"
  exit 1
fi

dev="$1"
echo "Checking $dev..."
output=$(nvme smart-log "$dev")

if [ $? -ne 0 ]; then
  echo "ERROR: Failed to read smart-log for $dev. Details: $output"
  exit 1
fi

# Extract the relevant fields from the output
critical_warning=$(echo "$output" | grep -i "^critical_warning" | awk -F: '{gsub(/ /,"",$2); print $2}')
available_spare=$(echo "$output" | grep -i "^available_spare[[:space:]]*:" | head -n 1 | awk -F: '{gsub(/[% ]/,"",$2); print $2}')
available_spare_threshold=$(echo "$output" | grep -i "^available_spare_threshold" | awk -F: '{gsub(/[% ]/,"",$2); print $2}')
media_errors=$(echo "$output" | grep -i "^media_errors" | awk -F: '{gsub(/ /,"",$2); print $2}')
err_log=$(echo "$output" | grep -i "^num_err_log_entries" | awk -F: '{gsub(/ /,"",$2); print $2}')
percentage_used=$(echo "$output" | grep -i "^percentage_used" | awk -F: '{gsub(/[% ]/,"",$2); print $2}')

# Initialize the unhealthy flag (0 = healthy, 1 = health issue detected)
unhealthy=0

if [ "$critical_warning" -ne 0 ]; then
    echo "$dev: Critical warning is not 0."
    unhealthy=1
fi

if [ "$available_spare" -lt "$available_spare_threshold" ]; then
    echo "$dev: Available spare ($available_spare%) is less than threshold ($available_spare_threshold%)."
    unhealthy=1
fi

if [ "$media_errors" -ne 0 ]; then
    echo "$dev: Media errors detected."
    unhealthy=1
fi

if [ "$err_log" -ne 0 ]; then
    echo "$dev: Error log entries detected."
    unhealthy=1
fi

if [ "$percentage_used" -gt 10 ]; then
    echo "$dev: Warning - percentage used is greater than 10% ($percentage_used%)."
    unhealthy=1
fi

echo "$dev: Check complete."

if [ $unhealthy -eq 1 ]; then
    echo "$dev: One or more health issues detected."
    exit 1
else
    echo "$dev: Device is healthy."
    exit 0
fi

```

用任务计划，root 每日运行一次，失败时发邮件即可。
