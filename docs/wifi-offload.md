# Wi-Fi offload on the EX530v (EN7523 + MT7916)

## What the stock firmware does

Stock loads, in this order: `hw_nat.ko`, `npu.ko` (+ `/userfs/npu_rv32.bin`, `npu_data.bin`),
`hostadpt.ko`, `mt_whnat.ko`, then `mt7916_ap.ko` with `WHNAT=1`. `config.sdk` has
`TCSUPPORT_NPU_WIFI_OFFLOAD=y`.

| Direction | Stock path |
|---|---|
| Wi-Fi -> WAN (upload) | MT7916 RX ring -> **NPU** (RXD ring, block-ack reorder in DRAM) -> QDMA -> PPE -> GDM |
| WAN -> Wi-Fi (download) | GDM -> PPE (bound, `dp=0`, NBQ = QDMA RX ring 5) -> CPU `PpeWifiFastTxHandler` (+ `sw_rps_for_wifi.ko`) -> `mt7916_ap` -> **WED/WHNAT TX ring** -> PCIe |

Download, from `hw_nat.ko`: `PpeTxHandler` -> `PpeSetForcePortInfo` -> `isWiFiInterface` ->
`PpeSetForcePortInfoOffload`. For skbs with FOE magic `0x7274` (to WLAN) the entry gets
`PpeSetInfoBlk2(dp=0)` and `PpeSetRxForceRingIdx(5)` (6 without NPU offload). So the PPE
returns bound Wi-Fi flows to the CPU, and the CPU hands them to the Wi-Fi driver. The
stock firmware does **not** bind Wi-Fi flows PPE -> WDMA.

## What this port does

| Direction | OpenWrt path |
|---|---|
| Wi-Fi -> WAN | MT7916 RX ring -> **NPU** (OpenNPU KITE firmware) -> QDMA0 TX ring 6 -> PPE -> GDM |
| WAN -> Wi-Fi | GDM -> PPE (bound, CDM1, `CRSN_22`) -> `airoha_eth` fast path -> mt76 -> **WED TX ring** |

The fast path queues the already NATed and L2-rewritten frame straight to the Wi-Fi device
(the bridge, which picks the port from its FDB). It skips GRO, routing and netfilter, and
reassembles frames that arrive split over two RX buffers.

Measured: 2 GB from a server on the WAN side to a phone on 5 GHz (HE160) at 58-70 MB/s,
both ARM cores at about 65-75 %, none at 100 %.

## NPU bring-up notes

- `en7523.dtsi` has `compatible = "airoha,en7523-npu"`. Since the fork's patch 930-228 the
  driver matches `airoha,airoha_en7523-npu`, so the NPU never bound. The board DTS
  overrides the compatible.
- The firmware is OpenNPU (`airoha-en7523-kite-npu-firmware`, built from source):
  `airoha/en7523_kite_npu_rv32.bin` + `_data.bin`. It identifies as
  `OpenNPU v8.0.0_KITE` on the NPU UART log.
- The mt76 NPU patch (`0301`) needs `enum airoha_npu_wlan_chip`
  (`NONE=0, MT7915, MT7915_DBDC, MT7916`) and the 4-argument `check_skb_reason`
  (mt76 `0302`).
- OpenNPU uses identity command IDs; the EN7523 table blocked `WLAN_FUNC_SET_WAIT_HWNAT_INIT`
  (`-EOPNOTSUPP`, mt7915e probe failed with -95). Kernel `9993`.
- The firmware takes over **QDMA0 TX ring 6 and TX IRQ queue 1** (it checks that both have
  1024 entries and are idle). `airoha_eth` must never queue to, reap from or reset them, and
  must not stop the QDMA engine on `ifdown`. Without this: `QDMA DMA engine busy timeout`
  and Ethernet TX stops. Kernel `9994`.
- With the NPU running, `wed_enable=0` leaves the APs beaconing but deaf (no association).
  Keep `options mt7915e wed_enable=1`; `uci-defaults/90-ex530v-wed` fixes configs kept from
  older images.
- With WED attached, the netfilter forward path stops at `br-lan`; the PPE treats a bridge
  with a Wi-Fi port as a Wi-Fi destination. PPE -> WDMA binding (`ib2` PSE port 3 + WDMA
  info) delivers nothing on this SoC; it is behind `airoha_eth.en7523_wdma_bind=1`, off
  by default. Kernel `9995`.

## Debugging

```sh
cat /sys/module/airoha_eth/parameters/en7523_wifi_fp_*   # fast path counters
echo 2 > /sys/module/airoha_eth/parameters/en7523_wifi_fastpath   # verbose
cat /sys/kernel/debug/airoha_npu/{status,rings,counters}
cat /sys/kernel/debug/ieee80211/phy0/mt76/npu_rx            # NPU RX stats (both bands)
grep npu-rx /proc/interrupts
cat /sys/kernel/debug/ppe/bind                              # bound PPE entries
cat /sys/kernel/debug/wed1/txinfo                           # WED TX ring indices
```
