SUID/SGID 檔案檢查是 Linux/Unix 系統中非常重要的資安議題。讓我詳細解釋：

## 什麼是 SUID/SGID？

**SUID (Set User ID)** 和 **SGID (Set Group ID)** 是特殊的檔案權限位元：

- **SUID**：當執行檔案時，程序會以檔案擁有者的身份執行，而非執行者的身份
- **SGID**：當執行檔案時，程序會以檔案所屬群組的身份執行

例如，`/usr/bin/passwd` 通常有 SUID 位元，讓一般使用者可以修改密碼檔案。

## 資安風險與危害

### 1. **權限提升攻擊**
- 攻擊者可利用有 SUID root 權限的程式進行權限提升
- 如果 SUID 程式有漏洞，攻擊者可能獲得 root 權限

### 2. **不當的權限設定**
- 不需要的程式被設定 SUID/SGID
- 自製或第三方程式被錯誤設定特殊權限

### 3. **系統完整性威脅**
- 惡意程式偽裝成系統程式並設定 SUID
- 後門程式利用 SUID 維持高權限存取

## 檢查方法

```bash
# 尋找系統中所有 SUID 檔案
find / -perm -4000 -type f 2>/dev/null

# 尋找所有 SGID 檔案
find / -perm -2000 -type f 2>/dev/null

# 同時尋找 SUID 和 SGID
find / -perm -6000 -type f 2>/dev/null

# 檢查特定目錄
find /usr/bin -perm -4000 -ls
```

## 資安最佳實務

### 1. **定期稽核**
- 建立已知 SUID/SGID 檔案清單
- 定期比對是否有新增的可疑檔案
- 監控系統變更

### 2. **最小權限原則**
- 移除不必要的 SUID/SGID 權限
- 評估每個特殊權限檔案的必要性

### 3. **監控與警報**
- 設定檔案完整性監控 (如 AIDE、Tripwire)
- 當發現新的 SUID/SGID 檔案時發出警報

### 4. **安全配置**
```bash
# 移除不必要的 SUID 權限
chmod u-s /path/to/file

# 檢查檔案權限
ls -la /usr/bin/passwd
# 輸出類似：-rwsr-xr-x (注意 's' 表示 SUID)
```

## 常見合法的 SUID 程式

- `/usr/bin/passwd` - 修改密碼
- `/usr/bin/su` - 切換使用者
- `/usr/bin/sudo` - 執行管理員命令
- `/bin/ping` - 網路連線測試

定期檢查 SUID/SGID 檔案是系統安全稽核的重要環節，可以有效防範權限提升攻擊和發現潛在的安全威脅。建議將此檢查納入日常的資安維護作業中。



以下是常見合法的 SUID/SGID 程式清單，你可以用來比對系統中是否有異常設置：

## 常見合法 SUID 程式 (權限 4755)

### 系統核心工具
```
/bin/su
/bin/ping
/bin/ping6
/bin/mount
/bin/umount
/bin/fusermount
```

### 用戶管理工具
```
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/sudo
/usr/bin/pkexec
```

### 網路工具
```
/usr/bin/traceroute6.iputils
/usr/sbin/pppd
/usr/bin/arping
```

### 其他系統工具
```
/usr/bin/at
/usr/bin/crontab
/usr/libexec/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
```

## 常見合法 SGID 程式 (權限 2755)

### 系統工具
```
/usr/bin/wall
/usr/bin/write
/usr/bin/chage
/usr/bin/expiry
```

### 郵件相關
```
/usr/bin/mail
/usr/sbin/postdrop
/usr/sbin/postqueue
```

### 其他
```
/usr/bin/ssh-agent
/usr/bin/locate
/usr/bin/mlocate
```

## 檢查腳本範例

建立一個檢查腳本來比對：## 使用方式

1. **儲存腳本**：
```bash
chmod +x suid_check.sh
sudo ./suid_check.sh
```

2. **手動檢查指令**：
```bash
# 快速檢查可疑檔案
find / -perm -4000 -type f 2>/dev/null | grep -v -E "(bin/su|bin/ping|usr/bin/passwd|usr/bin/sudo)"

# 詳細檢查特定檔案
ls -la /path/to/suspicious/file
stat /path/to/suspicious/file
```

## 重要注意事項

### 系統差異
- **不同發行版**可能有不同的合法程式路徑
- **Ubuntu/Debian** vs **CentOS/RHEL** 路徑可能不同
- 某些程式可能安裝在 `/usr/local/` 下

### 檢查重點
1. **擁有者是 root** 的 SUID 程式最危險
2. **位於不常見路徑**的程式 (如 `/tmp/`, `/var/tmp/`)
3. **最近修改**的檔案
4. **檔案名稱可疑**的程式

### 立即處理原則
如果發現可疑檔案：
```bash
# 立即移除特殊權限
chmod u-s /path/to/suspicious/file
chmod g-s /path/to/suspicious/file

# 備份檔案供分析
cp /path/to/suspicious/file /tmp/evidence/

# 如確認為惡意，完全刪除
rm /path/to/suspicious/file
```

建議定期執行這個檢查，並將結果記錄下來，以便追蹤系統變化。




```
#!/bin/bash

# SUID/SGID 檔案檢查腳本
# 用於識別系統中不合法的特殊權限設置

echo "=== SUID/SGID 安全檢查工具 ==="
echo "檢查時間: $(date)"
echo

# 已知合法的 SUID 程式清單
KNOWN_SUID=(
    "/bin/su"
    "/bin/ping"
    "/bin/ping6"
    "/bin/mount"
    "/bin/umount"
    "/bin/fusermount"
    "/usr/bin/passwd"
    "/usr/bin/gpasswd" 
    "/usr/bin/newgrp"
    "/usr/bin/chsh"
    "/usr/bin/chfn"
    "/usr/bin/sudo"
    "/usr/bin/pkexec"
    "/usr/bin/at"
    "/usr/bin/crontab"
    "/usr/bin/traceroute6.iputils"
    "/usr/sbin/pppd"
    "/usr/bin/arping"
    "/usr/libexec/dbus-daemon-launch-helper"
    "/usr/lib/openssh/ssh-keysign"
    "/usr/lib/x86_64-linux-gnu/openssh/ssh-keysign"
    "/usr/lib/dbus-1.0/dbus-daemon-launch-helper"
)

# 已知合法的 SGID 程式清單
KNOWN_SGID=(
    "/usr/bin/wall"
    "/usr/bin/write"
    "/usr/bin/chage"
    "/usr/bin/expiry"
    "/usr/bin/mail"
    "/usr/sbin/postdrop"
    "/usr/sbin/postqueue"
    "/usr/bin/ssh-agent"
    "/usr/bin/locate"
    "/usr/bin/mlocate"
    "/usr/bin/bsd-write"
)

# 函數：檢查檔案是否在已知清單中
is_known_file() {
    local file="$1"
    local type="$2"
    
    if [ "$type" == "suid" ]; then
        for known in "${KNOWN_SUID[@]}"; do
            if [ "$file" == "$known" ]; then
                return 0
            fi
        done
    elif [ "$type" == "sgid" ]; then
        for known in "${KNOWN_SGID[@]}"; do
            if [ "$file" == "$known" ]; then
                return 0
            fi
        done
    fi
    return 1
}

echo "1. 檢查 SUID 檔案..."
echo "=============================="

# 尋找所有 SUID 檔案
SUID_FILES=$(find / -perm -4000 -type f 2>/dev/null)
SUSPICIOUS_SUID=()

if [ -z "$SUID_FILES" ]; then
    echo "未發現任何 SUID 檔案"
else
    while IFS= read -r file; do
        if is_known_file "$file" "suid"; then
            echo "✓ [合法] $file"
        else
            echo "⚠ [可疑] $file"
            SUSPICIOUS_SUID+=("$file")
        fi
    done <<< "$SUID_FILES"
fi

echo
echo "2. 檢查 SGID 檔案..."
echo "=============================="

# 尋找所有 SGID 檔案
SGID_FILES=$(find / -perm -2000 -type f 2>/dev/null)
SUSPICIOUS_SGID=()

if [ -z "$SGID_FILES" ]; then
    echo "未發現任何 SGID 檔案"
else
    while IFS= read -r file; do
        if is_known_file "$file" "sgid"; then
            echo "✓ [合法] $file"
        else
            echo "⚠ [可疑] $file"
            SUSPICIOUS_SGID+=("$file")
        fi
    done <<< "$SGID_FILES"
fi

echo
echo "3. 檢查結果摘要"
echo "=============================="

if [ ${#SUSPICIOUS_SUID[@]} -eq 0 ] && [ ${#SUSPICIOUS_SGID[@]} -eq 0 ]; then
    echo "✅ 未發現可疑的 SUID/SGID 檔案"
else
    echo "🚨 發現可疑檔案，需要進一步調查："
    
    if [ ${#SUSPICIOUS_SUID[@]} -gt 0 ]; then
        echo
        echo "可疑 SUID 檔案："
        for file in "${SUSPICIOUS_SUID[@]}"; do
            echo "  - $file"
            # 顯示檔案詳細資訊
            ls -la "$file" 2>/dev/null
            echo "    擁有者: $(stat -c '%U' "$file" 2>/dev/null)"
            echo "    修改時間: $(stat -c '%y' "$file" 2>/dev/null)"
            echo
        done
    fi
    
    if [ ${#SUSPICIOUS_SGID[@]} -gt 0 ]; then
        echo
        echo "可疑 SGID 檔案："
        for file in "${SUSPICIOUS_SGID[@]}"; do
            echo "  - $file"
            # 顯示檔案詳細資訊
            ls -la "$file" 2>/dev/null
            echo "    群組: $(stat -c '%G' "$file" 2>/dev/null)"
            echo "    修改時間: $(stat -c '%y' "$file" 2>/dev/null)"
            echo
        done
    fi
fi

echo "4. 建議後續動作"
echo "=============================="
echo "對於可疑檔案，建議："
echo "1. 檢查檔案來源和用途"
echo "2. 驗證檔案完整性 (checksums)"
echo "3. 如確認為惡意檔案，立即移除 SUID/SGID 權限："
echo "   chmod u-s <檔案路徑>  # 移除 SUID"
echo "   chmod g-s <檔案路徑>  # 移除 SGID"
echo "4. 調查檔案建立時間和可能的入侵途徑"
echo "5. 更新系統和進行全面安全掃描"

echo
echo "檢查完成: $(date)"
```
