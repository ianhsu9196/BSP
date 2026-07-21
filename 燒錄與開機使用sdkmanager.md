**NVIDIA SDK Manager  
Jetson Orin Nano 燒錄流程報告**

Host：AAEON BOXER-6650-PTH｜Ubuntu 22.04 LTS｜x86_64

| **報告主題** | 使用 NVIDIA SDK Manager 重新燒錄 Jetson Orin Nano |
|--------------|---------------------------------------------------|
| **執行平台** | AAEON BOXER-6650-PTH，Ubuntu 22.04 LTS，8GB RAM   |
| **目標裝置** | NVIDIA Jetson Orin Nano Developer Kit             |
| **執行結果** | SDK Manager 安裝與燒錄成功                        |

# 一、目的與背景

**本次作業目的為：**將原本不適合 SDK Manager 的 Ubuntu 26.04 主機重灌為
Ubuntu 22.04，完成 SDK Manager 安裝，並透過 Recovery Mode 將 Jetson Orin
Nano 成功燒錄。

原先在 Ubuntu 26.04 上雖可安裝 SDK Manager 套件，但 GUI 啟動時發生
Chromium／GPU process
crash，且系統版本不屬於此流程的穩定支援環境，因此改採重灌 Ubuntu 22.04
的方式處理。

# 二、環境與設備

| **項目**   | **規格／型號**                 | **用途**                           |
|------------|--------------------------------|------------------------------------|
| Host 主機  | AAEON BOXER-6650-PTH           | 執行 Ubuntu 22.04 與 SDK Manager   |
| CPU        | Intel Core Ultra X7 358H       | x86_64 架構                        |
| 記憶體     | 8GB RAM                        | 額外建立 Swap 以降低記憶體不足風險 |
| 內部磁碟   | Phison 512GB NVMe              | 安裝 Ubuntu 22.04                  |
| 安裝隨身碟 | Transcend JetFlash 64GB        | 製作 Ubuntu 22.04 開機媒體         |
| Target     | Jetson Orin Nano Developer Kit | SDK Manager 燒錄目標               |
| 連接方式   | USB-C 資料線 + Jetson DC 電源  | Recovery Mode 與燒錄通訊           |

# 三、Ubuntu 22.04 重灌流程

## 3.1 下載 ISO

下載 Ubuntu 22.04.5 Desktop AMD64 映像檔：

ubuntu-22.04.5-desktop-amd64.iso

下載後確認檔案大小約 4.5GB：

ls -lh ~/Downloads/ubuntu-22.04.5-desktop-amd64.iso

## 3.2 確認磁碟裝置

使用 lsblk 辨識內部 NVMe 與 USB 隨身碟：

lsblk -o NAME,SIZE,TYPE,MOUNTPOINTS,MODEL

辨識結果：

| **裝置**     | **容量／型號**         | **判斷**         |
|--------------|------------------------|------------------|
| /dev/sda     | 58.8GB，Transcend 64GB | USB 安裝隨身碟   |
| /dev/nvme0n1 | 476.9GB，Phison NVMe   | BOXER 內部系統碟 |

## 3.3 製作 Ubuntu 開機隨身碟

先卸載 USB 分割區，再將 ISO 寫入整個 USB 裝置：

sudo umount /dev/sda1  
  
sudo dd \\  
if=/home/aaeon/Downloads/ubuntu-22.04.5-desktop-amd64.iso \\  
of=/dev/sda \\  
bs=4M \\  
status=progress \\  
conv=fsync  
  
sync

注意：輸出裝置為 /dev/sda，而不是 /dev/sda1；若誤選
/dev/nvme0n1，將直接覆寫主機系統碟。

## 3.4 從 USB 啟動

重新開機後進入 AMI Aptio BIOS，在 Boot 頁面將下列裝置設為第一順位：

USB Device: UEFI: JetFlashTranscend 64GB 1100, Partition 2

進入 GNU GRUB 後選擇：

Try or Install Ubuntu

## 3.5 安裝 Ubuntu 22.04

1.  選擇 Install Ubuntu。

2.  鍵盤配置選 English (US)。

3.  安裝類型選 Normal installation。

4.  磁碟設定選 Erase disk and install Ubuntu。

5.  確認目標為 Phison 512GB NVMe，而不是 Transcend USB。

6.  建立使用者帳號並完成安裝。

7.  重新開機時拔除 USB 安裝隨身碟。

## 3.6 驗證系統

lsb_release -a  
uname -m

確認結果應包含 Release 22.04 與 x86_64。

# 四、建立虛擬記憶體（Swap）

由於 BOXER 僅有 8GB RAM，為降低 SDK Manager
下載、解壓縮及燒錄過程中記憶體不足的風險，建立 32GB swapfile。

sudo swapoff -a  
sudo fallocate -l 32G /swapfile  
sudo chmod 600 /swapfile  
sudo mkswap /swapfile  
sudo swapon /swapfile

設定開機自動掛載：

echo '/swapfile none swap sw 0 0' \| sudo tee -a /etc/fstab

確認：

free -h  
swapon --show

# 五、安裝 NVIDIA SDK Manager

在 Ubuntu 22.04 上安裝 NVIDIA CUDA repository keyring 與 SDK Manager：

cd ~/Downloads  
  
wget
https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb  
  
sudo dpkg -i cuda-keyring_1.1-1_all.deb  
sudo apt update  
sudo apt install -y sdkmanager

確認套件：

dpkg -l \| grep sdkmanager

實際安裝版本：

sdkmanager 2.4.1-13536 amd64

啟動 SDK Manager：

sdkmanager

# 六、Jetson Orin Nano 進入 Recovery Mode

燒錄前，Jetson 必須進入 Force Recovery Mode，讓 Host 透過 USB-C
辨識並寫入系統。

1\. Jetson 完全關機並拔除 DC 電源。

2\. USB-C 資料線連接 BOXER Host 與 Jetson。

3\. 短接 Jetson J14 排針的 FC REC 與 GND。

4\. 保持短接狀態後插入 Jetson DC 電源。

5\. 等待約 2 至 3 秒，再移除短接線。

6\. Recovery Mode 下 Jetson 螢幕可能黑畫面，此為正常現象。

在 BOXER 端確認：

lsusb \| grep 0955

若看到 NVIDIA Vendor ID 0955，即代表 Recovery Mode 成功。

# 七、SDK Manager 燒錄設定

## 7.1 登入與目標設定

啟動 SDK Manager 後登入 NVIDIA Developer 帳號，於 STEP 01 選擇 Jetson
平台與 Orin Nano 目標硬體。

| **設定項目**     | **選擇內容**                             |
|------------------|------------------------------------------|
| Product Category | Jetson                                   |
| Target Hardware  | Jetson Orin Nano Developer Kit           |
| SDK Version      | 依公司／Mentor 指定的 JetPack 版本       |
| Host Machine     | 依需求取消或保留；本次重點為 Target 燒錄 |

## 7.2 元件選擇

在 STEP 02 保留 Jetson OS，若需完整開發環境則一併安裝 Jetson SDK
Components。

## 7.3 Flash 設定

| **項目**          | **設定**                             |
|-------------------|--------------------------------------|
| Recovery Mode     | Manual Setup                         |
| OEM Configuration | Pre-Config 或 Runtime                |
| 儲存裝置          | 依 Jetson 實際硬體選 NVMe 或 microSD |
| 連線              | USB-C 資料線持續連接                 |

按下 Flash 後，SDK Manager 會依序下載、解壓縮、寫入 BSP／Jetson
OS，並視選項安裝 CUDA、TensorRT 等 Target 元件。

# 八、燒錄期間注意事項

- 不可拔除 Jetson DC 電源。

- 不可拔除 USB-C 資料線。

- 不可讓 Host 進入休眠。

- 不可關閉 SDK Manager。

- 不可在燒錄過程中按下 Reset 或更換儲存裝置。

- 若記憶體不足，確認 Swap 已正常啟用。

# 九、完成後驗證

燒錄完成後 Jetson 會重新啟動。若選 Runtime，需在 Jetson 上完成 Ubuntu
初始設定；若選 Pre-Config，則可使用預先設定的帳號登入。

建議於 Jetson 端執行：

cat /etc/nv_tegra_release  
lsb_release -a  
apt-cache show nvidia-jetpack \| grep Version  
nvcc --version  
lsblk  
tegrastats

驗證重點包括：L4T／JetPack 版本、Ubuntu
版本、CUDA、儲存裝置掛載位置與系統運作狀態。

# 十、問題紀錄與排除

| **問題**                           | **原因**                                                         | **處理方式**                         |
|------------------------------------|------------------------------------------------------------------|--------------------------------------|
| sdkmanager\_\*\_amd64.deb 無法安裝 | Downloads 中沒有對應安裝檔，萬用字元未展開                       | 改用 NVIDIA repository 安裝          |
| Ubuntu 26.04 GUI 崩潰              | SDK Manager 內部 Chromium GPU process 無法啟動，且環境相容性不佳 | 重灌 Ubuntu 22.04                    |
| SDK Manager 顯示 already running   | 背景殘留程序                                                     | pkill -9 -f sdkmanager 後重新啟動    |
| 主機重新開機被阻擋                 | GNOME session inhibitor                                          | sudo systemctl reboot -i             |
| USB 無法辨識                       | 可能未進 Recovery Mode、線材僅支援充電或腳位接錯                 | 重新短接 FC REC/GND，並用 lsusb 驗證 |
| 8GB RAM 可能不足                   | 下載、解壓縮與套件安裝耗用記憶體                                 | 建立 32GB Swap                       |

# 十一、結論

本次流程完成了 Host 作業系統重建、USB 安裝媒體製作、Ubuntu 22.04
安裝、32GB Swap 建立、SDK Manager 2.4.1 安裝、Jetson Recovery Mode
連線與正式燒錄。最終 SDK Manager 燒錄成功，證明 Host
版本、硬體連線、Recovery Mode 與 SDK Manager 設定皆正確。

此流程也建立了可重複使用的 Jetson BSP／JetPack 燒錄 SOP，未來若需更換
JetPack 版本、重新初始化 NVMe 或進行 BSP
開發，可沿用本報告的檢查順序與故障排除方式。

# 附錄：快速 SOP

1\. 確認 Host 為 Ubuntu 22.04 x86_64。

2\. 確認磁碟空間與 Swap。

3\. 安裝並啟動 SDK Manager。

4\. USB-C 連接 Host 與 Jetson，Jetson 另接 DC 電源。

5\. 短接 FC REC 與 GND，使 Jetson 進 Recovery Mode。

6\. Host 執行 lsusb \| grep 0955。

7\. SDK Manager 選 Jetson Orin Nano 與指定 JetPack。

8\. 選 Manual Setup、儲存裝置與 OEM 設定。

9\. 按 Flash 並保持電源與 USB 連線。

10\. 燒錄完成後驗證 L4T、JetPack、CUDA 與儲存裝置。
