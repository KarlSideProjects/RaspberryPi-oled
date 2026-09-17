# RaspberryPi-oled

> **授權：僅限非商業用途，歡迎研究、教學與交流。** 完整條款見 [LICENSE](LICENSE)，適用範圍與第三方例外見 [授權規範](LICENSING.md)。

讓沒有接螢幕的 Raspberry Pi，也能直接在機身旁查看 Wi-Fi IP、記憶體與溫度。這是一個 Python 搭配 SH1106 SPI OLED 的小型硬體實作，適合放在桌面或實驗設備旁，減少為了確認主機狀態而登入終端機的步驟。

目前可查看下方既有實機照片，並在相符的 Raspberry Pi 與 OLED 上執行 [app.py](app.py)。儲存庫另保留一份獨立的 MicroPython 顯示測試；兩者的執行環境不同。

<img src="content/spi-oled.jpg" width="560" alt="既有實機照片：OLED 顯示 IP、可用／活躍／空閒記憶體與溫度">

## 畫面實際顯示什麼

- `IP`：從 `wlan0` 取得的 IPv4 位址，方便後續連線至 Pi。
- `MEM`、`MEM act.`、`MEM free`：psutil 回報的可用、活躍與空閒記憶體，除以 1024² 換算為 MiB。
- `Temp`：從 `psutil.sensors_temperatures()` 取得的溫度值。
- `RPi4B8G`：程式寫死的畫面標籤，**不是**自動辨識板型的結果。

程式也讀取主機名稱、CPU 核心數與使用率，但目前沒有把它們畫到 OLED。這份作品是本機資訊顯示原型，未包含歷史紀錄、遠端儀表板或告警功能。

## 從系統資訊到 OLED

```text
Linux 網路資訊 + psutil → 整理文字行 → luma canvas → SPI → SH1106 OLED
```

[app.py](app.py) 使用 luma.oled 內建的 SH1106 裝置支援，`Draw_Oled()` 每次清除背景、畫出外框與文字。選用現成繪圖介面能把注意力放在資料與接線；代價是每輪重畫整個畫面，而且版面為固定座標。迴圈中的 `cpu_percent(1)` 會等待約一秒，再加上查詢與繪圖時間，因此不是精準定時的量測器。

## 在 Raspberry Pi 上操作

需要 Raspberry Pi、Raspberry Pi OS／原 Raspbian、Python 3，以及相容的 SH1106 SPI OLED。先斷電接線，確認模組電壓與腳位標示，再上電；圖中的 `P01-xx` 是**實體針腳**編號，GPIO 是另一套編號。

<img src="content/spi.jpg" width="680" alt="既有 SPI 接線表：3.3V 接17、GND接20、SCLK接23、MOSI接19、RST接22、DC接18、CS接24（實體針腳）">

1. 用 `sudo raspi-config` 啟用 SPI，依提示重新啟動；確認 `/dev/spidev0.0` 存在，執行帳號有 SPI／GPIO 存取權。
2. 在 Pi 上建立 Python 環境並安裝 [app.py](app.py) 所需套件：

   ```bash
   git clone https://github.com/KarlSideProjects/RaspberryPi-oled.git
   cd RaspberryPi-oled
   python3 -m venv .venv
   . .venv/bin/activate
   python -m pip install luma.oled psutil
   python app.py
   ```

   若系統沒有 venv，先安裝 `python3-venv`。套件沒有鎖定版本；GPIO 後端與作業系統／Pi 板型的相容性需在目標設備確認。

3. 預期 OLED 出現外框與上述欄位並持續更新；按 `Ctrl+C` 停止。程式預設 `spi(device=0, port=0)`。接線若不同，需同步調整 luma SPI 參數。

**判讀與排錯：** 網路介面不是 `wlan0` 時，要修改程式中的介面名稱；未取得 IPv4 時 IP 會空白。溫度迴圈目前取最後一組感測器的第一筆值，未指定 CPU 感測器；沒有感測器時顯示 `0`，不能當作實測零度。`virtual_memory().active` 也依賴 Linux 提供該欄位。

### 另一份範例：MicroPython

[app2.py](app2.py) 使用 `machine.Pin`、`machine.SPI` 與儲存庫內的 [sh1106.py](sh1106.py)，只在 128×64 顯示器寫出 `Testing 1`。它不能直接在 Raspberry Pi OS 的一般 Python 執行，也沒有收集系統指標。

若要試用，需把兩個檔案放到支援的 MicroPython 裝置，先依板子的 GPIO 定義修正 `SPI(1)` 與 `Pin(18)`／`Pin(22)`／`Pin(24)`。原始註解的 ESP8266 接線表與這些實際參數不一致，不應直接照抄接線。

## 可延伸的探究活動

這個小型作品可以用來練習把「讀到的數字」連回其定義：比較 free 與 available 記憶體的差異，觀察負載與溫度變化，或檢查取樣間隔是否足以描述短暫事件。實作上也能追蹤 Linux 資料、Python 字串、像素座標與 SPI 接線之間的關係。

這些是可進一步設計的學習活動；儲存庫沒有提供課堂實驗、學生樣本或學習成效資料。

## 證據、驗證與來源

- 既有 [接線圖](content/spi.jpg) 與 [實機照片](content/spi-oled.jpg) 提供作品外觀與畫面的證據，不代表所有板型都已通過測試。
- 2026-09-16 文件整理時逐一核對三份 Python 原始碼、圖片與操作路徑，並做 Python 語法解析；未連接實機重跑。儲存庫目前沒有依賴鎖檔、自動測試或 CI 工作流程。
- 初始程式提交 `cb53fd5` 的作者紀錄為 Jhih-Wei Jhan；其中 [sh1106.py](sh1106.py) 明列第三方作者 Radomir Dopieralski、Robert Hammelrath、Tim Weber 與 MIT 授權全文。這份驅動不是本專案的原創成果；Pi 主程式使用的則是 luma.oled 的驅動。
- 專案自有內容依 [非商用研究授權](LICENSE) 發行；`sh1106.py` 的 MIT 聲明仍獨立有效。歷史 MIT 徽章及來源未確認圖片的界線見 [LICENSING.md](LICENSING.md)，本次不撤回已有效取得的舊版權利。
