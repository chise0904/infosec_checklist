## 最高安全標準的危險設置檢查清單

### sudo權限危險設置

**極高風險設置**
```bash
# 檢查這些危險的sudo設置
sudo grep -r "NOPASSWD.*ALL" /etc/sudoers /etc/sudoers.d/
sudo grep -r "ALL.*NOPASSWD" /etc/sudoers /etc/sudoers.d/
sudo grep -r "\!authenticate" /etc/sudoers /etc/sudoers.d/
sudo grep -r "ALL.*ALL.*ALL" /etc/sudoers /etc/sudoers.d/
```

**危險的萬用字元設置**
```bash
sudo grep -r "/bin/\*" /etc/sudoers /etc/sudoers.d/
sudo grep -r "/usr/bin/\*" /etc/sudoers /etc/sudoers.d/
sudo grep -r "\*" /etc/sudoers /etc/sudoers.d/
sudo grep -r "ALL = " /etc/sudoers /etc/sudoers.d/
```

**允許編輯系統檔案的設置**
```bash
sudo grep -r -E "(vi|vim|nano|emacs|sed|awk)" /etc/sudoers /etc/sudoers.d/
sudo grep -r "/etc/sudoers" /etc/sudoers /etc/sudoers.d/
sudo grep -r "/etc/passwd" /etc/sudoers /etc/sudoers.d/
```

### 檔案系統權限危險設置

**SUID/SGID檔案檢查**
```bash
# 檢查異常的SUID檔案
find / -perm -4000 -type f 2>/dev/null | grep -v -E "^/(usr/)?(s)?bin/(su|sudo|ping|mount|umount|passwd|gpasswd|newgrp|chfn|chsh|crontab)$"

# 檢查異常的SGID檔案
find / -perm -2000 -type f 2>/dev/null | grep -v -E "^/(usr/)?(s)?bin/(write|wall|locate|ssh-agent)$"

# 檢查可執行的SUID腳本
find / -perm -4000 -type f -exec file {} \; 2>/dev/null | grep -i script

# 檢查在/tmp和/var/tmp的SUID檔案
find /tmp /var/tmp -perm /6000 -type f 2>/dev/null
```

**World-writable危險檔案**
```bash
# 系統關鍵目錄的world-writable檔案
find /etc /usr /boot /lib* /sbin /bin -perm -002 -type f 2>/dev/null

# 可執行的world-writable檔案
find / -perm -002 -perm /111 -type f 2>/dev/null

# World-writable設定檔
find /etc -name "*.conf" -perm -002 2>/dev/null
find /etc -name "*.cfg" -perm -002 2>/dev/null
```

**危險的檔案所有權**
```bash
# root擁有但其他人可寫的檔案
find / -user root -perm -002 2>/dev/null

# 檢查/root目錄權限
ls -la /root/
ls -la /root/.ssh/

# 檢查系統檔案被非root使用者擁有
find /etc /usr/bin /usr/sbin /bin /sbin ! -user root -type f 2>/dev/null
```

### SSH安全設置檢查

**危險的SSH設置**
```bash
# 檢查/etc/ssh/sshd_config中的危險設置
grep -E "^PermitRootLogin yes" /etc/ssh/sshd_config
grep -E "^PasswordAuthentication yes" /etc/ssh/sshd_config
grep -E "^PermitEmptyPasswords yes" /etc/ssh/sshd_config
grep -E "^X11Forwarding yes" /etc/ssh/sshd_config
grep -E "^AllowUsers" /etc/ssh/sshd_config
grep -E "^Protocol 1" /etc/ssh/sshd_config
```

**SSH金鑰檔案權限**
```bash
# 檢查SSH私鑰權限
find /home -name "id_*" ! -name "*.pub" -perm /044 2>/dev/null
find /root/.ssh -name "id_*" ! -name "*.pub" -perm /044 2>/dev/null

# 檢查authorized_keys權限
find /home -name "authorized_keys" -perm /044 2>/dev/null
find /root -name "authorized_keys" -perm /044 2>/dev/null
```

### 使用者帳號危險設置

**危險的使用者設置**
```bash
# 檢查UID為0的非root帳號
awk -F: '$3 == 0 && $1 != "root" {print $1}' /etc/passwd

# 檢查空密碼帳號
awk -F: '$2 == "" {print $1}' /etc/shadow 2>/dev/null

# 檢查系統帳號有登入shell
awk -F: '$3 < 1000 && $3 > 0 && $7 !~ /(nologin|false)/ {print $1":"$7}' /etc/passwd

# 檢查相同UID的帳號
awk -F: '{print $3}' /etc/passwd | sort -n | uniq -d

# 檢查沒有密碼的帳號
passwd -S -a 2>/dev/null | grep " NP "
```

### K8s/容器危險設置

**K8s權限危險設置**
```bash
# 檢查cluster-admin權限
kubectl get clusterrolebindings -o json | jq -r '.items[] | select(.roleRef.name == "cluster-admin") | "\(.metadata.name): \(.subjects[].name // .subjects[].metadata.name)"'

# 檢查預設ServiceAccount的權限
kubectl get clusterrolebindings -o json | jq -r '.items[] | select(.subjects[]?.name == "default") | .metadata.name'

# 檢查匿名使用者權限
kubectl get clusterrolebindings -o json | jq -r '.items[] | select(.subjects[]?.name == "system:anonymous") | .metadata.name'
```

**容器運行時危險設置**
```bash
# 檢查特權容器
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}{" "}{.metadata.name}{" "}{.spec.containers[*].securityContext.privileged}{"\n"}{end}' | grep true

# 檢查以root運行的Pod
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}{" "}{.metadata.name}{" "}{.spec.securityContext.runAsUser}{"\n"}{end}' | grep -E "(^| )0( |$)"

# 檢查hostNetwork的Pod
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}{" "}{.metadata.name}{" "}{.spec.hostNetwork}{"\n"}{end}' | grep true
```

### 服務和程序危險設置

**以root運行的不必要服務**
```bash
# 檢查以root運行的網路服務
ps aux | grep -E "(httpd|nginx|apache|mysql|postgresql|redis)" | grep "^root"

# 檢查監聽在0.0.0.0的高風險服務
netstat -tlnp | grep "0.0.0.0:" | grep -E ":(21|23|25|53|135|139|445|993|995|1433|3389|5432|5900|6379)"

# 檢查不必要的root程序
ps aux | awk '$1=="root" && $11!~/^\[/ {print $11}' | sort | uniq -c | sort -nr
```

### 網路和防火牆危險設置

**防火牆危險設置**
```bash
# 檢查iptables規則
iptables -L -n | grep -E "(ACCEPT.*0\.0\.0\.0/0|ACCEPT.*anywhere)"

# 檢查UFW狀態
ufw status verbose 2>/dev/null

# 檢查firewalld設置
firewall-cmd --list-all 2>/dev/null
```

**危險的網路服務**
```bash
# 檢查telnet服務
systemctl status telnet 2>/dev/null
netstat -tlnp | grep ":23 "

# 檢查FTP服務
systemctl status vsftpd 2>/dev/null
netstat -tlnp | grep ":21 "

# 檢查SNMP服務
systemctl status snmpd 2>/dev/null
netstat -tlnp | grep ":161 "
```

### 日誌和監控危險設置

**日誌安全問題**
```bash
# 檢查日誌檔案權限
find /var/log -perm -044 -type f 2>/dev/null

# 檢查rsyslog設定
grep -E "^\*\.\* @@" /etc/rsyslog.conf /etc/rsyslog.d/* 2>/dev/null

# 檢查audit設定
systemctl status auditd
```

### 全面性自動檢查腳本

```bash
#!/bin/bash
# 最高安全標準檢查腳本

echo "=== 開始最高安全標準檢查 ==="

echo "1. 檢查危險sudo設置..."
sudo grep -rH -E "(NOPASSWD.*ALL|ALL.*NOPASSWD|\!authenticate|/bin/\*|/usr/bin/\*)" /etc/sudoers /etc/sudoers.d/ 2>/dev/null

echo "2. 檢查異常SUID檔案..."
find / -perm -4000 -type f 2>/dev/null | grep -vE "^/(usr/)?s?bin/(su|sudo|ping|mount|umount|passwd)$"

echo "3. 檢查UID為0的非root帳號..."
awk -F: '$3 == 0 && $1 != "root" {print "ALERT: " $1 " has UID 0"}' /etc/passwd

echo "4. 檢查系統帳號有shell..."
awk -F: '$3 < 1000 && $3 > 0 && $7 !~ /(nologin|false)/ {print "ALERT: " $1 " is system account but has shell: " $7}' /etc/passwd

echo "5. 檢查SSH危險設置..."
grep -E "^PermitRootLogin yes" /etc/ssh/sshd_config && echo "ALERT: Root login enabled"

echo "6. 檢查world-writable系統檔案..."
find /etc /usr /boot -perm -002 -type f 2>/dev/null | head -5

echo "7. 檢查root權限網路服務..."
netstat -tlnp 2>/dev/null | grep "^tcp.*:.*root" | head -5

echo "=== 檢查完成 ==="
```

這些檢查項目涵蓋了最高安全標準的所有危險設置。建議將這些檢查整合成自動化腳本，定期執行並產生報告給資安部門審核。
