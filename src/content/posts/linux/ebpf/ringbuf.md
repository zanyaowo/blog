---
title: 從source code理解ring buffer的運作原理
published: 2026-04-11
description: "深入探討香農熵 (Shannon Entropy) 的定義與其在資訊增益 (Information Gain) 中的應用"
pinned: false
tags: [entropy, information-gain, machine-learning, information-theory]
category: Computer Science
draft: true
---

## 前言
在 ebpf 中 ring buffer 讓 kernel 以高效的方式傳遞資料，但齊背後原理是如何實現的這是我所好奇的

source: [ringbuf.c](https://github.com/torvalds/linux/blob/master/kernel/bpf/ringbuf.c#L28)

## ring buffer 所定義的 class

### bpf_ringbuf

#### 為什麼會有兩種 lock

> This is done because the ring buffer must hold a lock across a BPF program's callback

在 user-space producer 的情境下，若使用 `spinlock` 跨越整個 eBPF callback 期間持鎖，由於 callback 執行時間可能很長，會造成其他任務長時間等待，這是不合理的行為。

kernel producer:
reserve -> 寫入資料 -> commit

user-space producer:

```
user-space 寫資料到 ring buffer
    ↓
kernel __bpf_user_ringbuf_peek() 找到這筆資料
    ↓
kernel 呼叫 BPF callback 程式來處理這筆資料  ← 問題在這裡
    ↓
處理完畢，推進 consumer_pos
```

因此改用 `atomic_t busy` 搭配 return `-EBUSY` 的方式：當 `__bpf_user_ringbuf_peek()` 發現 busy 已被設置，直接回傳錯誤讓呼叫方自行重試，避免持鎖等待。

| lock 類型 | 適用情境 | 機制 |
|---|---|---|
| `atomic_t busy` | user-space producer | 非阻塞，透過 return 值判斷是否佔用 |
| `spinlock` | kernel 內部操作 | 阻塞式，保護 IRQ context 下的臨界區 |

---

#### consumer / producer 的 page 權限設計

> producer 的角色分為 kernel 和 user-space 兩種

`consumer_pos` 和 `producer_pos` 被放在不同的 page，目的是讓每個位置能以不同權限映射，防止 user-space 竄改位置進而破壞 kernel 內部的追蹤狀態。

page 的權限依照誰是 producer 而有所不同：

- **kernel producer**：producer 的位置頁面與資料頁面在 user-space 中映射為 read-only。kernel 透過樣本 header 的 bit flags 通知 user-space 及其他 producer 該筆資料是否正在寫入中。

- **user-space producer**：只有存放 `consumer_pos` 的頁面在 user-space 中映射為 read-only。user-space 同樣透過 header 的 bit flags 與 kernel 溝通，但 kernel 必須主動驗證每筆樣本的格式是否正確，且確保資料完全包含在 ring buffer 範圍內。

> **NOTE**：`pending_pos` 與 `producer_pos` 共享同一條 cache line。

**Q：為什麼 user-space producer 模式下只有 `consumer_pos` 需要 read-only？**

因為誰負責推進哪個指標，誰就擁有那個指標的寫入權。user-space 作為 producer 需要寫入 `producer_pos`，而 `consumer_pos` 由 kernel 管理——若 user-space 能任意修改它，等同於偽造「已讀取」的狀態，覆蓋 kernel 尚未讀取的資料，破壞追蹤狀態。

### 欄位解析

```c
struct bpf_ringbuf {
	wait_queue_head_t waitq;
	struct irq_work work;
	u64 mask;
	struct page **pages;
	int nr_pages;
	bool overwrite_mode;
	rqspinlock_t spinlock ____cacheline_aligned_in_smp;
	atomic_t busy ____cacheline_aligned_in_smp;
	unsigned long consumer_pos __aligned(PAGE_SIZE);
	unsigned long producer_pos __aligned(PAGE_SIZE);
	unsigned long pending_pos;
	unsigned long overwrite_pos; /* position after the last overwritten record */
	char data[] __aligned(PAGE_SIZE);
};
```

#### `wait_queue_head_t waitq`

出現在以下函式：

| 函式 | 用途 |
|---|---|
| `bpf_ringbuf_alloc()` | 初始化 `waitq` |
| `bpf_ringbuf_notify()` | 呼叫 `wake_up_all()` 喚醒佇列中所有等待者 |
| `ringbuf_map_poll_kern()` | 將 file pointer（`filp`）掛入 `waitq`，供 poll/epoll 監聽 |
| `ringbuf_map_poll_user()` | 同上，user-space producer 情境 |

`waitq` 是一個等待佇列，consumer 在沒有資料時會在此等待；當 producer 提交新資料後，透過 `bpf_ringbuf_notify()` 呼叫 `wake_up_all()` 將其喚醒。


### `struct irq_work work;`

出現在以下函式：
bpf_ringbuf_notify => 用巨集 container_of 反推 bpf_ringbuf struct pointer 位置
bpf_ringbuf  => 初始化
bpf_ringbuf_free
bpf_ringbuf_commit
BPF_CALL_4