

## sudo權限危險設置詳細風險分析

### 1. NOPASSWD設置風險

**檢查項目**
```bash
sudo grep -r "NOPASSWD.*ALL" /etc/sudoers /etc/sudoers.d/
sudo grep -r "ALL.*NOPASSWD" /etc/sudoers /etc/sudoers.d/
```

**風險說明**
- **設置範例**: `user ALL=(ALL) NOPASSWD:ALL`
- **風險**: 使用者可以無需輸入密碼就獲得完整root權限
- **攻擊場景**: 
  - 攻擊者透過SSH金鑰或應用程式漏洞取得該使用者權限
  - 立即可以執行任何root指令，無需破解密碼
  - 內部威脅：離職員工的自動化腳本仍可運作
  - 會話劫持後可直接提權

**為什麼需要改善**
- 移除了sudo最後一道防線（密碼驗證）
- 違反「深度防禦」原則
- 無法阻止自動化攻擊

### 2. !authenticate設置風險

**檢查項目**
```bash
sudo grep -r "\!authenticate" /etc/sudoers /etc/sudoers.d/
```

**風險說明**
- **設置範例**: `Defaults !authenticate`
- **風險**: 全域關閉sudo密碼驗證
- **攻擊場景**:
  - 任何有sudo權限的使用者都不需要密碼驗證
  - 比NOPASSWD更危險，影響所有sudo使用者
  - 惡意腳本可以靜默執行特權操作

**為什麼需要改善**
- 一個設置影響所有sudo使用者
- 完全消除身分驗證環節
- 資安稽核必定被標記為高風險

### 3. 三重ALL權限風險

**檢查項目**
```bash
sudo grep -r "ALL.*ALL.*ALL" /etc/sudoers /etc/sudoers.d/
```

**風險說明**
- **設置範例**: `user ALL=(ALL:ALL) ALL`
- **風險**: 使用者可以以任何使用者身分執行任何指令
- **攻擊場景**:
  - `sudo -u postgres psql` - 以資料庫使用者身分操作
  - `sudo -u www-data cat /var/log/apache2/access.log` - 讀取web服務日誌
  - `sudo -u backup tar -czf /tmp/sensitive.tar.gz /etc/shadow` - 以備份使用者打包敏感檔案

**為什麼需要改善**
- 違反最小權限原則
- 可能繞過應用程式層級的安全控制
- 增加橫向移動攻擊面

### 4. 萬用字元設置風險

**檢查項目**
```bash
sudo grep -r "/bin/\*" /etc/sudoers /etc/sudoers.d/
sudo grep -r "/usr/bin/\*" /etc/sudoers /etc/sudoers.d/
```

**風險說明**
- **設置範例**: `user ALL=(ALL) /bin/*` 或 `user ALL=(ALL) /usr/bin/*`
- **風險**: 允許執行目錄下所有程式，包括危險的系統工具
- **攻擊場景**:
  - `sudo /bin/bash` - 直接獲得root shell
  - `sudo /usr/bin/vim /etc/passwd` - 編輯系統檔案
  - `sudo /bin/cp /bin/bash /tmp/rootshell; sudo /bin/chmod +s /tmp/rootshell` - 建立後門

**為什麼需要改善**
- 萬用字元設置等同於給予完整權限
- 無法預測和控制可執行的指令範圍
- 攻擊者可以找到意想不到的權限提升路徑

### 5. 文字編輯器權限風險

**檢查項目**
```bash
sudo grep -r -E "(vi|vim|nano|emacs|sed|awk)" /etc/sudoers /etc/sudoers.d/
```

**風險說明**
- **設置範例**: `user ALL=(ALL) /usr/bin/vim`
- **風險**: 文字編輯器可以被濫用來執行系統指令
- **攻擊場景**:
  ```bash
  # vim中執行shell指令
  sudo vim
  :!bash
  
  # nano中執行指令
  sudo nano
  Ctrl+T (然後可以執行指令)
  
  # 使用vim編輯任何系統檔案
  sudo vim /etc/passwd
  sudo vim /etc/sudoers
  ```

**為什麼需要改善**
- 編輯器通常有shell escape功能
- 可以修改關鍵系統配置檔案
- 難以審計實際執行了什麼操作

### 6. 系統檔案編輯權限風險

**檢查項目**
```bash
sudo grep -r "/etc/sudoers" /etc/sudoers /etc/sudoers.d/
sudo grep -r "/etc/passwd" /etc/sudoers /etc/sudoers.d/
```

**風險說明**
- **設置範例**: `user ALL=(ALL) /usr/bin/vim /etc/sudoers`
- **風險**: 可以修改sudo設置或使用者帳號資訊
- **攻擊場景**:
  ```bash
  # 修改sudoers給自己更多權限
  sudo vim /etc/sudoers
  # 加入: user ALL=(ALL) NOPASSWD:ALL
  
  # 修改passwd建立後門帳號
  sudo vim /etc/passwd
  # 加入: hacker:x:0:0:root:/root:/bin/bash
  ```

**為什麼需要改善**
- 可以自我提權，破壞權限控制機制
- 可以建立隱藏的管理員帳號
- 難以檢測和追蹤惡意修改

## 實際改善建議

### 1. 替代NOPASSWD的安全作法
```bash
# 壞的設置
user ALL=(ALL) NOPASSWD:ALL

# 好的設置 - 只允許特定指令
user ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart apache2
user ALL=(ALL) NOPASSWD: /usr/bin/systemctl status *
```

### 2. 使用精確的指令路徑
```bash
# 壞的設置
user ALL=(ALL) /bin/*

# 好的設置
user ALL=(ALL) /bin/systemctl, /usr/bin/docker
```

### 3. 使用指令別名管理複雜權限
```bash
# 在sudoers中定義
Cmnd_Alias WEB_CMDS = /usr/bin/systemctl restart apache2, \
                      /usr/bin/systemctl reload apache2, \
                      /usr/bin/systemctl status apache2

# 使用別名
webadmin ALL=(ALL) WEB_CMDS
```

### 4. 實施權限分離
```bash
# 不同角色給予不同權限
dbadmin  ALL=(postgres) NOPASSWD: /usr/bin/psql
webadmin ALL=(www-data) NOPASSWD: /usr/sbin/apache2ctl
backup   ALL=(ALL) NOPASSWD: /usr/bin/rsync, /bin/tar
```

這些危險設置的根本問題是**違反了最小權限原則**和**深度防禦策略**。每個設置看似方便，但都為攻擊者提供了權限提升的捷徑。在高安全環境中，寧可增加管理複雜度，也要確保每個權限都是必要且受控的。
