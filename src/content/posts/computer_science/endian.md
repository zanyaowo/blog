---
title: Endianness 機制的探討
published: 2026-03-29
description: "探討 Big Endian 與 Little Endian 的差異及其在實務上的影響"
pinned: false
tags: [endian, byte-order, network, memory]
category: Computer Science
draft: false
---

## 什麼是 Endianness

Endianness 是指位元組排列的順序由 ISA（Instruction Set Architecture，指令集架構）決定，其中探討的問題在於記憶體是一個線性一維陣列，每個單元存放一個 byte，每個 byte 有自己唯一的存放位置。當一個 word 存入時該如何決定放入資料的順序 [2]。
Endianness 有兩種形式，Big Endian 和 Little Endian。

- Big Endian：將資料的 MSB 存放在低位址，LSB 存放在高位址的方法，這種存放方式能很輕易的就看出原本存放了什麼資料。
- Little Endian：反過來，將 MSB 存放在高位址，LSB 存放在低位址的方法
- Middle Endian：混合使用如 PDP-11

以 `0x12345678` 為例：

```bash
address | Big    | Little | Middle (PDP-11)
-------------------------------------------
0x00    | 0x12   | 0x78   | 0x34
0x01    | 0x34   | 0x56   | 0x12
0x02    | 0x56   | 0x34   | 0x78
0x03    | 0x78   | 0x12   | 0x56
```

PDP-11 的 Middle Endian 將 32-bit 值拆成兩個 16-bit word，word 之間以 Big Endian 排列，但 word 內部卻是 Little Endian。現代架構已幾乎不使用這種方式。

> **名稱由來**：Big Endian 和 Little Endian 這兩個詞其實來自 Jonathan Swift 的《格列佛遊記》——書中有兩派人為了「雞蛋應該從哪端打開」而爭戰不休。1980 年 Danny Cohen 在論文中借用這個典故，諷刺電腦工程師在 byte order 上的無謂之爭，沿用至今 [3]。

### 實驗並確認目前 Endianness

```c
#include <stdio.h>
#include <stdint.h>
#include <arpa/inet.h>

int main()
{
    uint32_t num = 0x12345678;
    uint32_t netval = htonl(num);

    uint8_t *ptr = (uint8_t *)&num;

    printf("num = %d 的 4 個位元組 (從低位址到高位址):\n", num);
    for(int i = 0; i < 4; i++) {
        printf("%d: 0x%02X (%d)\n", i, ptr[i], ptr[i]);
    }

    printf("Big endian(net): \n");

    ptr = (uint8_t *)&netval;
    for(int i = 0; i < 4; i++) {
        printf("%d: 0x%02X (%d)\n", i, ptr[i], ptr[i]);
    }

    return 0;
}
```

實驗後確認目前是採用 little-endian。

### Big Endian vs. Little Endian

| 特性                 | Big-Endian               | Little-Endian    |
| -------------------- | ------------------------ | ---------------- |
| MSByte 位置          | 低地址                   | 高地址           |
| 代表架構             | PowerPC, SPARC, 網路協議 | x86, x86-64      |
| 人類閱讀直覺         | 符合（高位在前）         | 不符合           |
| 低位元組直接存取     | 需偏移                   | 直接讀取低地址   |
| 無符號整數字典序排序 | 與數值排序一致           | 不一致           |
| 嵌入式系統彈性       | 較低                     | 較高（x86 主導） |

Little-Endian：在加法或乘法運算實作電路較簡單，處理器可以直接讀取低位元組進行進位處理，並且在轉換資料長度時，記憶體起始位置不變 [2]。

Big-Endian：進行大小比對或者是字串排序時，邏輯直觀且有效率 [2]。

不過發展至今日效能差距基本可以忽略，更多定義的是統一定義標準避免不同系統解讀資料錯誤，比如網路傳輸。

值得一提的是，**ARM** 架構支援 **Bi-Endian**。在 ARMv7 以前可透過 `SETEND` 指令切換 endianness；ARMv8（AArch64）則移除了 `SETEND`，改由系統暫存器（如 `SCTLR_EL1.EE`）在 EL 切換時控制。RISC-V 規格同樣預留了 bi-endian 的可能性，但目前幾乎所有實作皆採用 Little Endian。

### 各語言的 Byte Swap API

在跨平台或網路程式設計中，常需要手動轉換 byte order，以下是各語言的常見做法：

**C / C++**
```c
#include <stdint.h>

// GCC built-in（效率高，編譯器會優化成單一指令）
uint32_t val = __builtin_bswap32(0x12345678); // => 0x78563412

// POSIX 標準函式
#include <arpa/inet.h>
uint16_t port = ntohs(raw_port);  // network to host (16-bit)
uint32_t addr = ntohl(raw_addr);  // network to host (32-bit)
```

**Python**
```python
import struct

# struct module：明確指定 byte order
big    = struct.pack('>I', 0x12345678)   # b'\x12\x34\x56\x78'
little = struct.pack('<I', 0x12345678)   # b'\x78\x56\x34\x12'

# int.to_bytes / int.from_bytes
val = (0x12345678).to_bytes(4, byteorder='little')
back = int.from_bytes(val, byteorder='big')
```

**Rust**
```rust
let val: u32 = 0x12345678;
let be = val.to_be_bytes();  // [0x12, 0x34, 0x56, 0x78]
let le = val.to_le_bytes();  // [0x78, 0x56, 0x34, 0x12]
let swapped = val.swap_bytes(); // 0x78563412
```

### 檔案格式中的 Endianness

Endianness 不只存在於執行時期，各種檔案格式也明確規定了 byte order：

| 格式  | Endianness    | 備註                                   |
| ----- | ------------- | -------------------------------------- |
| BMP   | Little Endian | Windows 原生格式                       |
| PNG   | Big Endian    | 網路傳輸導向 [9]                        |
| WAV   | Little Endian | PCM 音訊                               |
| ELF   | 由 header 決定 | `e_ident[5]`（offset 5）：`0x01` = LE，`0x02` = BE [8] |
| JPEG  | Big Endian    | EXIF 區塊例外，由 header 標記          |
| Java Class File | Big Endian | JVM 統一使用 Big Endian           |

ELF 的做法是個好範例：在 header 裡明確標記自己的 endianness，讓 loader 能正確解讀後續的資料，而不是假設系統的預設值 [8]。

## Endianness 不同所造成的影響

不同系統所用的 Endianness 可能不相同，上面有提到這會影響到資料的解讀，以下舉幾個例子。

1. 資料解讀錯誤

假設系統 A（Big Endian）將日期 `2012 年 12 月` 編碼為 32-bit 整數 `0x20121200` 並寫入檔案，系統 B（Little Endian）直接讀取同一份檔案：

```bash
記憶體位址:  0x00   0x01   0x02   0x03
寫入 (BE):  [0x20] [0x12] [0x12] [0x00]   → 0x20121200 (正確)
讀取 (LE):  [0x20] [0x12] [0x12] [0x00]   → 0x00121220 (錯誤)
```

同樣的 byte sequence，Big Endian 解讀為 `0x20121200`（536,875,520），Little Endian 卻解讀為 `0x00121220`（1,184,288）——完全不同的數值。如果應用程式再從這個整數拆出年月，就會得到錯誤的日期。

2. 構建 buffer overflow payload

在覆蓋 return address 時，要考慮到 ISA 所預設的 Endianness。
以 x86 為例，x86 採取 Little Endian，所以 return address 應該轉換。

```python
# target => 0x12345678
payload = b'\x12\x34\x56\x78'  # 錯誤
payload = b'\x78\x56\x34\x12'  # 正確（Little Endian）
```

3. 構建網路封包

網路封包採取 Big Endian，又被稱為 Network Byte Order [7]。

情境：一個資安設備（如 Firewall）在解析 TCP Header 中的 Port Number（16-bit）[7]。

問題：網路封包使用 Big-Endian。如果防火牆開發者直接用 `uint16_t` 去讀取封包而忘了呼叫 `ntohs()` (Network to Host Short)，則封包中的 Port 80 (`0x0050`) 會被 x86 主機誤讀為 `0x5000` (20480)。

```c
#include <arpa/inet.h>
#include <stdint.h>

void parse_tcp_header(uint8_t *packet) {
    uint16_t *port_ptr = (uint16_t *)(packet + 2); // TCP destination port offset

    // 錯誤：直接讀取，x86 上 0x0050 會被解讀為 0x5000 (20480)
    uint16_t port_wrong = *port_ptr;

    // 正確：使用 ntohs() 將 Network Byte Order 轉換為 Host Byte Order
    uint16_t port_correct = ntohs(*port_ptr);
}
```

:::note[網路傳輸 vs. 檔案格式的 Endianness]
網路協議的 Big Endian（Network Byte Order）只作用於 **TCP/IP header 欄位**（Port Number、IP Address、Sequence Number 等），而 payload 是原封不動的 byte stream。因此，即使透過網路傳輸 Little Endian 格式的檔案（如 WAV、BMP），檔案內容不會被改變 byte order——接收端拿到的資料和傳送前完全一致。只有當你自訂應用層協議，在 payload 中塞入多位元組數值時，才需要自行約定並轉換 byte order。
:::

## Bit-Level Endianness

上面討論的都是 **Byte-Level**：以 byte 為單位決定排列順序。那 **Bit-Level** 呢？

Bit-level endianness 描述的是一個 byte 內部，哪個 bit 是 MSB（Most Significant Bit）、哪個是 LSB（Least Significant Bit），以及在序列傳輸時先送哪個 bit。

一般記憶體存取不涉及 bit-level endianness（CPU 總是把 byte 當作最小單位），但在以下場景中就很重要：

**序列傳輸協議**

| 協議         | Bit 傳輸順序 | 說明                               |
| ------------ | ------------ | ---------------------------------- |
| Ethernet     | LSB first    | 每個 byte 從最低位先送出 [4]       |
| USB          | LSB first    | [5]                                |
| SPI          | 可設定       | 由 SPI 控制暫存器設定，通常 MSB first |
| I²C          | MSB first    |                                    |
| CAN bus      | MSB first    | [6]                                |

**實例：Ethernet MAC Address**

Ethernet frame 中，MAC address 的每個 byte 是 LSB first 傳輸的 [4]。這代表 `0xAC`（`10101100`）在線路上會以 `00110101` 的順序傳出。這也是為什麼在解析 802.11 Wireless frame 時，multicast bit 的判斷要特別注意 bit order。

**實例：TCP Checksum 與位元欄位**

TCP/IP Header 中有許多 bit field（如 Flags: SYN, ACK, FIN）[7]，這些欄位在記憶體中的排列順序由 C struct 的 bitfield 實作決定，而 C 標準並未規定 bitfield 的 bit order，這使得跨平台解析 packet 時必須格外小心，通常建議用 bitmask 和 shift 手動操作而非依賴 bitfield struct。

```c
// 不可靠的做法（bitfield 的 bit order 由編譯器決定）
struct tcp_flags {
    uint8_t fin : 1;
    uint8_t syn : 1;
    uint8_t rst : 1;
    // ...
};

// 可靠的做法
uint8_t flags = packet[13];
int syn = (flags >> 1) & 1;
int ack = (flags >> 4) & 1;
```

總結來說，byte-level endianness 影響記憶體中多位元組資料的排列，而 bit-level endianness 影響序列傳輸中每個 byte 內的 bit 傳送順序——兩者是獨立的概念，在網路協議或硬體介面開發時都需要同時考慮。

## Reference

1. [Understanding computer endianness](https://manistein.github.io/blog/post/common/understanding_computer_endianness/)
2. [Computer Systems: A Programmer's Perspective](https://csapp.cs.cmu.edu/) (3rd ed., Pearson, 2016) §2.1.3 Addressing and Byte Ordering
3. Danny Cohen, ["On Holy Wars and a Plea for Peace"](https://www.rfc-editor.org/ien/ien137.txt), IEN 137, USC/ISI, 1980; 後刊載於 IEEE Computer, Vol. 14, No. 10, October 1981
4. [IEEE 802.3-2018](https://standards.ieee.org/ieee/802.3/7071/) — Bit transmission order (Ethernet LSB first)
5. [USB 2.0 Specification](https://www.usb.org/document-library/usb-20-specification) — Bit ordering (LSB first)
6. [ISO 11898-1:2015](https://www.iso.org/standard/63648.html) CAN Data Link Layer — bit-level transmission order (MSB first)
7. [RFC 9293](https://www.rfc-editor.org/rfc/rfc9293) Transmission Control Protocol (TCP) — TCP Header Format & Flags（取代原 [RFC 793](https://www.rfc-editor.org/rfc/rfc793), 1981）
8. [ELF-64 Object File Format](https://refspecs.linuxfoundation.org/elf/gabi4+/ch4.eheader.html) — ELF Identification (`EI_DATA` field)
9. [PNG Specification 1.2 — Section 2.1: Integers and byte order (Big Endian)](https://libpng.org/pub/png/spec/1.2/png-1.2.pdf)
