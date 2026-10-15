# CLAUDE.md

拾音 Shiyin 是 macOS 原生的會議紀錄 app（SwiftUI，macOS 26）。目前是第三版（3.0.0）：

- 錄音：AVAudioEngine 收麥克風、Core Audio process tap 收電腦聲音，寫成雙聲道 WAV（`Sources/Shiyin/Capture/`）
- 即時逐字稿：系統的 SpeechAnalyzer（zh_TW）
- 正式轉錄：whisper.cpp（官方 XCFramework）+ Breeze ASR 25 GGML q8_0，可以改用 macOS 內建（SpeechAnalyzer 讀檔）；說話者辨識：FluidAudio（`Sources/Shiyin/Transcribe/`）
- 整理：claude CLI、codex CLI、Anthropic API、OpenAI API（`Sources/Shiyin/Notes/`）
- 資料庫：GRDB（SQLite、FTS5 全文搜尋）
- 發布：自簽憑證或 ad-hoc 簽章的 DMG（沒有 Apple 開發者帳號）

不需要 Python、Homebrew、ffmpeg。第二版（Python + mlx-whisper）的規格在 `docs/archive/v2/`，選型依據在 `docs/spikes/asr-engines.md`。

## 開發流程：SDD（規格驅動）

本專案先寫規格再寫程式，規格放在 `docs/`：

| 文件 | 內容 |
|---|---|
| `docs/requirements.md` | 要做什麼：需求編號 R#、N#，第 8 節是驗收結果 |
| `docs/design.md` | 怎麼做：架構、資料模型、提示詞、介面、決策紀錄 D# |
| `docs/tasks.md` | 實作順序 T#，每項都有驗證方式和結果，最後是變更紀錄 |

開發時的規則：
- 開工前先讀這三份文件，從 `tasks.md` 還沒打勾的任務接著做。
- 每完成一項任務，照它的「驗證」實際跑過一次才打勾，並寫下跑了什麼、結果如何。`[~]` 表示程式完成，但有部分驗證要等使用者（權限、真實錄音、API key）。
- 程式要偏離 `design.md` 時，先改 `design.md`，再到 `tasks.md` 的「變更紀錄」記一筆，最後才改程式。
- 有新需求時，先寫進 `requirements.md`，再改設計、加任務。不要直接寫程式。

## 指令

```bash
scripts/build-app.sh             # 建置 release 版到 build/Shiyin.app（實際在 ~/Library/Caches/ShiyinBuild/app）
scripts/build-app.sh --install   # 建置並安裝到 ~/Applications
scripts/build-app.sh --debug     # DEBUG 版：含驗證用控制通道（沿用第二版，見 docs/archive/v2/design.md §13）
scripts/make-dmg.sh              # release 建置並產生 DMG（dist/Shiyin-<版本>.dmg，含安裝說明 PDF）
scripts/make-signing-cert.sh     # 建立自簽憑證；加 --trust 信任它（要使用者輸入登入密碼）
scripts/export-signing-cert.sh --to-github <owner/repo>   # 把自簽憑證設成 GitHub Actions 的 secrets（私鑰會離開電腦，要使用者同意）
# CI：.github/workflows/release.yml，推 v* tag 就建置並發布 Release；手動執行只產出 artifact
swiftc scripts/dev/shiyinctl.swift -o ~/Library/Caches/ShiyinBuild/shiyinctl
~/Library/Caches/ShiyinBuild/shiyinctl state /tmp/s.json      # 讀狀態；其他指令見 Sources/Shiyin/Debug/DebugControl.swift
open -g build/Shiyin.app
/usr/bin/log stream --predicate 'subsystem == "com.weiwei.shiyin"'   # 看 log
sqlite3 ~/Library/Application\ Support/Shiyin/shiyin.sqlite # 看資料
```

建置暫存放在 `~/Library/Caches/ShiyinBuild`，專案裡的 `build/`、`dist/` 是指過去的捷徑（桌面有 iCloud 同步，不要把大檔案放進專案）。SwiftPM 要加 `--disable-keychain`，腳本已經處理。

## 重要限制

- 錄音、逐字稿、資料庫、模型一律放在 `SHIYIN_HOME`（預設 `~/Library/Application Support/Shiyin`），不要放進專案資料夾。
- API key 只能存在鑰匙圈（service `com.weiwei.shiyin.apikey`），不能寫進檔案、資料庫或 log。
- 呼叫外部程式一律用 `Process` 搭配參數陣列，不要組 shell 字串。
- log 不寫逐字稿和會議紀錄的內容。
- 從 Finder 開的 app 拿不到使用者 shell 的 PATH。要執行 claude、codex 時，用固定路徑，或在子程序的環境變數裡補上 PATH。
- 回覆和文件一律用繁體中文（台灣用語）。

## 驗證時要注意

- 使用者可能正在用電腦：
  - 開 app 一律用 `open -g`（不搶前景）。
  - **絕對不要送按鍵**（System Events 的 `keystroke`、`key code`）：按鍵會送到最前面的 app，曾經把 ⌘W 送進使用者的 Chrome。關視窗用 `shiyinctl closewindow <id>`，其他操作也優先用 shiyinctl。
  - 不要點會跳出選單或系統面板的按鈕（分享、存檔對話框），會出現在使用者的畫面上。
  - 不要用 AppleScript 叫 app 結束：有工作在跑時會跳出確認對話框。建置腳本會自己處理結束。
- 截圖：
  - `shiyinctl snapshot` 用 ImageRenderer，不會跳出視窗，但畫不出 AppKit 元件（TextField、Picker、ScrollView 會變成黃色佔位框），適合看排版、顏色。
  - 要看真正的畫面，用 `screencapture -x -o -l <視窗編號>` 只擷取拾音的視窗（視窗編號用 CGWindowListCopyWindowInfo 依 PID 找）。螢幕鎖定時拍不到。
- 還沒完成首次使用引導的話，每次啟動都會打開引導視窗，測試時記得用 `shiyinctl closewindow onboarding` 關掉。
- 測試資料會寫進使用者真正的資料庫。測完用 `shiyinctl purge <id>` 刪掉自己建立的會議，改過的設定用 `defaults delete com.weiwei.shiyin <key>` 還原。
- 說話者辨識不能用 `say` 合成的聲音測（會判成同一人），要用真人錄音，例如 AMI 會議語料（`~/FluidAudioDatasets`）。
- 模擬錄音（`shiyinctl record-sim <檔案> <倍速> <1=雙聲道>`）的雙聲道是同一個讀檔迴圈同時送兩個聲道。不要改成兩個獨立的音源：高倍速時兩邊速度不同，寫檔時會補靜音，檔案變長，那是測試工具的假象。
- 只想確認能不能編譯時，用 `swift build -c debug --arch arm64 --disable-keychain --scratch-path ~/Library/Caches/ShiyinBuild`，不會動到正在跑的 app；`build-app.sh` 會先結束 app（有長時間測試在跑時不要執行）。
- 測 app 內更新：`scripts/dev/make-update-fixtures.sh <資料夾>` 產生正常、SHA 不符、簽章不符、同版本四種更新檔，在那個資料夾跑 `python3 -m http.server 18765 --bind 127.0.0.1`，再用 `shiyinctl setupdateurl http://127.0.0.1:18765/good.json`、`checkupdate`、`installupdate`。
- 全域快捷鍵不能用送按鍵測（見上面），用 `shiyinctl hotkey` 模擬按下。
- `AVAudioFile` 關檔後 `length` 會變 0，不能用它判斷錄了多久（曾經因此刪掉整場錄音）。
- zsh 內建的 `log` 會蓋掉系統指令，要用 `/usr/bin/log`；zsh 不會依 IFS 切字串；`rm -f *.wav` 沒有符合的檔案時會報錯。
- bash 腳本裡，變數後面緊接全形字時要寫成 `${VAR}`，否則全形字的位元組會被吃進變數名稱。
