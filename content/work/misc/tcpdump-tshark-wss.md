---
title: "用 tcpdump 和 tshark 分析 wss 流量"
date: 2024-12-13
taxonomies:
  categories: ["network"]
---

## 需求

用 tcpdump 抓本地所有 wss 流量并得到对应的时间戳和明文

## 实现

### 1. SSL Keylog

为了解密 TLS 流量，需要得到每个会话的 keylog

如果 TLS 是自己写的，以 c++ openssl 为例，可以通过 `SSL_CTX_set_keylog_callback` 设置 keylog 回调，直接将输出的 keylog 写入文件即可。无须维护 keylog 和会话的对应关系。

假设最后得到的 keylog 文件是 `keylog`，be-like:
```
CLIENT_RANDOM 1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef 1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
CLIENT_RANDOM 1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef 1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
CLIENT_RANDOM 1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef 1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
```

### 2. 抓包

为了成功解密 TLS 流量，抓到的包必须包含 TLS handshake，所以抓包需要在目标连接建立以前开始

```bash
sudo tcpdump -w output.pcap -i any 'tcp port 443 or tcp port 9443'
# run wss client ...
```

### 3. 解密

抓包结束后，只要提供 keylog 文件和抓包文件，tshark 会自动解密并输出明文

```bash
tshark -r output.pcap -o "tls.keylog_file:./keylog"  -Y websocket -T fields -e frame.time -e websocket.payload.text
```

## 遇到的问题

### truncated

一开始 `tshark -e text` 来获取明文，发现 websocket payload 超出 226B 的部分会被截断，并且输出前会展示 [truncated] 或者 [...]，[原因](https://osqa-ask.wireshark.org/questions/43023/want-to-use-tshark-to-decode-a-specific-packet-and-do-not-truncate-lines/)

解决方法参考 [Displaying WebSocket traffic content](https://ask.wireshark.org/question/35302/displaying-websocket-traffic-content/)
- 使用 `-e websocket.payload.text` 来获取明文

如果一定要用 `-e text`，可以改源码里的 `ITEM_LABEL_LENGTH`，然后从源码编译 tshark

## 附

### 从源码编译 tshark，并修改 ITEM_LABEL_LENGTH

修改 `ITEM_LABEL_LENGTH` 为 2048:
```
diff --git a/epan/proto.h b/epan/proto.h
index 0d2a27ee86..42d627c168 100644
--- a/epan/proto.h
+++ b/epan/proto.h
@@ -48,7 +48,7 @@ extern "C" {
 WS_DLL_PUBLIC int hf_text_only;
  
 /** the maximum length of a protocol field string representation */
-#define ITEM_LABEL_LENGTH       240
+#define ITEM_LABEL_LENGTH       2048
  
 #define ITEM_LABEL_UNKNOWN_STR  "Unknown"
```

编译流程
```bash
git clone https://gitlab.com/wireshark/wireshark.git
cd wireshark
# setup， 需要根据系统选择脚本，参考 https://www.wireshark.org/docs/wsdg_html_chunked/ChapterSetup.html
sudo ./tools/rpm-setup.sh 
# 应用修改，假设修改是 patch.diff
git apply patch.diff
# 编译安装
mkdir build && cd build
cmake .. -DBUILD_wireshark=OFF
make && sudo make install
```
