# Danh sách API Zalo mà client gọi

Tài liệu tra cứu đầy đủ cho đội phát triển: với mỗi endpoint ghi **host thật**, **điều kiện tiên quyết**, **thời điểm client gọi**, **tham số gửi**, **trường phản hồi được dùng**, **mã lỗi xử lý** và **vị trí mã**. Trích từ `src/` ngày 2026-11-02, cập nhật 2026-11-03 theo upstream v0.0.6 (sau đợt khắc phục kiểm toán). Mọi hành vi giao thức là bản tái hiện từ Zalo PC 26.10.11; đổi tham số phải có bằng chứng capture (`re-notes/`).

---

## 0. Nền tảng chung cho mọi request

### 0.1 Bảng host

Host được chọn theo thứ tự: hằng đã xác minh trong `src/endpoints.rs` (luôn thắng, `Domains::heal_from_verified`) → giá trị máy chủ cấp trong `getServerInfo`/`getLoginInfo` (chỉ cho khoá không có hằng) → mặc định trong `src/session.rs`.

| Khoá `Domains` | Host mặc định / đã xác minh | Nguồn cập nhật lúc chạy | Dùng cho |
| :--- | :--- | :--- | :--- |
| `auth` | `https://wpa.zaloapp.com` | hằng | QR, đăng nhập mật khẩu, getServerInfo, getLoginInfo |
| `wpa_chat` | `https://wpa.chat.zalo.me` | hằng | getLoginInfo (host thứ hai) |
| `chat` | `https://tt-chat3-wpa.api.zaloapp.com` | hằng (`SERVICE_MAP_V3.chat`) | gửi tin 1-1, quote, link, sticker, delete, undo |
| `group` | `https://tt-group-wpa.zaloapp.com` | hằng | mọi `/api/group/*` trừ cloud message |
| `friend` | `https://tt-friend-wpa.chat.zalo.me` | hằng | bạn bè, tìm theo SĐT |
| `profile` | `https://tt-profile-wpa.chat.zalo.me` | hằng | me-v2, members, lastOnline |
| `conversation`, `convers_api` | `https://tt-convers-wpa.chat.zalo.me` | hằng | get-last-msgs, pinconvers, getUnreadMark |
| `file` | `https://tt-files-wpa.zaloapp.com` | hằng | upload/gửi ảnh, tệp, parselink |
| `sticker` | `https://tt-sticker-wpa.chat.zalo.me` | hằng | danh mục, tìm, gợi ý sticker |
| `voice_call` | `https://voicecall-wpa.chat.zalo.me` | hằng | mọi `/api/voicecall/*` |
| `cloud_msg` | `https://tt-group-cm.api.zaloapp.com` | `zpw_service_map_v3.group_cloud_message` (server cấp) | `/api/cm/*` |
| `lp_chat` | `https://p4-msg-chat.zaloapp.com` | `zpw_chts_new[0]` (server cấp, xoay p4/p5/p7…) | long-poll tin nhắn |
| `lp_control` | `https://p4-ctl-chat.zaloapp.com` | `zpw_ctrs_new[0]` (server cấp) | long-poll điều khiển/VoIP |
| `ws` | `wss://ws1-msg.chat.zalo.me` … `ws8` | `zpw_ws` (server cấp); chỉ nhận `wss://` host Zalo | WebSocket thời gian thực |
| cố định | `https://reaction.api.zaloapp.com` | hằng `endpoints::REACTION` | thả cảm xúc |
| cố định | `https://api.zaloapp.com`, `https://stc-chat.zdn.vn` | hằng `STICKER_API`, `STICKER_CDN` | ảnh sticker |
| cố định | `https://zalo-api.zadn.vn` | `dataUrl` server cấp, mặc định hằng | catalog sticker |
| server cấp | `https://qr-id.zaloapp.com` | từ phản hồi `reqqr` | long-poll QR |
| server cấp | relay UDP `ip:port` | từ `requestcall` (`servers[]`, `rtpIP`) | media VoIP |

### 0.2 Header và client HTTP (`Session::new`, `Session::send`)

- `User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) ZaloPC/26.10.11 Chrome/122.0.0.0 Electron/29.0.0 Safari/537.36` (bắt buộc để máy chủ chấp nhận).
- `Origin: https://chat.zalo.me`, `Referer: https://chat.zalo.me/` ở mọi request.
- Khi đã có `zpw_sek` **và** host thuộc `zalo.me` / `zaloapp.com`: thêm `Cookie: zpw_sek=<token>; _zlang=vi` và header `zpw_sek: <token>`. Host khác không bao giờ nhận cookie.
- Timeout kết nối 10 s, toàn request 35 s (multipart 90 s, long-poll 12 s).
- `rustls`; không proxy riêng.

### 0.3 Tham số chung và mã hoá

| Thành phần | Giá trị / cách tạo | Mã |
| :--- | :--- | :--- |
| Query chung | `zpw_ver=694&zpw_type=30` | `Session::common_params` |
| `params=` sau đăng nhập | `urlencode(base64(AES-CBC(secret_key, IV=0, JSON)))`; `secret_key` giải từ `zpw_enk` (16 hoặc 32 byte) | `Session::encoded_params`, `crypto::encode_aes` |
| `params=` trước đăng nhập (ParamCipher v2) | `zcid = AES(KEY cố định, "30,{imei},{ts}")` hex; `zcid_ext` 10 hex ngẫu nhiên (CSPRNG); `encrypt_key = even(md5(zcid_ext))[..8] + even(zcid)[..12] + rev(odd(zcid))[..12]`; `params = base64(AES(encrypt_key, JSON))`; `signkey = md5("zsecure" + tag + values sorted)` trên `{client_version, enc_ver, params, type, zcid, zcid_ext}` | `param_cipher.rs` |
| `imei` | UUIDv4 + `-` + md5(UA cố định), 69 ký tự, sinh một lần và lưu trong `session.json` | `call::official_imei` |
| `computer_name` | `%COMPUTERNAME%` hoặc `DESKTOP` | `Session::new` |
| Giải mã phản hồi | nếu `data` là chuỗi base64: AES-CBC cùng khoá → JSON envelope mới (có `error_code` riêng) | `Session::maybe_decrypt` |

### 0.4 Mã lỗi chung

| `error_code` | Ý nghĩa | Client làm gì |
| :--- | :--- | :--- |
| `0` | thành công | đọc `data` |
| `600`, `-102` | `zpw_sek` thiếu/hết hạn | đánh dấu `expired_at`, giữ file, mở QR lại (#21) |
| `18012` | QR đã quét, chờ bấm | tiếp tục poll |
| `18022` | sai chữ ký | lỗi lập trình, không retry |
| `18060` | tham số chưa mã hoá | lỗi lập trình, không retry |
| `114` | tham số sai kiểu | không retry |
| `216` | SĐT không có / bị chặn tìm | báo người dùng |
| `429` | giới hạn tần suất | lùi theo thời gian |
| khác ≠ 0 | lỗi dịch vụ | hiển thị, không đụng phiên |

Điều kiện chung của mọi mục có đánh dấu **[Đã đăng nhập]**: `session.login.is_some()` và `zpw_sek` không rỗng và `secret_key` đã có (nếu thiếu, `encoded_params` trả lỗi "no session key yet").

---

## 1. Đăng nhập và phiên

### `GET {auth}/api/login/getServerInfo` — `src/login.rs::get_server_info`

- **Điều kiện**: không cần đăng nhập. Gọi khi khởi động worker và TUI.
- **Query**: tham số *inline* (không `params=`): `imei`, `type=30`, `client_version=694`, `computer_name`, `sp_mtn=-1`, `bkt=-1`, `signkey=md5("zsecure"+"getserverinfo"+values)`.
- **Dùng**: `data.zpw_chts_new`, `zpw_ctrs_new`, `zpw_ws`, `zpw_service_map_v3` → `Session::apply_server_info` (chỉ điền khoá chưa có hằng).

### `GET {auth}/api/login/reqqr` — `src/qr.rs::request_qr`

- **Điều kiện**: không cần đăng nhập; client gọi `s.logout()` trước khi xin QR mới.
- **Query**: `language=vi`, `client_time` (giây), `imei`, `computer_name`, `logged_uids="[]"`, `self_g=1` (xin payload text để tự vẽ), `qr_form=0`, `type`, `client_version`, `signkey` tag `reqqr`.
- **Dùng**: `token_id`, `qr_cnt` (chuỗi QR `http://zaloapp.com/qr/l?tk=zaloqr:v2_…`), `base64_qr` (nếu server tự vẽ), `chk_wait_scan`, `chk_wait_cfirm`, `qr_timeout` (180 s), `qr_retry` (3).

### `GET <chk_wait_scan>&lang=vi` (`qr-id.zaloapp.com/polling/qr/waiting?src=pc&code=…`) — `qr.rs::wait_for_scan`

- **Điều kiện**: có vé QR còn hạn. Dùng `get_raw` (không qua `Session::send`, cần đọc status HTTP).
- **Vòng lặp**: HTTP 408 = chưa quét → ngủ 800 ms; 200 với `error_code 0` và `data` ≠ null → đã quét (`display_name`); lỗi quá `qr_retry` lần → dừng; hết `qr_timeout` → dừng.

### `GET <chk_wait_cfirm>` (`qr-id.zaloapp.com/polling/qr/pc-status?src=pc&code=…`) — `qr.rs::confirm`

- **Điều kiện**: đã quét.
- **Vòng lặp**: 408 → chờ; `18012` → đã quét chờ bấm; `0` → đọc `Set-Cookie: zpw_sek=…` (bỏ giá trị `EXPIRED`) rồi gọi `set_zpw_sek`; mã khác → lỗi.

### `POST {auth}/api/login/loginByPasswv2?{body v2}&ref=https%3A%2F%2Fchat.zalo.me` — `login.rs::login_by_password`

- **Điều kiện**: không cần đăng nhập; GUI chỉ gọi khi người dùng nhập SĐT + mật khẩu (TUI luôn hỏi tương tác, `--pass` đã gỡ).
- **Payload trước mã hoá**: `phone_num`, `iso_country_code` (`VN`), `password = base64(AES-256-CBC(enk, salt + reverse(pwd)))` với `enk` từ `sha256("9fc5vEW3-{phone}-{imei}-{salt}")`, `salt` 8 ký tự CSPRNG, `cptch_token=""`, `cptch_answer=""`, `make_answer=0`, `language=en`, `logged_uids=null`, `computer_name`, `imei`. Body ParamCipher v2, tag `loginbypasswv2`; `ref` thêm sau khi ký.
- **Phản hồi**: giải bằng `cipher.key()`; `error_code` bên trong envelope; `data` → `apply_login_config` (`uid|userId|dkey`, `zpw_enk`, `zpw_sek` (có thể trong `account`), `display_name`, `send2me_id`, bảng host).
- `loginByPassw` (v1, plain `params=`, tag `loginbypassw`) chỉ dùng với `--v1` để chẩn đoán.

### `GET {auth|wpa_chat}/api/login/getLoginInfo?{body v2}&nretry=0` — `login.rs::get_login_info`

- **Điều kiện**: đã có cookie `zpw_sek` (sau QR confirm hoặc login mật khẩu). Thử `auth` trước, rồi `wpa_chat`; nhận host đầu tiên trả `error_code 0`.
- **Payload v2**: `imei`, `computer_name`, `language=vi`, `ts`; tag `getlogininfo`.
- **Dùng**: `zpw_enk` (khoá phiên), `uid`, `send2me_id`, `display_name`, `zpw_service_map_v3`, `zpw_chts_new`, `zpw_ctrs_new`, `zpw_ws`. Sau đó `save_to_file("session.json")`.

### `GET {profile}/api/social/profile/me-v2?{common}&params=…&os=8` — `session.rs::session_status`, `sync_profile_me`

- **Điều kiện**: [Đã đăng nhập]. Gọi khi: khởi động với phiên cũ (kiểm tra phiên), mỗi `RefreshThreads`, sau QR thành công.
- **Payload**: `{avatar_size: 160, imei}`.
- **Dùng**: `data.profile.{userId|uid, displayName|zaloName, avatar|avt}` → cập nhật `login.uin/display_name/avatar`.
- **Phân loại**: `0` Valid; `600`/`-102` Expired; mọi lỗi khác (mạng, HTTP, parse) Unreachable → **không** xoá phiên.

---

## 2. Danh bạ, hồ sơ, bạn bè (`src/chat.rs`)

### `GET {friend}/api/social/friend/getfriends?{common}&params=…` — `get_friends`

- **Điều kiện**: [Đã đăng nhập]. Gọi sau đăng nhập và mỗi `RefreshThreads`; nếu `friend` rỗng thì thử `profile`. Nếu không có khoá phiên, rơi về `plain_params` (server sẽ trả `18060`).
- **Payload**: `incInvalid=1`, `page=1`, `count=20000`, `avatar_size=160`, `actiontime=0`, `imei`.
- **Dùng**: mảng `data` / `data.friends` / `data.data` với `userId|uid|id|friendId|globalId`, `displayName|zaloName|display_name|name`, `avatar|avt`, `isGroup`, `lastActionTime`.

### `POST {profile}/api/social/group/members?{common}` body `params=…` — `get_mini_profiles`

- **Điều kiện**: [Đã đăng nhập]; danh sách uid không rỗng; gọi theo lô ≤ 50 khi thread chưa có tên/avatar thật (`resolve_thread_names`) hoặc để dựng avatar nhóm.
- **Payload**: `friend_pversion_map: ["<uid>_0", …]`.
- **Dùng**: `data.profiles.<uid>.{zaloName|displayName|name, avatar|avt}` (có thể nằm trong chuỗi JSON `data`).

### `GET {profile}/api/social/profile/lastOnline?{common}&params=…` — `get_user_online_status`

- **Điều kiện**: [Đã đăng nhập]; gọi khi mở hội thoại 1-1 (`CheckFriendStatus`).
- **Payload**: `uid`, `conv_type=1`, `imei`.
- **Dùng**: `settings.show_online_status` (ẩn → không hiển thị), `lastOnline` (giây hoặc mili-giây) → "Vừa mới truy cập" nếu ≤ 5 phút.

### `GET {friend}/api/friend/profile/get?{common}&params=…` — `find_user_by_phone`

- **Điều kiện**: [Đã đăng nhập]; người dùng gõ SĐT (chuẩn hoá `0…` → `84…`, ≥ 8 chữ số).
- **Payload**: `phone`, `avatar_size=240`, `language=vi`, `imei`, `reqSrc=40`.
- **Dùng**: `uid|userId|id|globalId`, `display_name|displayName|zalo_name|zaloName`, `avatar|avt`, `status`; sau đó gọi tiếp `reqstatus`. `216` = không có tài khoản hoặc chặn tìm.

### `GET {friend}/api/friend/reqstatus?{common}&params=…` — `get_friend_request_status`

- **Payload**: `fid`, `imei`. **Dùng**: `is_friend`, `is_requesting` (họ mời tôi), `is_requested` (tôi đã mời) → `FriendshipStatus`.

### `POST {friend}/api/friend/sendreq` — `send_friend_request`: `toid`, `msg` ("Xin chào, kết bạn với mình nhé!"), `reqsrc=30`, `imei`, `language=vi`. Lỗi ≠ 0 → bail.

### `POST {friend}/api/friend/accept` — `accept_friend_request`: `fid`, `language=vi`. (GUI thử accept trước khi sendreq để bắt lời mời đến.)

### `POST {friend}/api/friend/reject` — `reject_friend_request`: `fid`, `imei`.

### `POST {friend}/api/friend/undo` — `undo_friend_request`: `fid`, `imei`.

Tất cả: [Đã đăng nhập], body `params=<AES>`, phản hồi qua `maybe_decrypt`.

---

## 3. Nhóm (`src/chat.rs`)

### `GET {group}/api/group/getlg/v4?{common}` — `get_groups` (cmd 12027)

- **Điều kiện**: [Đã đăng nhập]; gọi trong `get_friends` và mỗi `RefreshThreads`. Không có `params`.
- **Dùng**: `data.gridVerMap` (khoá = grid). Sau đó gọi `getmg-v2` cho toàn bộ grid.

### `POST {group}/api/group/getmg-v2?{common}` body `params=…` — `fetch_group_profiles`

- **Điều kiện**: [Đã đăng nhập]; theo lô ≤ 50 grid.
- **Payload**: `gridVerMap: "{\"<grid>\":0,…}"` (chuỗi JSON) để server trả đủ `gridInfoMap` thay vì `unchangedsGroup`; `imei`.
- **Dùng**: `gridInfoMap.<grid>.{name|groupName|displayName, fullAvt|avt|avatar, totalMember, memVerList, creatorId, adminIds, topMember}`, `removedsGroup`.

### `GET {group}/api/group/history?{common}&params=…` — `get_group_history` (cmd 11240)

- **Payload**: `grid`, `count`, `imei`; bản `_raw` chỉ `grid`, `count` (đúng bundle). Hiện GUI dùng `/api/cm/*` thay cho endpoint này.

Lưu ý grid: client bỏ tiền tố `group_`, `g`, `G` trước khi gửi (`clean_grid`).

---

## 4. Nhận tin và hội thoại

### `GET {lp_chat}/?{common}&params=<lastActionId>,<lastActionIdOther>,<urlencode(base64(JSON queues) + ",1")>&ts=<giây>` — `chat.rs::get_chat_from_polling` (cmd 99100)

- **Điều kiện**: [Đã đăng nhập]. GUI gửi `PollMessages` mỗi 1,2 s khi có cửa sổ; worker chạy tối đa 3 vòng liên tiếp khi `data.more == 1` hoặc có `resetLastActionId`; timeout 12 s; body rỗng = keepalive (coi như `{error_code:0}`).
- **Trạng thái** (`PollState`): `last_action_id`, `last_action_id_other`, queues `{"1":{lastId, ids},"2":…,"3":…}`; chỉ cập nhật queue 1–3 (queue 15/16/7/8 là của WebSocket, gửi lên sẽ bị từ chối). `ids` ≤ 10 phần tử tăng dần, ≤ `lastId`.
- **Dùng**: `data.msgs`, `data.groupMsgs`, `data.clearUnreads`, `lastActionId`, `queueStatus`, `more`, `resetLastActionId` → `merge_previews`.

### `GET {lp_control}/?{common}&params=…` — `call.rs::poll_voip_controls` (cmd 99101)

- **Điều kiện**: [Đã đăng nhập]; worker lặp liên tục (ngủ 5 s khi lỗi); queue `4`, `5`. Lần đầu (cursor "1") chỉ nhận lời mời `request|group_request` mới hơn 45 s.
- **Dùng**: sự kiện `act_type == "voip"` với `act` ∈ {`request`, `ring_ring`, `answer`, `cancel`, `end_call`, `group_request`…}; `data.params` là chuỗi JSON (`sessId`, `rtpIP`, `extendData`, `callId`, `callerId`).

### `GET {convers_api}/api/preloadconvers/get-last-msgs?{common}&params=…` — `get_last_msgs` (cmd 10053)

- **Điều kiện**: [Đã đăng nhập]; gọi trong mỗi `poll_new_messages` (một lần với `{}` rồi theo lô 25 thread; grid nhóm có hậu tố `_1`, 1-1 `_0`) và khi mở hội thoại 1-1.
- **Payload**: `threadIdLocalMsgId: "<JSON map>"`, `imei`.

### `GET {conversation}/api/pinconvers/list?{common}` — `get_pinned` (cmd 12120): [Đã đăng nhập], không `params`.

### `GET {conversation}/api/conv/getUnreadMark?{common}&params=…` — `unread.rs::fetch_manual_unread_marks` (cmd 12011)

- **Payload**: `{}` mã hoá. **Dùng**: danh sách thread đánh dấu chưa đọc thủ công.

### `GET {cloud_msg}/api/cm/getrecentv2?{common}&params=…&nretry=0` và `/api/cm/getoldv2` — `get_cloud_messages` (cmd 12506/12507)

- **Điều kiện**: [Đã đăng nhập]; cho mọi nhóm và cho Cloud của tôi (`send2me_id`) trong mỗi `poll_new_messages`; khi mở nhóm: `getrecentv2` 50 tin rồi tối đa 5 trang `getoldv2`.
- **Payload**: `groupId` (grid sạch), `globalMsgId` (cursor; `"9223372036854775808"` = mới nhất), `count` (50), `msgIds: []`, `imei`, `src` (1 recent, 3 old), `isOA=1` khi là Cloud của tôi.
- **Dùng**: `data.groupMsgs|msgs|messages`, `hasMore`, `lastMsgId`; `data` có thể là chuỗi JSON.

---

## 5. Gửi tin nhắn (`src/chat.rs`)

Điều kiện chung: [Đã đăng nhập]; `clientId = epoch_ms`; body `params=<AES>`; `error_code ≠ 0` → `SessionError` (nếu `600`/`-102` GUI mở QR lại).

| Endpoint | Hàm | Payload | Ghi chú |
| :--- | :--- | :--- | :--- |
| `POST {chat}/api/message/sms` | `send_message` | `toid`, `message`, `clientId`, `imei` | nếu `chat` là host long-poll (`-msg-chat.`) hoặc `tt-chat2-wpa` thì dùng `MSG_CHAT_FALLBACK` |
| `POST {group}/api/group/sendmsg` | `send_group_message` | `grid`, `message`, `clientId`, `imei`, `visibility=0` | |
| `POST {chat}/api/message/quote` (cmd 11771) | `send_quote_message` | `toid`, `message`, `clientId`, `qmsgOwner` (uid người gửi gốc hoặc `toid`), `qmsgId` (u64), `qmsgCliId`, `qmsgType=1`, `qmsgTs`, `qmsg`, `imei`, `ttl=0`, `zsource=101` | tin gốc phải có id server (không `out_`/`local_`); thất bại → fallback `sms` |
| `POST {group}/api/group/quote` (cmd 11197) | `send_quote_message` | như trên + `grid`, `language=vi`, `visibility=0` | |
| `GET {file}/api/message/parselink?{common}&params=…` (cmd 11285) | `parse_link_metadata` | `link`, `version=1`, `imei` | **nguồn duy nhất** cho mọi bản xem trước liên kết; client không bao giờ tải URL trực tiếp (ADR-0005) |
| `POST {chat}/api/message/link` (cmd 11286) | `send_link_message` | `toId`, `msg`, `href`, `src`, `title`, `desc`, `thumb`, `type=0`, `media` (JSON chuỗi), `clientId`, `mentionInfo=""`, `language=vi`, `ttl=0`, `imei` | lỗi → log + fallback `sms` |
| `POST {group}/api/group/sendlink` (cmd 11193) | `send_link_message` | như trên với `grid`, `mentionInfo="[]"` | |
| `POST {chat}/api/message/sticker` (cmd 11154) | `send_sticker` | `toid`, `stickerId`, `cateId`, `type`, `clientId`, `imei`, `zsource=101` | |
| `POST {group}/api/group/sticker` (cmd 11165) | `send_sticker` | `grid`, … | |
| `POST reaction.api.zaloapp.com/api/message/reaction` (cmd 12410) | `send_reaction` | `react_list: [{message: "<JSON {rMsg:[{gMsgID,cMsgID,msgType}], rIcon, rType, source:6}>", clientId}]`, `toid` | `rIcon`/`rType`: 👍 `/-strong` 3, ❤️ `/-heart` 5, 😆 `:>` 0, 😮 `:o` 32, 😢 `:-((` 2, 😠 `:-h` 20; tin local gửi `gMsgID=""` |
| `POST reaction.api.zaloapp.com/api/group/reaction` (cmd 12411) | `send_reaction` | + `grid`, `imei` | |
| `POST {chat}/api/message/delete` (cmd 11779) | `delete_message_for_me` | `toid`, `cliMsgId`, `msgs: [{cliMsgId, globalMsgId, ownerId, destId}]`, `onlyMe=1`, `imei` | chỉ gọi khi tin có id server; bản cục bộ xoá thẳng SQLite |
| `POST {group}/api/group/deletemsg` (cmd 11834) | `delete_message_for_me` | `grid`, … | |
| `POST {chat}/api/message/undo` (cmd 11281) | `recall_message` | `msgId`, `toid`, `clientId`, `cliMsgIdUndo` | chỉ tin của mình |
| `POST {group}/api/group/undomsg` (cmd 11192) | `recall_message` | `msgId`, `grid`, `clientId`, `visibility=0`, `cliMsgIdUndo`, `imei` | |

---

## 6. Tệp đính kèm (`src/chat.rs::send_attachment`)

- **Điều kiện**: [Đã đăng nhập]; tệp là file thường ≤ 1 GiB (`MAX_ATTACHMENT_BYTES`), không rỗng. Multipart dùng `Session::post_multipart` (boundary CSPRNG, tên file đã lọc `"` `\r` `\n`, timeout 90 s), cookie/header `zpw_sek` gắn thủ công.
- **Query mỗi request**: `{common}&params=<AES>&type=<2 cho 1-1 | 11 cho nhóm>`; `send`/`msg` không có `type`.

| Bước | Endpoint 1-1 / nhóm | Payload mã hoá | Phản hồi dùng |
| :--- | :--- | :--- | :--- |
| Ảnh (`png jpg jpeg webp gif bmp`), mỗi chunk 1 MiB | `POST multipart {file}/api/message/photo_original/upload` / `/api/group/photo_original/upload`, field `chunkContent` | `totalChunk`, `fileName`, `clientId`, `totalSize`, `imei`, `isE2EE=0`, `jxl=0`, `chunkId` (từ 1), `toid`/`grid` | `photoId|clientFileId`, `normalUrl|hdUrl|oriUrl|url`, `thumbUrl` |
| Gửi ảnh đã tải | `POST {file}/api/message/photo_original/send` / `/api/group/photo_original/send` | `photoId`, `clientId`, `desc`, `width`, `height`, `rawUrl`, `hdUrl`, `thumbUrl`, `oriUrl`, `normalUrl`, `hdSize`, `zsource=-1`, `jcp="{\"convertible\":true}"`, `imei`, `toid`/`grid`(+`visibility=0`) | `data.msgId|cliMsgId` |
| Dự phòng ảnh một lần (khi upload không trả `photoId`) | `POST multipart {file}/api/message/photo` (cmd 11152) / `/api/group/sendpt`, field `fileContent` | `clientId`, `desc`, `width`, `height`, `imei`, `toid`/`grid` | `msgId` |
| File/video, mỗi chunk 1 MiB | `POST multipart {file}/api/message/asyncfile/upload` / `/api/group/asyncfile/upload`, field `chunkContent` | `totalChunk`, `fileName`, `clientId`, `totalSize`, `imei`, `isE2EE=0`, `chunkId`, `toid`/`grid` | `fileId|clientFileId`, `fileUrl|url|normalUrl` |
| Gửi file đã tải | `POST {file}/api/message/asyncfile/msg` / `/api/group/asyncfile/msg` | `fileId`, `fileUrl`, `checksum` (MD5 toàn file), `checksumSha=""`, `extension`, `totalSize`, `fileName`, `clientId`, `fType` (2 video, 1 khác), `fileCount=0`, `fdata="{}"`, `imei`, `toid`/`grid` | `msgId`, `fileUrl|url|href` |
| Dự phòng file một lần | `POST multipart {file}/api/message/sharefile_full` (cmd 11146) / `/api/group/sharefile_full`, field `fileContent` | `toid`/`grid`, `clientId` | `msgId` |

MIME suy từ đuôi file (`mp4`, `mov`, `mkv`, `webm`, `mp3`, `wav`, `pdf`, `zip`, `txt|md|log`, còn lại `application/octet-stream`).

---

## 7. Sticker (`src/chat.rs`, `src/endpoints.rs`)

| Endpoint | Hàm | Điều kiện / thời điểm | Payload | Dùng |
| :--- | :--- | :--- | :--- | :--- |
| `GET {sticker}/api/message/sticker/category/list/v2?{common}&params=<AES("")>` | `fetch_sticker_store_packs` | [Đã đăng nhập]; chỉ khi cache `stickers_cache/store_packs.json` có < 20 bộ | chuỗi rỗng mã hoá | `exCateIds`, `dataUrl` |
| `GET <dataUrl>` (mặc định `https://zalo-api.zadn.vn/api/emoticon/pcweb/getAllCate`) | như trên | host phải thuộc Zalo (`is_trusted_media_url`), body ≤ 16 MiB, gzip tự nhận | — | mảng `{id, name, totalImage, iconUrl|thumbImg}`; bỏ `id==0`, trong `exCateIds`, tên chứa "test" |
| `GET {sticker}/api/message/sticker/category/sticker_detail?{common}&params=…` (cmd 11284) | `fetch_sticker_pack_items` | [Đã đăng nhập]; cache `pack_<cid>.json`; `cid=0` trả 30 sticker "Củ hành" cục bộ | `cid` (chuỗi) | `data[].{id|stickerId, cateId, type}` |
| `GET {sticker}/api/message/sticker/suggest/keywords?{common}&params=…` (cmd 11740) | `fetch_sticker_suggest_keywords` | sau đăng nhập; cache `suggest_keywords_v2.json` (< 100 từ thì tải lại) | `imei` | `data.keywords[]` (bỏ `zkwd_*`) |
| `GET {sticker}/api/message/sticker/suggest/stickers?{common}&params=…` (cmd 11741) | `search_stickers_by_keyword` | khi gõ (debounce 145 ms) hoặc tìm trong kho; ≤ 4 biến thể từ khoá (có dấu/không dấu/đồng nghĩa) chạy song song | `keyword`, `gif=1`, `guggy=0`, `imei` | `data.sugg_sticker[].{sticker_id|id, cate_id|cateId, type}` |
| `GET {sticker}/api/message/sticker/search?{common}&params=…` (cmd 11747) | như trên | cùng lúc với suggest | `keyword`, `limit=30`, `srcType=0`, `imei` | `data[]` |
| `GET https://api.zaloapp.com/api/emoticon/sticker/webpc?eid=<id>&size=130` | `endpoints::sticker_thumb_url` | khi vẽ sticker chưa có texture; cache `stickers_cache/stk_<cid>_<sid>.png`; tải bounded 16 MiB qua client phiên (cookie không gửi vì host khác domain jar) | — | PNG |
| `GET https://stc-chat.zdn.vn/images/stickers/default/thumb/<1..30>.png` | như trên | `cate_id == 0` | — | PNG |

---

## 8. Gọi thoại và video (`src/call.rs`, `src/media.rs`)

### 8.1 Signaling HTTP — `voicecall_get`

- **Điều kiện**: [Đã đăng nhập]; `uin` phải là số (nếu không, worker gọi `me-v2` trước). Đích gọi không rỗng.
- **Request**: `GET {voice_call}{path}?zpw_ver=694&zpw_type=30&params=<AES(JSON)>`; phản hồi `maybe_decrypt`; `error_code ≠ 0` → lỗi; `data` chuẩn hoá thành object (`normalize_voicecall_data`). Mọi request/response ghi `call_trace.log` **chỉ khi** `--dev` hoặc `ZALO_VOIP_LOG` (đã redact imei/session/p2p).

| Mã | cmd bundle | Đường dẫn | Khi gọi | Payload | Dùng |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 401 | 11300 | `/api/voicecall/requestcall` | người dùng bấm gọi | nếu đã có relay cache: `callId`, `calleeId`, `codec`, `extendData`, `rtcpAddress`, `rtpAddress`, `session` (đúng 7 khoá); nếu chưa: `calleeId`, `callId`, `codec="[]"`, `typeRequest`, `imei` | `id` (callId thật), `fromId`, `toId`, `sessId`, `servers[]` (relay), `rtpIP`, `status` |
| 416 | 11301 | `/api/voicecall/request` | ngay sau probe relay (280 ms) | `calleeId`, `rtcpAddress`, `rtpAddress`, `codec` (`opus/16000/1` PT 112, ptime 20), `session`, `callId`, `imei`, `extendData`, `subCommand=3` | `status` |
| 407 | 11302 | `/api/voicecall/ringring` | callee nhận `request` | `callerId`, `callId`, `status=0`, `imei` | — |
| 402 | 11303 | `/api/voicecall/answer` | callee bấm Nghe (`status 0`) hoặc Từ chối (`status 3`); gửi **trước** khi bật media | `callerId`, `callId`, `status`, `codec`, `rtcpAddress`, `rtpAddress`, `session_id`, `extendData`, `subCommand=3` (bỏ khoá rỗng, server trả `114` nếu `rtpAddress`/`session_id` rỗng) | `status` |
| 408 | 11304 | `/api/voicecall/answerack` | caller nhận `answer status 0` | `calleeId`, `callId` | — |
| 405 | 11305 | `/api/voicecall/cancel` | caller kết thúc trước khi kết nối, hoặc hết 60 s | `calleeId`, `callId` | — |
| 409 | 11306 | `/api/voicecall/endcall` | kết thúc khi đã kết nối, hoặc media lỗi | `partnerId`, `callId`, `duration` | — |
| 406 | 11311 | `/api/voicecall/logendcall` | sau 409 | `callId`, `logdata` | — |
| 411–414 | 11307–11310 | `holdreq`, `holdresp`, `unholdreq`, `unholdresp` | giữ máy (ánh xạ sẵn, UI chưa dùng) | `callId`, `partnerId` | — |
| 12433 | — | `/api/voicecall/group/requestcall` | gọi nhóm | `groupId`, `callId`, `codec`, `extendData`, `session`, `members` | `servers`, `sessId` |
| 12434 | — | `/api/voicecall/group/request` | sau probe | như 416 với `groupId` | — |
| 12437 / 12438 | — | `/api/voicecall/group/cancel`, `/api/voicecall/group/endcall` | kết thúc nhóm | `groupId`, `callId` | — |
| — | — | `/api/voicecall/group/{answer, ringring, join, adduser, notify, ping, state, broadcast, finish, sharescreen}`, `/api/voicecall/conf`, `/api/voicecall/requestsharescreen` | ánh xạ trong `dispatch_send_signal_inner`, chưa có UI | | |

`extendData` client dựng (`build_caller_extend_data`): `callType` (0 thoại, 1 video), `fecTP=0`, `gccAudio=1`, `gccMode=1`, `maxFT=60`, `newZrtc=1`, `numServers`, `p2p` (LAN và public `ip:port`), `packetMode=2`, `platform=2`, `sP2P=0`, `serverAddr` (relay đã probe), `serverResult {recv, rtt}` (0,0 khi không có echo), `spTcp=1`, `srtcp=0`, `srtpMode=0`, `supportCallBusy=1`, `supportHevcDecode=0`, `video.codec h264/97`.

### 8.2 Sự kiện đến (qua WebSocket 601–604 và long-poll `lp_control`)

`act`: `request` / `group_request` (lời mời; `params` JSON chuỗi gồm `sessId`, `rtpIP`, `extendData`, `callerId`, `callId`, `isVideo`), `ring_ring`, `answer` (`status` 0 nhận, 1–4 bận/từ chối, 5 degraded), `cancel`, `end_call`. Khử trùng lặp theo `(act, callId, ts)`; replay > 90 s bị bỏ.

### 8.3 Relay UDP (sau 401)

- **Điều kiện**: địa chỉ phải là `SocketAddr` IP công khai (`is_global_ip`), ưu tiên `servers[]`/`rtpIP` của server; `extendData.serverAddr` của peer chỉ là dự phòng; tối đa 8 relay. Bind `0.0.0.0:0` cho RTP và RTCP.
- **Gói điều khiển** (header 21 byte LE `[0x01][payload_type][seq u32][ssrc u32][from_id u32][call_or_magic u32][cmd u16][sub_cmd u8]`): `InitZRTP` cmd 0x0B (caller) / 0x0C (callee), sub 2, thân `call_id, partner_uid, len+sessId` → phản hồi `res, call_id, token u32, "ip|port"`; `Echo` cmd 5 (đo RTT); `Ping` cmd 2 mỗi 400 ms với token; `End` cmd 0x20 khi kết thúc.
- **Media**: Opus 16 kHz mono 20 ms → RTP PT 112 (SSRC = uid) bọc `[0x03][token]`; H.264 15 fps PT 97/98 (IDR) bọc `[0x0D][token]`, mảnh `[0x1C][flags]` ≤ 1200 byte; RTCP RR/SR/REMB mỗi 2 s. Chỉ nhận gói từ IP relay đã biết. **Không mã hoá đầu-cuối** (`media::MEDIA_ENCRYPTED = false`).

---

## 9. WebSocket điều khiển (`src/socket.rs`)

- **URL**: `wss://ws<N>-msg.chat.zalo.me/?zpw_ver=694&zpw_type=30&t=<ms>`; danh sách từ `zpw_ws`, xoay sang shard kế khi đóng với `3002 NOT_FOUND_WORKER`; chỉ chấp nhận `wss://` trên host Zalo. Header: `Cookie: zpw_sek=…; _zlang=vi`, `zpw_sek`, `Origin: https://chat.zalo.me`, UA như HTTP.
- **Điều kiện**: [Đã đăng nhập]; worker mở lại khi `zpw_sek` đổi; reconnect vô hạn, backoff 2 → 30 s.
- **Khung**: `u8 version=1 | u16 LE cmd | u8 subCmd | JSON`.
- **Giải mã `data`** theo `encrypt`: 0 rõ; 1 base64 + inflate; 2 AES-GCM + inflate; 3 AES-GCM. Khoá = base64 `key` trong frame AUTHEN (16/32 byte), blob `iv(16) | aad(16) | ct+tag`; giải nén tối đa 16 MiB.

| cmd | Hướng | Nội dung | Phản ứng của client |
| :--- | :--- | :--- | :--- |
| 1 AUTHEN | server → client | `{key, encrypt, data:{error_code, ctrl_qCmds[], qCmds[]}}` | gửi `pull` cho từng queue được liệt kê (`{first:true, reqId:"req_N", lastId:1, preIds:[]}`); không tự thêm queue (gây 3002) trừ khi `ZCALL_WS_FORCE_VOIP=1` (debug) |
| 2 PING sub 1 | client → server mỗi 15 s | `{eventId: ms}` | |
| 501 / 521 | server → client | tin 1-1 / nhóm thời gian thực | `merge_previews` → `ThreadsUpdated` |
| 510 / 511 / 513 / 514 / 515 / 517 / 518 | hai chiều | kéo tin offline theo queue; `data.more == 1` → `pull_next` với `lastActionId`/`queueStatus` | |
| 533 HAS_MSG | server → client | `{queueId, ids}` | gửi 570 `{queueId, ids, lastId:"1", reqId}` |
| 534 | server → client | tin bỏ lỡ | như 501 |
| 570 GET_MSG | hai chiều | nội dung tin theo ids | |
| 601 / 602 / 603 / 604 | hai chiều | sự kiện điều khiển; 604 là queue VoIP | `extract_voip_controls` → `CallBridge` |

---

## 10. Kết nối ngoài hệ thống Zalo

| Đích | Điều kiện | Request | Ràng buộc |
| :--- | :--- | :--- | :--- |
| `GET https://api.github.com/repos/ctdoteam1110/ctdozlite/releases/latest` (dự phòng `/releases?per_page=5`) | **chỉ khi** người dùng bật `auto_check_updates` trong Cài đặt (mặc định tắt) hoặc bấm kiểm tra thủ công | `Accept: application/vnd.github+json`, UA `ctdozlite/<ver>`, client riêng không cookie, timeout 12 s | chọn asset theo hệ máy; đọc `SHA256SUMS.txt` |
| `GET https://github.com/.../releases/latest/download/latest.json` | như trên, khi API bị giới hạn | — | cần `sha256.<asset>` trong manifest |
| `GET <browser_download_url>` | người dùng bấm Cập nhật | `updater::download_release`: ≤ 512 MiB, ≥ 512 KiB, SHA-256 **bắt buộc**, magic PE/ELF | URL phải thuộc `github.com` / `objects.githubusercontent.com` |

Không còn request nào khác ra ngoài Zalo: xem trước liên kết luôn đi qua `parselink`; ảnh/thumbnail trong tin nhắn chỉ tự tải khi host thuộc `zalo.me`, `zaloapp.com`, `zdn.vn`, `zadn.vn` (`endpoints::is_trusted_media_url`), host khác cần người dùng bấm.

---

## 11. Bổ sung từ upstream v0.0.4 (2026-11-02)

### 11.1 Failover nhiều host (`Session::request_with_candidates`, `Domains::candidates`)

Với các khoá `chat`, `friend`, `group`, `conversation`, `profile`, client thử lần lượt: hằng đã xác minh → mọi host máy chủ liệt kê trong `zpw_service_map_v3` (`service_map_candidates`) → host cũ đã lưu trong `session.json` (`saved_candidates`). Host thắng được ghi nhớ trong phiên (`remember_winner`). Các host dự phòng đã thấy: `tt-chat4-wpa.api.zaloapp.com`, `tt-chat5-wpa.api.zaloapp.com`, `tt-friend2-wpa.chat.zalo.me`, `legacy-saved-chat.zaloapp.com`. Chỉ lỗi mạng/HTTP 5xx/404 mới chuyển host (`should_failover_candidate`); lỗi `error_code` của Zalo thì không.

### 11.2 Nhật ký và hồ sơ xã hội (`src/feed.rs`)

| Method | Host | Đường dẫn | Điều kiện / thời điểm | Payload | Dùng |
| :--- | :--- | :--- | :--- | :--- | :--- |
| GET | `profile` | `/api/social/friend/getprofiles?{common}&params=…&os=8` | [Đã đăng nhập]; mở trang Nhật ký của một bạn | `fid`, `avatar_size`, `language=vi`, `imei` | ảnh bìa (`cover`), avatar, `sdob`, `bio`, tên |
| GET | `profile` | `/api/social/avatar-list?{common}&params=…&os=8` | mở kho ảnh của chính mình | `page`, `count`, `imei` | danh sách ảnh (`photo_id`, `url`, `thumbnail`, kích thước) |
| POST | `profile` | `/api/social/friend/getprofiles/v2?{common}` | tải timeline: hồ sơ hàng loạt | `friend_pversion_map: ["<uid>_0", …]` | `data.changed_profiles` |
| WebSocket | `ws` | cmd `50001` (`CMD_FEED_TIMELINE`), subCmd `0x00150102` lấy feed / `0x00220411` đăng / `0x26580022` sửa | gói nhị phân do `feed::packet::build_request_packet` dựng | `owner_id`, `viewer_id`, `tab`, `page`, `count` | timeline (hiện client dựng feed từ hồ sơ + ảnh, gói socket được đóng gói nhưng chưa dùng làm nguồn chính) |

### 11.3 Tin hẹn giờ (`src/scheduler.rs`)

Không có endpoint mới: tin được lưu trong bảng `scheduled_messages` của `zalo_history.db`; worker quét mỗi 2 giây và gửi qua `send_message` / `send_group_message` (mục 5) khi đến hạn.

### 11.4 Đồng bộ lịch sử có cursor (`src/history_sync.rs`)

Dùng `GET {cloud_msg}/api/cm/getoldv2` (mục 4) theo trang 50 tin, tối đa 20 trang mỗi lần chạy, nghỉ 400 ms giữa các trang, cursor là `msg_id` cũ nhất đã lưu. Chỉ cho nhóm và Cloud của tôi; 1-1 không có API lịch sử qua HTTP.

### 11.5 Server cục bộ cho công cụ ngoài (`src/api_mcp.rs`) — không gọi Zalo trực tiếp

Chỉ chạy khi có `--api-port <PORT>` (HTTP trên `127.0.0.1`) hoặc `--mcp` (JSON-RPC 2.0 qua stdio). Trên nhánh `personal`, HTTP **bắt buộc** `CTDOZLITE_API_TOKEN` (≥ 16 ký tự, so sánh thời gian hằng) trong `Authorization: Bearer`, không trả header CORS, preflight bị từ chối. Route: `GET /api/v1/{status, threads, search, scheduled, autoreply, export}`, `POST /api/v1/{messages/send, messages/schedule, autoreply, mcp}`. Mọi hành động gửi tin của API này đi qua các endpoint Zalo ở mục 5 với phiên hiện tại; `autoreply_rules.json` rỗng mặc định nên không tự trả lời.

---

## 12. Bổ sung từ upstream v0.0.6 (2026-11-03)

Trích từ `src/chat.rs`, `src/history_sync.rs`, `src/unread.rs`, `src/session.rs` sau khi gộp v0.0.6 vào nhánh `personal`. Mọi request vẫn đi đường chuẩn: `params` → `ParamCipher v2` (`encoded_params`) → `signkey` → cookie phiên; phản hồi giải mã qua `maybe_decrypt`.

### 12.1 Tắt thông báo hai chiều (#67)

| Method | Host | Đường dẫn | Điều kiện / thời điểm | Payload (`params` mã hoá) | Phản hồi dùng | Mã |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| POST | `profile` | `/api/social/profile/setmute?{common}` body `params=…` | [Đã đăng nhập]; người dùng bấm Tắt/Bật thông báo trên một hội thoại (`WorkerCmd::SetMute`) | `toid` (ID đã bỏ tiền tố `group_`/`g`), `duration` (`3600`, `14400`, số giây tới 8:00 sáng, hoặc `-1` = vô thời hạn), `action` (`1` tắt, `2` bật), `startTime` (giây Unix), `muteType` (`1` cá nhân, `2` nhóm), `imei` | chỉ kiểm `error_code`; ngay sau đó gọi `getmute` để lấy bản chuẩn | `chat::set_mute_conversation` (cmd 11288) |
| GET | `profile` | `/api/social/profile/getmute?{common}&params=…` | sau đăng nhập, mỗi tick danh sách hội thoại, và sau mỗi `setmute` | `imei` | `chatEntries[]`, `groupChatEntries[]` với `id`, `muteMode` (chỉ nhận `1`), `duration` (`0` bỏ qua, `<0` vô hạn), `startTime` hoặc `systemTime` → `thread_id → until_unix` (`unread::extract_mute_map`), đồng thời gắn `presentation.muted` (`merge_mute_metadata`) | `chat::get_mute_conversations` (cmd 11280) |
| WS / long-poll | `ws`, `lp_control` | gói `cmd 601/602/603` hoặc `lp_control` có `act_type: "mute"`, `act: "set" \| "unset"` | nhận bất kỳ lúc nào khi máy khác đổi trạng thái | `controls[]` hoặc `data` phẳng: `id`/`toid`, `isGroup`/`muteType`, `duration`, `startTime` | `MuteControlEvent { thread_id, is_group, muted, muted_until, settings_until }` → `AppSettings::muted_threads` | `chat::extract_mute_controls` (gọi từ `call.rs:986` và `worker.rs:594`) |

Điểm cần nhớ: máy chủ là nguồn chuẩn; client **không** tự tin vào kết quả `setmute` mà luôn đọc lại `getmute`. Thời hạn hết thì bỏ qua entry (`exp <= now`).

### 12.2 Đồng bộ tin nhắn từ điện thoại (#68)

Kích hoạt từ Cài đặt (`WorkerCmd::SyncPhoneMessages`), chạy trên worker với cờ `PHONE_SYNC_RUNNING` (chặn chạy trùng) và `PHONE_SYNC_CANCEL` (huỷ giữa chừng). `public_key` là 32 byte ngẫu nhiên mã hoá base64 (`history_sync::generate_sync_public_key`), dùng chung cho cả ba request của một lượt.

| Method | Host | Đường dẫn | Điều kiện / thời điểm | Payload (`params` mã hoá) | Phản hồi dùng | Mã |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| GET | `file` | `/api/message/pull_mobile_msg?{common}&params=…` | pha 1 của lượt đồng bộ | `pc_name` (tên máy), `public_key`, `from_seq_id=0`, `is_retry=0`, `min_seq_id=0`, `temp_key=""`, `imei` | lỗi bị bỏ qua (`let _ =`), chỉ là tín hiệu đánh thức điện thoại | `chat::request_phone_sync` (cmd 12000) |
| GET | `file` | `/api/transfer-sync-v2/request-sync?{common}&params=…` | ngay sau `pull_mobile_msg` | `pc_name`, `public_key`, `version=1`, `is_retry=0`, `temp_key=""`, `imei` | như trên | `chat::request_phone_sync_v2` (cmd 12888) |
| GET | `file` | `/api/message/cancel_pull_mobile_msg?{common}&params=…` | người dùng bấm Huỷ trong lúc đồng bộ | `public_key`, `imei` | như trên | `chat::cancel_phone_sync` (cmd 12700) |

Các pha sau **không có endpoint mới**: nghỉ 600 ms, rút tối đa 5 lô `lp_chat` (`get_chat_from_polling`, mục 4, dừng khi `more == 0` và không có `resetLastActionId`), rồi `get-last-msgs` theo lô 25 hội thoại (mục 4), rồi `getrecentv2`/`getoldv2` cho Cloud và nhóm (mục 4), cuối cùng ghi `zalo_history.db` + `threads_cache.json`. Kết quả về GUI qua `PhoneSyncProgress` / `PhoneSyncFinished { added, active_threads, cancelled, error }`.

### 12.3 Proxy và đa tài khoản (#64) — thay đổi đường đi, không thêm endpoint

- Nguồn cấu hình theo thứ tự: `--proxy <url>` → biến môi trường `ZALO_PROXY` rồi `CTDOZLITE_PROXY` → `proxy_url` lưu trong `session.json` / `settings.json`. `session::normalize_proxy_url` chỉ chấp nhận `http://`, `https://`, `socks5://`, `socks5h://` (chuỗi `host:port` trần được gắn `http://`); `redact_proxy_url` che `user:pass@` khi hiển thị.
- **Phạm vi proxy chỉ là client HTTP `reqwest` trong `Session`** (`build_http_client`). WebSocket (`src/socket.rs`), relay UDP VoIP (`src/call.rs`, `src/media.rs`), tải ảnh/thumbnail và updater **không** đi qua proxy. Người dùng bật proxy để ẩn địa chỉ IP vẫn lộ IP thật trên các đường này; xem phát hiện bổ sung trong `docs/KIEM_TOAN_HE_THONG.html`.
- `--profile <tên>` (chỉ chữ, số, `-`, `_`) đặt thư mục dữ liệu thành `<data_dir>/profiles/<tên>` nên mỗi tài khoản có `session.json`, DB và cookie riêng; nút "Mở cửa sổ Zalo mới" trong Cài đặt khởi chạy thêm một tiến trình với `--gui --profile <tên> [--proxy <url>]`.

### 12.4 Sticker GIF nội bộ và định dạng văn bản — không gọi mạng

Gói "Mèo Ami Bụng Bự" nhúng sẵn 15 cặp GIF/PNG trong `src/assets/ami_fat_cat/` (`cate_id` riêng, `sticker_id` 999001–999015) và được gửi như sticker thường qua `/api/.../sticker` (mục 7). Giải mã `params.textProperties` (RTF của Zalo PC/Mobile) và render Markdown diễn ra hoàn toàn trên client từ nội dung tin nhắn đã nhận; đây là bề mặt phân tích dữ liệu không tin cậy mới, chưa có bằng chứng lỗi nhưng cần đọc kỹ khi kiểm toán lại.
