# CLAUDE.md

拾音 Shiyin 是 macOS 原生的會議紀錄 app（SwiftUI，macOS 26）。目前是 3.5.0（第三版加上英文版、開源準備、轉錄準確度、日文、更多整理工具、穩定性修正；3.5 加上草稿紀錄、聲紋、猜說話者、對會議提問、偵測會議）：

- 錄音：AVAudioEngine 收麥克風、Core Audio process tap 收電腦聲音，寫成雙聲道 WAV（`Sources/Shiyin/Capture/`）
- 即時逐字稿：系統的 SpeechAnalyzer（zh_TW、en_US 或 ja_JP，依會議語言）
- 正式轉錄：whisper.cpp（官方 XCFramework）+ Breeze ASR 25 GGML q8_0，可以改用 macOS 內建（SpeechAnalyzer 讀檔），模型還沒下載好時也先用內建；日文會議用 Whisper large-v3-turbo q5_0；whisper 從檔案依序讀、每 10 分鐘一段（design §21.16）；說話者辨識：FluidAudio（`Sources/Shiyin/Transcribe/`），聲紋比對在 `Store/Voiceprints.swift`
- 整理：claude CLI、codex CLI、Anthropic API、OpenAI API、Gemini API、OpenAI 相容 API，共同介面 `LLMBackend`（`Sources/Shiyin/Notes/`，各家在 `Notes/Backends/`，design §19）
- 資料庫：GRDB（SQLite、FTS5 全文搜尋）
- 多語言：介面繁中、英文（跟著 macOS 語言）；每場會議另外有會議語言（中文／英文／日文），決定轉錄、整理、匯出用哪個語言（design §16、§18）；錄音開始後用 Whisper base 自動偵測（§18.8）
- 發布：自簽憑證或 ad-hoc 簽章的 DMG（沒有 Apple 開發者帳號）

不需要 Python、Homebrew、ffmpeg。選型的理由在 design.md 的決策紀錄（D#）。已經刪掉的舊文件（第二版的規格 `docs/archive/v2/`、選型實驗 `docs/spikes/`、3.4.0 以前完整的 `docs/tasks.md`）用 `git show b621ed6:<路徑>` 看。

## 開發流程：SDD（規格驅動）

本專案先寫規格再寫程式，規格放在 `docs/`：

| 文件 | 內容 |
|---|---|
| `docs/requirements.md` | 要做什麼：需求編號 R#、N#，第 8 節是驗收結果 |
| `docs/design.md` | 怎麼做：架構、資料模型、提示詞、介面、決策紀錄 D# |
| `docs/tasks.md` | 還沒完成的任務，每項都有驗證方式；做完就刪掉 |

開發時的規則：
- 開工前先讀這三份文件，從 `tasks.md` 的任務接著做。
- 每完成一項任務，照它的「驗證」實際跑過一次，從 `tasks.md` 刪掉，跑了什麼、結果如何寫在 commit 訊息。`[~]` 表示程式完成，但有部分驗證要等使用者（權限、真實錄音、API key），只寫還差什麼。
- docs 只留還用得到的：做完的任務、用不到的文件直接刪掉，不另外留紀錄檔（都在 git 歷史裡）。
- 程式要偏離 `design.md` 時，先改 `design.md`（重要的取捨寫進決策紀錄 D#），再改程式，commit 訊息寫原因。
- 有新需求時，先寫進 `requirements.md`，再改設計、加任務。不要直接寫程式。

## 指令

```bash
scripts/build-app.sh             # 建置 release 版到 build/Shiyin.app（實際在 ~/Library/Caches/ShiyinBuild/app）
scripts/build-app.sh --install   # 建置並安裝到 ~/Applications
scripts/build-app.sh --debug     # DEBUG 版：含驗證用控制通道（指令見 Sources/Shiyin/Debug/DebugControl.swift）
scripts/make-dmg.sh              # release 建置並產生 DMG（dist/Shiyin-<版本>.dmg，含中英文安裝說明 PDF）
scripts/dev/l10n.py check        # 抽出介面字串合併進翻譯表，列出缺英文的條目和沒查表的中文（有問題時 exit 1）
scripts/dev/make-third-party-licenses.sh   # 更新套件版本後重跑：產生 Resources/ThirdPartyLicenses.txt（「關於拾音」和 DMG 附的授權條款）
scripts/dev/l10n.py apply <翻譯.json>   # 寫入英文翻譯：{"中文鍵": "English"}，複數用 {"one": …, "other": …}
scripts/make-signing-cert.sh     # 建立自簽憑證；加 --trust 信任它（要使用者輸入登入密碼）
scripts/export-signing-cert.sh --to-github <owner/repo>   # 把自簽憑證設成 GitHub Actions 的 secrets（私鑰會離開電腦，要使用者同意）
# CI：.github/workflows/release.yml，推 v* tag 就建置並發布 Release；手動執行只產出 artifact
node site/build.mjs              # 產生官網到 site/dist（讀 GitHub Releases，不用裝套件）；node site/serve.mjs 本機預覽 http://localhost:4321/
site/make-images.sh              # docs/images 的截圖或圖示換過後跑：產生官網用的 AVIF、favicon、分享預覽圖，連同 site/public 一起 commit
# 官網 CI：.github/workflows/site.yml，推到 main 改到 site/ 或 docs/images/、發布新版之後，部署到 Cloudflare Pages
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

## 介面文字（3.1 起）

- 程式裡照樣寫中文；SwiftUI 的字面值會自動查 `Resources/Localizable.xcstrings`，其他地方（錯誤、提示、通知、狀態文字）要包 `String(localized:)`。鍵就是中文原文。
- 改過介面文字一定要跑 `scripts/dev/l10n.py check`，再把新條目的英文用 `apply` 寫進去。不會顯示在介面上的中文行尾加 `// l10n-skip`。
- 同一句中文在不同地方要不同的英文時，用 `MeetingLanguage.ui(zh:en:)`；依會議語言（不是介面語言）的文字用 `meeting.meetingLanguage.text(zh:en:)`，例如匯出、預設說話者名稱。
- 中文提示詞、中文匯出格式要跟 3.0 一字不差，改英文版時不要動到中文那一份。

## 發布新版的說明文字

官網的版本說明直接讀 GitHub Release（`docs/website.md` §7），格式不對時官網建置失敗、不會更新。tag 的說明文字（`git tag -a vX.Y.Z -F 說明.txt`，會變成 Release 的內容）一律寫成：

```
拾音 3.3.0：日文會議、自動偵測會議語言
- 一行一條，用「- 」開頭
- 寫使用者看得到的改變，跟 app 用同樣的詞

Shiyin 3.3.0: Japanese meetings and automatic language detection
- One item per line, starting with "- "
- What users will notice, in the app's own words
```

中文一段、空一行、英文一段；標題的版本號要跟 tag 一致，「：副標」可以省略。發布前可以先用 `node site/build.mjs --releases <檔案>` 拿一份跟 GitHub API 同樣格式的 JSON 試建置。

## 驗證時要注意

- 使用者可能正在用電腦：
  - 開 app 一律用 `open -g`（不搶前景）。
  - **絕對不要送按鍵**（System Events 的 `keystroke`、`key code`）：按鍵會送到最前面的 app，曾經把 ⌘W 送進使用者的 Chrome。關視窗用 `shiyinctl closewindow <id>`，其他操作也優先用 shiyinctl。
  - 不要點會跳出選單或系統面板的按鈕（分享、存檔對話框），會出現在使用者的畫面上。
  - 不要用 AppleScript 叫 app 結束：有工作在跑時會跳出確認對話框。建置腳本會自己處理結束。
- 截圖：
  - `shiyinctl snapshot` 用 ImageRenderer，不會跳出視窗，但畫不出 AppKit 元件（TextField、Picker、ScrollView 會變成黃色佔位框），適合看排版、顏色。
  - 要看真正的畫面，用 `screencapture -x -o -l <視窗編號>` 只擷取拾音的視窗（視窗編號用 CGWindowListCopyWindowInfo 依 PID 找）。螢幕鎖定時拍不到。
- 從這個終端機（Claude Code）用 `open` 開 app，app 會繼承 Claude Code session 的環境變數，子程序的 `claude` 會沿用這個 session 的登入。要模擬使用者的電腦（例如沒登入）時，用 `env -i HOME=… PATH=/usr/bin:/bin SHIYIN_HOME=… <app>/Contents/MacOS/Shiyin &` 啟動。
- 英文介面：直接執行 `build/Shiyin.app/Contents/MacOS/Shiyin -AppleLanguages "(en)"`（要用示範資料時前面加 `SHIYIN_HOME=…`）；中文介面就是系統語言。改會議語言用 `shiyinctl language <id> zh|en`。
- 還沒完成首次使用引導的話，每次啟動都會打開引導視窗，測試時記得用 `shiyinctl closewindow onboarding` 關掉。
- 編輯器、對話框的按鈕（完成、取消、儲存）要在真的 app 上用 AX 按過，不能只用 shiyinctl 直接存資料：3.2.0 的會議紀錄編輯器一按「完成」就閃退，就是因為當時只用 `shiyinctl editnotes` 驗證。
- 拾音的視窗在別的桌面（Space）時，AX 列不出它的視窗。要先用 System Events `set frontmost`，等 AX 看得到視窗再操作；使用者在用電腦時會被切回去，先跟使用者說一聲。
- 測試資料會寫進使用者真正的資料庫。測完用 `shiyinctl purge <id>` 刪掉自己建立的會議，改過的設定用 `defaults delete com.weiwei.shiyin <key>` 還原。
- 要用使用者的真實資料測（例如 migration）時，另外建一個 `SHIYIN_HOME`：資料庫用 `sqlite3 <正式資料庫> ".backup <複本>"` 複製（app 開著也安全）；`audio/recording/`（錄音中的暫存檔）不要複製，複本裡錄音中的會議（`source='recording'`、`audio_file=''`）刪掉；DEBUG 版加 `-llmProvider none -autoSummarize NO -correctTranscript NO` 啟動。2026-10-19 曾經複製到使用者正在錄的會議，DEBUG 版把它當成當掉的錄音復原、轉錄，還用使用者的 Claude 整理了一次。
- 設定（UserDefaults `com.weiwei.shiyin`）跟使用者正式安裝的 app 共用，換 `SHIYIN_HOME` 也一樣。測完要 `defaults read com.weiwei.shiyin` 檢查有沒有留下測試改的值（曾經把 `defaultMeetingLanguage` 留成 en，使用者升級後新會議會預設英文）。
- 說話者辨識不能用 `say` 合成的聲音測（會判成同一人），要用真人錄音，例如 AMI 會議語料（`~/FluidAudioDatasets`）。
- 說話者分不出人（3.5.1）：`shiyinctl voiceprint-eval` 的設定每一項可以加 `threshold`、`minSpeakers`、`numSpeakers`（字串），同一段錄音換設定比較分出幾個人；`shiyinctl rediarize <id> <人數|auto>` 只重新辨識說話者。
- 轉錄的記憶體、分段（3.5）：`shiyinctl transcribe-eval` 的輸出有各階段的記憶體峰值（`memory`）和分段位置（`seams`）；設定 JSON 加 `"chunkSeconds": 0` 不分段。量記憶體要每次重開 app（同一個 app 連續跑，前一次留下的記憶體會算進去）。
- 模擬錄音（`shiyinctl record-sim <檔案> <倍速> <1=雙聲道>`）的雙聲道是同一個讀檔迴圈同時送兩個聲道。不要改成兩個獨立的音源：高倍速時兩邊速度不同，寫檔時會補靜音，檔案變長，那是測試工具的假象。
- 只想確認能不能編譯時，用 `swift build -c debug --arch arm64 --disable-keychain --scratch-path ~/Library/Caches/ShiyinBuild`，不會動到正在跑的 app；`build-app.sh` 會先結束 app（有長時間測試在跑時不要執行）。
- 測 app 內更新：`scripts/dev/make-update-fixtures.sh <資料夾>` 產生正常、SHA 不符、簽章不符、同版本四種更新檔，在那個資料夾跑 `python3 -m http.server 18765 --bind 127.0.0.1`，再用 `shiyinctl setupdateurl http://127.0.0.1:18765/good.json`、`checkupdate`、`installupdate`。
- 全域快捷鍵不能用送按鍵測（見上面），用 `shiyinctl hotkey` 模擬按下。
- 日文（3.3）：`scripts/dev/make-eval-data-ja.py` 產生日文評估資料（合成會議、國會聽證、FLEURS）。`shiyinctl import lang=ja <檔案>` 用日文匯入（不用改共用的預設語言）；`shiyinctl detect-eval <音檔> <開始秒數> <輸出.json>` 用 app 的語言偵測跑一段；`shiyinctl recordlang <zh|en|ja>` 錄音中換語言（等於按「改回」）；`shiyinctl model start|pause|delete japanese|languageDetection` 管其他模型。
- 設定頁是 `Form`，`shiyinctl snapshot` 畫不出來（一片空白）。 要測按鍵或右鍵時，用 `CGEvent.postToPid(拾音的 pid)` 只送給拾音（不經過最前面的 app）；輸入框和清單的焦點用 AX 的 `AXFocused` 設，不要靠點擊。座標點擊（`click at`）會點到當下最上層的視窗，點之前一定要確認拾音在最前面。測模型下載出錯用 `shiyinctl modelurl <模型> <網址>` 指到本機伺服器。要看真的畫面：把 app 拉到前景（System Events 的 `set frontmost`），用 AX 點 app 自己的選單和按鈕（設定的分頁是 `button "錄音" of toolbar 1`），再 `screencapture -l <視窗編號>`；螢幕上鎖時截不到。選單裡的選項用 AX 點會寫進共用的 UserDefaults，測完要 `defaults delete`。
- 示範資料夾的 `models` 是指到正式資料夾（`~/Library/Application Support/Shiyin/models`）的連結：在示範資料夾下載的模型會存進正式資料夾；設定頁的「模型」大小在示範資料夾會顯示 0（不跟著連結算）。
- `AVAudioFile` 關檔後 `length` 會變 0，不能用它判斷錄了多久（曾經因此刪掉整場錄音）。
- zsh 內建的 `log` 會蓋掉系統指令，要用 `/usr/bin/log`；zsh 不會依 IFS 切字串；`rm -f *.wav` 沒有符合的檔案時會報錯。
- bash 腳本裡，變數後面緊接全形字時要寫成 `${VAR}`，否則全形字的位元組會被吃進變數名稱。
- 用腳本做大段替換後，檢查檔案的行數和結尾：曾經因為替換腳本漏接後半段，把整個檔案截斷。
- 整理工具（3.4）：測 API 用假伺服器 `scripts/dev/fake-llm-server.py` 加上 `shiyinctl llmurl`；key、整理工具、模型用只放記憶體的 `llmkey`、`llmprovider`。DEBUG 版（3.4.1 起）預設不讀真的鑰匙圈，要測時加 `-debugRealKeychain YES`（會讀使用者存的 key，可能跳出確認視窗）。**不要用 `setkey` 和 `defaults write` 測**：鑰匙圈項目綁定建立它的 app 簽章，ad-hoc 重建後或使用者自己的拾音去讀都會跳出系統的存取確認；整理工具設定跟使用者的拾音共用。
- 使用者的拾音可能也開著（`~/Desktop/Shiyin.app`），跟測試用的 DEBUG 版共用設定和視窗狀態。DEBUG 版用 `open -g -n … --args -ApplePersistenceIgnoreState YES` 啟動，否則可能照上次被 `pkill` 時的狀態還原成一個視窗都沒有；設定值可以用啟動參數帶（`-llmProvider gemini`），只在這次執行有效。
- 開關類設定的啟動參數（`-autoSummarize NO`、`-correctTranscript NO`）在 3.4.1 以前的建置沒有作用：讀進來是字串 "NO"，`as? Bool` 轉不成功就用預設值（開）。3.5 起 `AppSettings` 改用 `bool(forKey:)`。測舊版時不要以為加了參數就不會整理；`-llmProvider none` 一直有效。
- 要跟舊版比對（例如匯出格式）時，用 `git worktree` 建置舊版的 DEBUG 版。3.4.0 以前的 DEBUG 版啟動時就讀真的鑰匙圈，會在使用者畫面上一直跳出存取確認、app 卡住（2026-10-19），建置前先把 worktree 裡的 `Keychain.get` 改成直接 `return nil`；啟動後確認 log 有「debug control installed」。`shiyinctl` 會廣播給所有 DEBUG 版，同時只能開一個，3.4.0 的 `quit` 不會結束 app，要 `kill` 並確認程序真的不在了（曾經因為舊版沒結束，兩個版本同時寫同一個匯出檔）。
- 螢幕上鎖時 AX 列不出視窗。筆記輸入框可以用 `shiyinctl notestype`、`notesvalue` 直接操作背後的 NSTextView。
- SwiftUI 的 `Menu` 在畫面上只畫標題的文字（標籤裡的背景、膠囊標記會被拿掉），`shiyinctl snapshot` 卻畫得出來；HStack 裡的 Text 加 `.fixedSize(horizontal: false, vertical: true)`，量最小高度時會用很窄的寬度，整個視窗的內容被往上推（3.5 驗收時都是在真的畫面上才發現）。ScrollView、popover 裡的 Text 才可以用 fixedSize。
- 用 AX 操作的小技巧：設定頁要捲動時，設垂直捲軸的 `AXValue`（0～1）；選單的「設定⋯」用的是 ⋯（U+22EF）；只想按某個視窗裡的按鈕時，從那個視窗（`AXWindows` 裡標題相符的）往下找，不然會按到別的視窗裡同名的按鈕。
- 設定頁、引導在背景打不開又不想搶前景時，用 `shiyinctl preview <settings|onboarding [步驟]>` 開在桌面層（被所有視窗蓋住），再 `screencapture -l` 截。設定頁打開時可能把欄位寫回共用設定，測完 `defaults read` 比對。
