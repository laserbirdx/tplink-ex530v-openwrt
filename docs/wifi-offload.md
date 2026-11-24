# Wi-Fi offload on the EX530v (EN7523 + MT7916)

## What the stock firmware does

Stock loads, in this order: `hw_nat.ko`, `npu.ko` (+ `/userfs/npu_rv32.bin`, `npu_data.bin`),
`hostadpt.ko`, `mt_whnat.ko`, then `mt7916_ap.ko` with `WHNAT=1`. `config.sdk` has
`TCSUPPORT_NPU_WIFI_OFFLOAD=y`.

| Direction | Stock path |
|---|---|
| Wi-Fi -> WAN (upload) | MT7916 RX ring -> **NPU** (RXD ring, block-ack reorder in DRAM) -> QDMA -> PPE -> GDM |
| WAN -> Wi-Fi (download) | GDM -> **PPE -> WDMA -> WED (WHNAT) -> PCIe -> MT7916**, no CPU |

The first packets of a flow still go through the CPU. When the Wi-Fi driver transmits one,
`mt_whnat.ko` `wifi_tx_tuple_add()` calls `ra_sw_nat_hook_tx(skb, info, 0x7615)` with the
WHNAT magic `0x7615`. `hw_nat.ko` `PpeSetPortInfo()` then writes the FOE entry (decoded with
`objdump` from the stock modules):

| FOE field | Value |
|---|---|
| `ib2` PSE port | 3 |
| `ib2` PSE_QOS | 1 |
| `ib2` NBQ | `ring` = `band \| wed_index << 1` |
| `data` CHANNEL (bits 15:11) | `6 + ring` |
| `data` SHAPER_ID (bits 23:16) | `0x7f` |
| `l2.etype` (WINFO) | `bss \| wcid << 6` |

Flows the stock firmware does not hand to WHNAT (magic `0x7274`, plain WLAN) are bound with
`dp=0` and an RX ring (5/6) and forwarded by the CPU in `PpeWifiFastTxHandler`.

## What this port does

| Direction | OpenWrt path |
|---|---|
| Wi-Fi -> WAN | MT7916 RX ring -> **NPU** (OpenNPU KITE firmware) -> QDMA0 TX ring 6 -> PPE -> GDM |
| WAN -> Wi-Fi | GDM -> **PPE -> WDMA -> WED -> MT7916** with the stock FOE layout above (default) |
| WAN -> Wi-Fi, fallback | PPE -> CDM1 -> `airoha_eth` fast path -> mt76 -> WED TX ring (`en7523_wdma_bind=0`) |

The mainline/fork driver bound Wi-Fi flows to PSE port 3, but put the WED index in NBQ and
left CHANNEL and SHAPER at 0 with a different WINFO format. WDMA never received a frame.

### Measured (2 GB, server on the WAN side, client on 5 GHz HE80/160)

| Downstream mode | Throughput | CPU |
|---|---|---|
| PPE -> WDMA -> WED (default) | 92-106 MB/s | both cores 6-12 % |
| CDM1 fast path | 57-67 MB/s | both cores ~70 % |
| software forwarding | ~60 MB/s | one core 100 % |

During a WDMA download the WED WDMA RX ring indices (`0x918`/`0x91c` in `wed1`) cycle while
the fast path and software TX counters stay at zero.

## NPU bring-up notes

- `en7523.dtsi` has `compatible = "airoha,en7523-npu"`. Since the fork's patch 930-228 the
  driver matches `airoha,airoha_en7523-npu`, so the NPU never bound. The board DTS
  overrides the compatible.
- The firmware is OpenNPU (`airoha-en7523-kite-npu-firmware`, built from source):
  `airoha/en7523_kite_npu_rv32.bin` + `_data.bin` (`OpenNPU v8.0.0_KITE` on the NPU UART).
- The mt76 NPU patch (`0301`) needs `enum airoha_npu_wlan_chip`
  (`NONE=0, MT7915, MT7915_DBDC, MT7916`) and the 4-argument `check_skb_reason`
  (mt76 `0302`).
- OpenNPU uses identity command IDs; the EN7523 table blocked `WLAN_FUNC_SET_WAIT_HWNAT_INIT`
  (`-EOPNOTSUPP`, mt7915e probe failed with -95). Kernel `9993`.
- The firmware takes over **QDMA0 TX ring 6 and TX IRQ queue 1**. `airoha_eth` must never
  queue to, reap from or reset them, and must not stop the QDMA engine on `ifdown`.
  Without this: `QDMA DMA engine busy timeout` and Ethernet TX stops. Kernel `9994`.
- With the NPU running, `wed_enable=0` leaves the APs beaconing but deaf. Keep
  `options mt7915e wed_enable=1`; `uci-defaults/90-ex530v-wed` fixes configs kept from older
  images.
- With WED attached, the netfilter forward path can stop at `br-lan`; the PPE resolves the
  Wi-Fi port itself. Kernel `9995` (also the stock WHNAT FOE layout).

## Knobs and debugging

```sh
P=/sys/module/airoha_eth/parameters
cat $P/en7523_wdma_bind            # 1 = PPE -> WDMA -> WED (default), 0 = CDM1 fast path
ls  $P/en7523_wdma_*               # port, qos, nbq_base, chan_base, shaper, winfo (stock defaults)
cat $P/en7523_wifi_fp_*            # fast path counters
echo 2 > $P/en7523_wifi_fastpath   # log flow binds (wdma bind: ib2/data/winfo)
cat /sys/kernel/debug/ppe/bind     # bound PPE entries (Wi-Fi: ib2=...263, data=007f4800-ish)
cat /sys/kernel/debug/airoha_npu/{status,rings,counters}
cat /sys/kernel/debug/ieee80211/phy0/mt76/npu_rx
cat /sys/kernel/debug/wed1/txinfo
```

Read a WED register: `echo $((0x918)) > /sys/kernel/debug/wed1/regidx; cat /sys/kernel/debug/wed1/regval`.
