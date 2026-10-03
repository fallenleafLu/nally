# Nally

Open-Source telnet/ssh BBS client.

## Announcement

支援ssh連線，以ptt為例`ssh://bbs@ptt.cc`即可。 注意，要有**bbs@**。

- 第一次用 ssh 連到某個站時，會自動記住對方的 host key，不會跳出確認提示；之後如果 host key 改變，連線會被拒絕。
- Nally 跑在 App Sandbox 裡，ssh 使用的 known_hosts 是 `~/Library/Containers/com.rayer.nally/Data/.ssh/known_hosts`，而不是你自己的 `~/.ssh/known_hosts`。如果站台更換了 host key，請從這個檔案刪掉對應的那一行。

## History

### 3.0.0

Release Date : 2026.09.26

這個版本主要是讓 Nally 能在 Xcode 27 與 macOS 27 上重新編譯、執行，並修正 sandbox 底下 ssh 連不上的問題。

* 最低系統需求提高到 **macOS 12.0**。Xcode 27 已經不支援更舊的 Deployment Target。
* 修正 ssh 在 sandbox 底下讀不到 known_hosts，導致一律出現 `Host key verification failed` 而連不上的問題。
* 修正 ssh 連線時終端機沒有進入 raw mode 的問題：之前登入欄位會出現 `^[[15;3R` 之類的亂碼而無法登入、按鍵要按 Return 才送出，非 bbs 帳號登入時密碼也會被顯示在畫面上。
* 對方斷線以後，畫面上的內容會保留，不再整片清空，可以看到站方最後送出的訊息。
* 修正在新版 macOS 上開新分頁時什麼事都沒發生、連線不會建立的問題（分頁列的 tracking rect 例外）。
* 修正連線位址查詢失敗時，在背景執行緒更新畫面的問題。
