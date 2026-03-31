# mac80211 Subsystem Specification

This document describes the architecture, state machines, and data-plane
processing of the mac80211 stack located under `net/mac80211/` with its public
API defined in `include/net/mac80211.h`.

---

## 1. Framework Overview

### 1.1 Layering

```
 Userspace  (wpa_supplicant / hostapd / iw)
      |  nl80211
 cfg80211   (regulatory, nl80211 command dispatch)
      |
 mac80211   (MLME, TX/RX pipelines, aggregation, crypto, rate control)
      |  ieee80211_ops callbacks
 Driver     (hardware-specific implementation)
      |
 Firmware / Hardware
```

mac80211 is an **MLME mid-layer**: it implements 802.11 management logic
(authentication, association, beacon handling, aggregation handshake) and
provides a full TX/RX processing pipeline. Drivers register a
`struct ieee80211_hw` and a table of `struct ieee80211_ops` callbacks.

### 1.2 Key Source Files

| Area | Files | Purpose |
|------|-------|---------|
| **Core** | `main.c`, `iface.c`, `link.c` | HW registration, netdev lifecycle, MLD links |
| **Driver glue** | `driver-ops.c`, `driver-ops.h` | Wrappers that add tracing/locking around every driver callback |
| **Station mgmt** | `sta_info.c`, `sta_info.h` | STA hash table, state machine, power-save buffering |
| **MLME** | `mlme.c`, `ibss.c`, `ocb.c`, `tdls.c` | Client-side authentication/association, IBSS, OCB, TDLS |
| **TX path** | `tx.c`, `status.c`, `wme.c`, `airtime.c` | TX handler chain, TX status, WMM classification, airtime fairness |
| **RX path** | `rx.c`, `rate.c` | RX handler chain, rate control interface |
| **Aggregation** | `agg-tx.c`, `agg-rx.c` | A-MPDU TX/RX session management (ADDBA/DELBA) |
| **Scanning** | `scan.c`, `offchannel.c` | HW/SW scan state machines, remain-on-channel |
| **Config** | `cfg.c`, `chan.c` | nl80211 handlers, channel context management |
| **Crypto** | `key.c`, `wep.c`, `tkip.c`, `wpa.c`, `aes_cmac.c`, `aes_gmac.c`, `aead_api.c`, `fils_aead.c` | Key management, per-cipher encrypt/decrypt |
| **PHY generation** | `ht.c`, `vht.c`, `he.c`, `eht.c`, `uhr.c`, `s1g.c` | 802.11n / ac / ax / be / UHR / ah capability handling |
| **Mesh** | `mesh.c`, `mesh_plink.c`, `mesh_hwmp.c`, `mesh_pathtbl.c`, `mesh_ps.c`, `mesh_sync.c` | Mesh peering, HWMP routing, path table, mesh PS |
| **Rate control** | `rc80211_minstrel_ht.c` | Minstrel-HT rate control algorithm |
| **Power** | `pm.c` | Suspend/resume, WoWLAN |
| **Tests** | `tests/` | KUnit tests (chan-mode, elems, mfp, s1g_tim, tpe) |

### 1.3 Core Data Structures

| Structure | Location | Role |
|-----------|----------|------|
| `ieee80211_hw` | `mac80211.h:3107` | Hardware descriptor (one per PHY). Contains `wiphy`, `conf`, capability flags, queue count. Driver private data appended at the end. |
| `ieee80211_local` | `ieee80211_i.h` | mac80211 private extension of `ieee80211_hw`. Holds STA hash tables, work queues, TXQ scheduler state, interface list, scan state, and all runtime flags. |
| `ieee80211_ops` | `mac80211.h:4562` | ~90 driver callback function pointers. See Section 1.4. |
| `ieee80211_vif` | `mac80211.h:2080` | Per-virtual-interface data: type, BSS config, MLD link bitmaps, hardware queue map, driver private area. |
| `ieee80211_sub_if_data` | `ieee80211_i.h` | mac80211 private extension of `ieee80211_vif`. Contains interface-type-specific unions (managed, AP, mesh, IBSS, etc.), crypto keys, fragment cache, management queue. |
| `ieee80211_sta` | `mac80211.h:2567` | Public station descriptor (peer). MLD address, AID, WME, MFP flags, per-TID TXQs, PHY capability structs (HT/VHT/HE/EHT), MLD link array, driver private area. |
| `sta_info` | `sta_info.h` | mac80211 private extension of `ieee80211_sta`. Hash linkage, PTK keys, rate control state, fast-path caches, PS buffers, aggregation MLME, per-link data, airtime accounting. |
| `ieee80211_bss_conf` | `mac80211.h:765` | BSS parameters: BSSID, DTIM period, beacon interval, channel, TX power, ERP/HT/VHT/HE settings, BSS color. |
| `ieee80211_tx_info` | `mac80211.h:1248` | Stored in `skb->cb` for TX. Control rates, retry info, hardware key, VIF pointer. Re-used for TX status on completion. |
| `ieee80211_rx_status` | `mac80211.h:1663` | Stored in `skb->cb` for RX. Band, frequency, signal, rate/MCS, encoding (Legacy/HT/VHT/HE/EHT/UHR), flags. |
| `ieee80211_key_conf` | `mac80211.h:2285` | Key configuration: cipher, IV/ICV lengths, key index, hardware key index, key material. |
| `ieee80211_conf` | `mac80211.h:1820` | Device-level configuration: channel, power, PS, SMPS. |

### 1.4 Driver Callback Interface (`ieee80211_ops`)

Drivers implement a subset of these callbacks (8 are mandatory: `tx`, `start`,
`stop`, `config`, `add_interface`, `remove_interface`, `configure_filter`,
`wake_tx_queue`).

| Category | Callbacks | Context |
|----------|-----------|---------|
| **Lifecycle** | `start`, `stop`, `suspend`, `resume` | may sleep |
| **Interface** | `add_interface`, `remove_interface`, `change_interface`, `prep_add_interface`, `update_vif_offload` | may sleep |
| **Configuration** | `config`, `bss_info_changed`, `vif_cfg_changed`, `link_info_changed`, `conf_tx` | may sleep |
| **AP** | `start_ap`, `stop_ap`, `set_tim`, `release_buffered_frames`, `allow_buffered_frames` | mixed |
| **Station** | `sta_add`, `sta_remove`, `sta_state`, `sta_pre_rcu_remove`, `sta_notify`, `sta_set_txpwr`, `sta_statistics`, `link_sta_rc_update` | mixed |
| **Scan** | `hw_scan`, `cancel_hw_scan`, `sched_scan_start`, `sched_scan_stop`, `sw_scan_start`, `sw_scan_complete` | mixed |
| **Channel** | `add_chanctx`, `remove_chanctx`, `change_chanctx`, `assign_vif_chanctx`, `unassign_vif_chanctx`, `switch_vif_chanctx` | may sleep |
| **Filtering** | `prepare_multicast`, `configure_filter`, `config_iface_filter` | mixed |
| **Crypto** | `set_key`, `update_tkip_key`, `set_rekey_data`, `set_default_unicast_key` | mixed |
| **Aggregation** | `ampdu_action` (RX_START/STOP, TX_START/STOP_*, TX_OPERATIONAL) | mixed |
| **TX queue** | `wake_tx_queue`, `sync_rx_queues` | atomic |
| **Channel switch** | `channel_switch`, `pre/post_channel_switch`, `abort_channel_switch` | mixed |
| **Off-channel** | `remain_on_channel`, `cancel_remain_on_channel` | mixed |
| **MLO** | `can_activate_links`, `change_vif_links`, `change_sta_links`, `can_neg_ttlm`, `set_eml_op_mode` | may sleep |
| **TDLS** | `tdls_channel_switch`, `tdls_cancel_channel_switch`, `tdls_recv_channel_switch` | mixed |
| **TWT** | `add_twt_setup`, `twt_teardown_request` | mixed |
| **Misc** | `flush`, `flush_sta`, `get_tsf`, `set_tsf`, `set_antenna`, `get_survey`, `rfkill_poll`, `event_callback`, `set_sar_specs` | mixed |

All callbacks are invoked through `drv_*()` wrappers (`driver-ops.c/h`) that
add `might_sleep()` / `lockdep_assert_wiphy()` assertions, tracepoints, and
legacy-API compatibility shims.

### 1.5 Hardware Registration Flow

```
Driver probe
  |-> ieee80211_alloc_hw()         allocate wiphy + ieee80211_local + drv_priv
  |-> (driver sets hw->flags, queues, etc.)
  |-> ieee80211_register_hw()      validate bands/caps, init rate control,
  |                                  create workqueue, register wiphy with cfg80211
  |
  |-> ieee80211_add_interface()    (may be automatic, creating wlan0 STA)
       |-> drv_start()             first interface triggers HW enable
       |-> drv_add_interface()     notify driver, set SDATA_IN_DRIVER flag
```

### 1.6 Interface Lifecycle

```
 Created: ieee80211_if_add()          alloc netdev, type-specific init
    |
 Opened:  ieee80211_do_open()         drv_start() (first), drv_add_interface(),
    |                                  configure_filter, set up WMM
    |
 Running: ndo_start_xmit / iface_work / ...
    |
 Stopped: ieee80211_do_stop()         flush STAs, cancel scan, teardown BA,
    |                                  drv_remove_interface(), drv_stop() (last)
    |
 Removed: ieee80211_if_remove()       unregister netdev
```

Interface types: `managed` (STA), `AP`, `AP_VLAN`, `mesh`, `IBSS`, `monitor`,
`P2P_*`, `NAN`, `OCB`.

---

## 2. State Machines

### 2.1 Station State Machine

Defined by `enum ieee80211_sta_state` (`mac80211.h:2365`). Each station
(`struct sta_info`) transitions through these ordered states:

```
  NOTEXIST ──> NONE ──> AUTH ──> ASSOC ──> AUTHORIZED
                                                   |
                      <─────────────────────────────
                      (reverse transitions allowed)
```

| Transition | Trigger (AP side) | Trigger (STA side) | Driver notified? |
|------------|-------------------|--------------------|------------------|
| NOTEXIST -> NONE | `sta_info_alloc()` + `sta_info_insert_rcu()` | `ieee80211_prep_connection()` | Yes (`drv_sta_state`) |
| NONE -> AUTH | Received auth frame | `ieee80211_rx_mgmt_auth()` success | Yes |
| AUTH -> ASSOC | Received assoc request, accepted | `ieee80211_set_associated()` | Yes |
| ASSOC -> AUTHORIZED | 802.1X / upper-layer authorization | cfg80211 call after key install | Yes |
| AUTHORIZED -> ASSOC | De-authorization | Upper-layer disconnection | Yes (must not fail) |
| ASSOC -> AUTH | Disassociation | `ieee80211_set_disassoc()` | Yes (must not fail) |
| AUTH -> NONE | De-authentication | `ieee80211_set_disassoc()` | Yes (must not fail) |

**Transition rules** (`_sta_info_move_state()`):

- Up-transitions: driver is notified **before** updating internal flags. Driver
  may veto by returning an error.
- Down-transitions: internal flags are updated **before** notifying driver.
  Driver must not fail down-transitions.
- Side effects:
  - `ASSOC -> AUTHORIZED`: increment multicast counter, build fast TX/RX caches,
    send layer-2 update frame (AP mode).
  - `AUTHORIZED -> ASSOC`: decrement multicast counter, flush queues, clear
    fast TX/RX caches.

### 2.2 MLME Connection State Machine (STA mode)

Driven by `ieee80211_sta_work()` in `mlme.c`. This is the client-side
connection management state machine:

```
  Disconnected
       |
       v
  ieee80211_prep_connection()     allocate STA, set up channel context
       |
       v
  IEEE80211_STA_NONE
       |
       v  ieee80211_auth()
  Authenticating ──(timeout)──> retry (max 3, AUTH_MAX_TRIES)
       |
       |  RX auth response (success)
       v
  IEEE80211_STA_AUTH
       |
       v  ieee80211_do_assoc() / ieee80211_send_assoc()
  Associating ──(timeout)──> retry (max 3, ASSOC_MAX_TRIES)
       |
       |  RX assoc response (success)
       v
  IEEE80211_STA_ASSOC            ieee80211_set_associated():
       |                           configure BSS params, notify driver,
       |                           start beacon monitoring
       |
       |  wpa_supplicant installs keys
       v
  IEEE80211_STA_AUTHORIZED        traffic flows

  Connection loss at any point:
       |  beacon loss (7 consecutive missed)
       |  deauth/disassoc received
       |  internal error
       v
  ieee80211_set_disassoc()       move STA down to NONE,
                                  cfg80211 connection loss notification
```

**Key timers:**

| Timer | Value | Purpose |
|-------|-------|---------|
| `AUTH_TIMEOUT` | 200 ms (`HZ/5`) | Wait for authentication response |
| `ASSOC_TIMEOUT` | 200 ms (`HZ/5`) | Wait for association response |
| `beacon_timeout` | `beacon_int * DTIM_period * beacon_loss_count` | Detect beacon loss (default: 7 missed) |
| `dynamic_ps_timer` | configurable | Enter PS after TX idle |
| `conn_idle_timer` | 30 s | Detect idle connection |

### 2.3 AP Mode and the Unified Station State Machine

There is **no separate AP-mode MLME connection state machine**. Mac80211 uses a
single, unified per-peer state machine (`enum ieee80211_sta_state`) shared across
all interface modes (STA, AP, mesh, IBSS). Each connected client (from the AP's
perspective) is tracked as a `struct sta_info` with the same five ordered states:

```
NOTEXIST → NONE → AUTH → ASSOC → AUTHORIZED
```

The critical architectural difference is **who drives the transitions**:

| Mode | Who drives transitions | Mechanism |
|------|----------------------|-----------|
| **STA** | mac80211 internally | `ieee80211_sta_work()` in `mlme.c` manages auth/assoc timeouts, retries, beacon monitoring |
| **AP** | Userspace (hostapd) | nl80211 station flags (`NL80211_STA_FLAG_AUTHENTICATED/ASSOCIATED/AUTHORIZED`) processed by `sta_apply_auth_flags()` in `cfg.c` |
| **Mesh** | mac80211 mesh FSM | 7-state `enum nl80211_plink_state` peer link FSM in `mesh_plink.c`, ultimately transitions `sta_state` |
| **IBSS** | mac80211 IBSS logic | 2-state MLME (`SEARCH`/`JOINED`) in `ieee80211_i.h` |

In STA mode, mac80211 **owns** the connection logic (authentication, association,
timeouts, retries, beacon loss detection). In AP mode, mac80211 acts as a
**pass-through**: the userspace daemon decides when to authenticate, associate,
and authorize each client, then instructs mac80211 to move the station's state.

The AP interface structure itself (`struct ieee80211_if_ap` in `ieee80211_i.h`)
is minimal — just `bool active` plus VLAN and power-save buffering data. All
meaningful connection state lives in the per-client `struct sta_info` entries.

**Other MLME-related state machines in mac80211:**

| State Machine | Enum / States | Key File |
|---------------|---------------|----------|
| Station state (shared STA+AP) | `ieee80211_sta_state`: NOTEXIST, NONE, AUTH, ASSOC, AUTHORIZED | `include/net/mac80211.h` |
| STA connection (implicit) | `auth_data`/`assoc_data` non-NULL + `associated` flag | `net/mac80211/mlme.c` |
| Mesh peer link | `nl80211_plink_state`: LISTEN, OPN_SNT, OPN_RCVD, CNF_RCVD, ESTAB, HOLDING, BLOCKED | `include/uapi/linux/nl80211.h` |
| IBSS MLME | SEARCH, JOINED | `net/mac80211/ieee80211_i.h` |
| Scan | DECISION, SET_CHANNEL, SEND_PROBE, SUSPEND, RESUME, ABORT | `net/mac80211/ieee80211_i.h` |
| A-MPDU aggregation | HT_AGG_STATE_* bit flags | `net/mac80211/sta_info.h` |

### 2.4 TX Aggregation Session State Machine

Managed per-TID via `struct tid_ampdu_tx`. State is tracked as a bitmap
(`HT_AGG_STATE_*`):

```
                 ieee80211_start_tx_ba_session()
  (none) ─────────────────────────────────────────> WANT_START
                         |
                         v  ieee80211_tx_ba_session_handle_start()
                      drv_ampdu_action(TX_START)
                      send ADDBA Request
                         |
                         v  (timer: ADDBA_RESP_INTERVAL)
                      WAITING_FOR_RESP
                         |
                         |  RX ADDBA Response (success)
                         v  ieee80211_process_addba_resp()
                      drv_ampdu_action(TX_OPERATIONAL)
                      RESPONSE_RECEIVED + DRV_READY + OPERATIONAL
                      start TXQ via ieee80211_agg_start_txq()
                         |
                         v
                      OPERATIONAL (normal A-MPDU TX)

  Teardown at any point:
      __ieee80211_stop_tx_ba_session()
        -> set STOPPING, clear OPERATIONAL
        -> drv_ampdu_action(TX_STOP_CONT / TX_STOP_FLUSH)
        -> splice pending frames back to local->pending[]
        -> free tid_ampdu_tx via RCU
```

### 2.5 RX Aggregation Session State Machine

Managed per-TID via `struct tid_ampdu_rx`. Simpler than TX:

```
  (none) ──> RX ADDBA Request ──> __ieee80211_start_rx_ba_session()
                                       |
                                       v  allocate reorder_buf[]
                                    drv_ampdu_action(RX_START)
                                    send ADDBA Response
                                       |
                                       v
                                    OPERATIONAL (reorder active)
                                       |
                                       |  timeout / DELBA / session teardown
                                       v  __ieee80211_stop_rx_ba_session()
                                    drv_ampdu_action(RX_STOP)
                                    send DELBA, free reorder_buf via RCU
```

Reorder buffer: `tid_agg_rx->reorder_buf[buf_size]` holds out-of-order MPDUs.
`head_seq_num` tracks next expected sequence. Direct-pass optimization: if
the received MPDU is at `head_seq_num` and the buffer is empty, it passes
through without buffering. Timeout for gaps: `HT_RX_REORDER_BUF_TIMEOUT` = 100 ms.

### 2.6 Interface State

```
  Created  (ieee80211_if_add)     SDATA_IN_DRIVER = false
     |
  Opened   (ieee80211_do_open)    drv_start (first if), drv_add_interface
     |                              SDATA_IN_DRIVER = true
  Running
     |
  Stopped  (ieee80211_do_stop)    drv_remove_interface -> SDATA_IN_DRIVER = false
     |                              drv_stop (last if)
  Removed (netdev unregistered)
```

### 2.7 Scan State Machine (`scan.c`)

```
  IDLE
    |-> start_scan() / hw_scan()
    v
  SCANNING
    |-> scan complete callback
    |-> or: cancel_hw_scan() / sw_scan_complete()
    v
  IDLE
```

Hardware scan is offloaded to the driver (`hw_scan`). Software scan is
orchestrated by mac80211 with `sw_scan_start/complete` notifications.

---

## 3. TX Packet Processing Flow

### 3.1 Overview

```
  Network stack (802.3 frame)
       |
       v
  ieee80211_subif_start_xmit()        ndo_start_xmit
       |
       v
  __ieee80211_tx_prepare()            build ieee80211_tx_data, look up STA
       |
       v
  invoke_tx_handlers_early()          PS buffering, key selection
       |
       v
  TXQ scheduling (fq + CoDel AQM)    airtime fairness, per-TID queues
       |  or direct queue if no TXQ
       v
  drv_wake_tx_queue()                 notify driver frames are ready
       |
       v
  invoke_tx_handlers_late()           rate control, sequence number, encrypt
       |
       v
  __ieee80211_tx()
       |-> ieee80211_tx_frags()
       |-> drv_tx()                   driver callback: local->ops->tx(skb)
       |
       v
  Hardware
       |
       v
  ieee80211_tx_status_ext()           TX completion (ACK / timeout)
       |-> rate_control_tx_status()   feedback to rate algorithm
       |-> ieee80211_handle_filtered_frame()  if filtered
       |-> BA window tracking
       v
  Done
```

### 3.2 Entry Points

| Entry | When | Path |
|-------|------|------|
| `ieee80211_subif_start_xmit()` | Data frame from network stack | Full TX pipeline |
| `ieee80211_subif_start_xmit_8023()` | Data frame, encap offload enabled | Bypasses 802.11 header construction |
| `ieee80211_monitor_start_xmit()` | Monitor mode injection | Radiotap header parsing |
| `__ieee80211_xmit_fast()` | Fast TX path (cached `fast_tx` descriptor) | Bypasses handler chain |
| `ieee80211_tx_skb()` / `ieee80211_tx_skb_tid()` | Management frames, internal use | Direct handler chain |

### 3.3 TX Handler Chain

Handlers execute in two phases split by TXQ queuing.

**Phase 1 -- Early handlers** (`invoke_tx_handlers_early()`):

| # | Handler | Purpose |
|---|---------|---------|
| 1 | `tx_h_dynamic_ps` | Exit power save on TX, arm dynamic PS timer |
| 2 | `tx_h_check_assoc` | Drop frames to unassociated stations; drop multicast when no STAs |
| 3 | `tx_h_ps_buf` | Buffer unicast in `sta->ps_tx_buf[ac]` if STA asleep; buffer multicast in `ps->bc_buf` for DTIM |
| 4 | `tx_h_check_control_port_protocol` | Mark EAPOL frames: no-encrypt, minimum rate |
| 5 | `tx_h_select_key` | Choose encryption key: PTK (unicast), GTK/IGTK (multicast), default WEP keys. Set `info->control.hw_key` for HW crypto. Drop if encryption required but key missing. |

**Phase 2 -- Late handlers** (`invoke_tx_handlers_late()`, after dequeue):

| # | Handler | Purpose |
|---|---------|---------|
| 6 | `tx_h_rate_ctrl` | `rate_control_get_rate()`: select TX rate, RTS/CTS, retry counts. Falls back to station rate table if RC returns nothing. |
| 7 | `tx_h_michael_mic_add` | Append TKIP Michael MIC |
| 8 | `tx_h_sequence` | Assign per-TID sequence number (`sta->tid_seq[tid]`). Per-VIF for non-QoS. MLD multicast uses `sdata->mld_mcast_seq`. |
| 9 | `tx_h_fragment` | Fragment if frame exceeds frag_threshold; build fragment chain |
| 10 | `tx_h_stats` | Update per-AC TX byte/packet statistics |
| 11 | `tx_h_encrypt` | Software encryption dispatch: WEP / TKIP / CCMP / CCMP-256 / AES-CMAC / BIP-CMAC-256 / BIP-GMAC / GCMP |
| 12 | `tx_h_calculate_duration` | Compute Duration/ID field for NAV virtual carrier sense |

### 3.4 Fast TX Path

When a `fast_tx` descriptor is cached for a STA (built by
`ieee80211_check_fast_xmit()`), `__ieee80211_xmit_fast()` bypasses the
entire handler chain:

```
  Pre-computed: 802.11 header template, key info, RA/TA
  On TX:  copy header template -> set sequence -> encrypt -> drv_tx()
```

Conditions: STA must be authorized, key must be stable, no A-MSDU
aggregation needed, no special flags.

### 3.5 TX Queueing and Scheduling

When TXQ support is active (drivers implementing `wake_tx_queue`):

```
  ieee80211_tx_prepare()
       |-> check for active BA session on TID
       |   if session pending: queue in tid_tx->pending
       |   if no session + RC supports it: trigger ieee80211_start_tx_ba_session()
       |
       v
  ieee80211_txq_enqueue()           enqueue to fair-queuing scheduler
       |
       v
  local->fq (fq_codel)             per-TID TXQ, CoDel AQM for latency
       |
       v  drv_wake_tx_queue() notification
       |
  Driver dequeues via ieee80211_tx_dequeue() / ieee80211_tx_dequeue_nq()
       |  -> invoke_tx_handlers_late() on dequeued frame
       |  -> may aggregate multiple frames into A-MPDU
       v
  drv_tx()
```

When TXQ is not active, frames go directly to `local->pending[queue]` or
the driver queue.

### 3.6 A-MPDU TX Aggregation

When a BA session is `OPERATIONAL` on a TID:

1. Driver dequeues multiple frames for the same TID via
   `ieee80211_tx_dequeue()`.
2. Frames are grouped into an A-MPDU (hardware or software aggregation).
3. `info->flags |= IEEE80211_TX_CTL_AMPDU`.
4. On TX status: `status.ampdu_ack_len / ampdu_len` report per-MPDU ACK
   results back to rate control.

### 3.7 TX Status Processing

After the driver reports TX completion:

```
  ieee80211_tx_status_ext() / ieee80211_tx_status_irqsafe()
       |
       |-> Filtered frame?  -> ieee80211_handle_filtered_frame()
       |                       (requeue to ps_tx_buf or software retry)
       |
       |-> rate_control_tx_status()    feed ACK/retry info to rate algorithm
       |
       |-> ieee80211_frame_acked()     check for pending BAR on TID
       |
       |-> Monitor mode: build radiotap TX status header
       |
       |-> A-MPDU status: update BA window tracking
       v
  Done
```

---

## 4. RX Packet Processing Flow

### 4.1 Overview

```
  Hardware (received 802.11 MPDU)
       |
       v
  ieee80211_rx_napi()  /  ieee80211_rx_list()  /  ieee80211_rx_irqsafe()
       |
       v
  Monitor interfaces delivery  (ieee80211_rx_monitor)
       |
       v
  __ieee80211_rx_handle_packet()  (802.11 format)
  __ieee80211_rx_handle_8023()    (802.3 format, HW decap offload)
       |
       |  Parse QoS -> extract TID, set skb->priority
       |  Scan RX for probe responses / beacons
       |  Look up STA by addr2 (RCU hash table lookup)
       |
       v
  Fast RX path available?  (ieee80211_invoke_fast_rx)
       |  yes: validate -> 802.11-to-802.3 -> deliver to stack
       |  no:  fall through to full handler chain
       v
  ieee80211_invoke_rx_handlers()
       |
       |--- Stage A (before reordering) ---|
       |                                    |
       |  reorder point                     |
       |  (ieee80211_rx_reorder_ampdu)      |
       |                                    |
       |--- Stage B (after reordering) -----|
       |
       v
  ieee80211_deliver_skb()  ->  netif_receive_skb()  ->  local network stack
                                   (and intra-BSS multicast forwarding in AP mode)
```

### 4.2 Entry Points

| Entry | Context | When |
|-------|---------|------|
| `ieee80211_rx_napi()` | NAPI poll (softirq) | Preferred. Takes RCU read lock, delivers via `napi_gro_receive()`. |
| `ieee80211_rx_list()` | Any | Core path. Processes frame, appends to local `list` for batch delivery. |
| `ieee80211_rx_irqsafe()` | Hard IRQ | Queues to `local->skb_queue`, schedules tasklet for deferred processing. |

The driver fills `IEEE80211_SKB_RXCB(skb)` with band, rate, signal, encoding,
flags before calling these functions.

### 4.3 Fast RX Path

When a `fast_rx` descriptor is available (built by
`ieee80211_check_fast_rx()`), `ieee80211_invoke_fast_rx()` provides an
optimized bypass:

```
  Validate: frame control, DS bits, addresses, encryption state
  Strip IV/MIC (if HW decryption)
  Convert 802.11 header -> 802.3 header (direct memcpy)
  Mesh forwarding (if applicable)
  -> ieee80211_rx_8023() -> netif_receive_skb()
```

### 4.4 Full RX Handler Chain

**Stage A -- Before reordering** (`ieee80211_invoke_rx_handlers()`):

| # | Handler | Purpose |
|---|---------|---------|
| 1 | `rx_h_check_dup` | Duplicate detection via per-TID sequence tracking (`sta->last_seq_ctrl[]`). Multicast dedup via `sdata->u.mgd.mcast_seq_last`. |
| 2 | `rx_h_check` | Drop Class 3 frames from unassociated STAs (except control port EAPOL). Mesh validation. |

**Reordering point:**

| # | Handler | Purpose |
|---|---------|---------|
| 3 | `rx_reorder_ampdu` | If TID has active BA session: buffer out-of-order MPDUs in `reorder_buf[]`. Direct-pass if at `head_seq_num` and buffer empty. Release in-order frames via `ieee80211_sta_reorder_release()`. Timeout: 100 ms. |

**Stage B -- After reordering** (`ieee80211_rx_handlers()`):

| # | Handler | Purpose |
|---|---------|---------|
| 4 | `rx_h_check_more_data` | If PS-polling: send another PS-poll when More Data bit set |
| 5 | `rx_h_uapsd_and_pspoll` | AP-side: handle PS-Poll / U-APSD trigger frames. Deliver buffered frames. |
| 6 | `rx_h_sta_process` | Update STA statistics (signal, bytes, last_rx). Handle PS mode transitions. Drop nullfunc. |
| 7 | `rx_h_decrypt` | Key selection + decryption. Order: PTK (unicast), BIGTK (beacons), IGTK (mcast mgmt), GTK (mcast data), WEP indices. Dispatches to cipher-specific decrypt. |
| 8 | `rx_h_defragment` | Reassemble fragmented MSDUs. Per-STA fragment cache. Validate sequential PN for CCMP/GCMP. |
| 9 | `rx_h_michael_mic_verify` | TKIP Michael MIC verification |
| 10 | `rx_h_amsdu` | Split A-MSDU into individual 802.3 subframes. Each subframe -> `ieee80211_deliver_skb()`. |
| 11 | `rx_h_data` | Convert 802.11 data -> 802.3 (`ieee80211_data_to_8023()`). Mesh forwarding. 802.1X port control check. `ieee80211_deliver_skb()`. |
| 12 | `rx_h_ctrl` | Handle BlockAck Request (BAR): release reorder buffer up to BAR's SSN. |
| 13 | `rx_h_mgmt_check` | Validate management frames. Report beacons to cfg80211. Check BSS color collision. |
| 14 | `rx_h_action` | Process action frames by category: HT, VHT, BACK (ADDBA/DELBA), spectrum mgmt, mesh, self-protected, TWT. Unknown actions queued for userspace. |
| 15 | `rx_h_userspace_mgmt` | Deliver unhandled management frames to userspace via `cfg80211_rx_mgmt_ext()`. |
| 16 | `rx_h_action_post_userspace` | Post-userspace kernel action handling (SA Query). |
| 17 | `rx_h_action_return` | Return unknown action frames to sender with category bit 0x80 set. |
| 18 | `rx_h_ext` | S1G extended frame handling. |
| 19 | `rx_h_mgmt` | Queue management frames (auth, beacon, probe resp, deauth, disassoc) to `sdata->skb_queue` for MLME processing. |

### 4.5 Frame Delivery

`ieee80211_deliver_skb()` is the final exit point for data frames:

```
  ieee80211_deliver_skb()
       |
       |-> Control port frame? -> cfg80211_rx_control_port() (to userspace)
       |
       |-> netif_receive_skb()          deliver to local network stack
       |
       |-> AP mode + multicast/broadcast?
       |    -> dev_queue_xmit()          forward to wireless medium (intra-BSS)
       v
  Done
```

Management frames exit via `ieee80211_queue_skb_to_iface()` which enqueues
to `sdata->skb_queue` and schedules `ieee80211_iface_work()` for deferred
MLME processing.

### 4.6 Queues and Buffering Summary

| Queue | Owner | Purpose |
|-------|-------|---------|
| `sta->ps_tx_buf[ac]` | STA (AP side) | Per-AC unicast power-save buffer (max `STA_MAX_TX_BUFFER`) |
| `ps->bc_buf` | AP BSS | Multicast/broadcast buffer for DTIM delivery |
| `sta->tx_filtered[ac]` | STA (AP side) | TX-filtered frames (HW rejected, e.g., STA asleep) |
| `tid_tx->pending` | BA session | A-MPDU TX session pending queue |
| `tid_agg_rx->reorder_buf[]` | BA session | Per-TID RX reorder buffer |
| `local->pending[queue]` | Hardware | Pending frames when hardware queues stopped |
| `local->fq` | TXQ scheduler | Fair-queuing + CoDel AQM per-TID TXQ |
| `local->active_txq[ac]` | TXQ scheduler | Per-AC active TXQ list for airtime scheduling |
| `sdata->skb_queue` | Interface | Management frame queue for `ieee80211_iface_work()` |

---

## 5. WMM / QoS Classification

File: `wme.c`. Maps 802.1D priorities to 802.11e Access Categories:

```
  802.1D Priority    Access Category
  0, 3               BE  (Best Effort)
  1, 2               BK  (Background)
  4, 5               VI  (Video)
  6, 7               VO  (Voice)    -- control port frames always VO
```

ACM (Admission Control Mandatory) enforcement: if an AC has ACM set, frames
are downgraded: `VO -> VI -> BE -> BK`.

---

## 6. Rate Control

File: `rate.c`, algorithm in `rc80211_minstrel_ht.c`.

mac80211 provides a rate control framework. Drivers that set
`IEEE80211_HW_HAS_RATE_CONTROL` bypass this entirely.

Default algorithm: **Minstrel-HT** (`CONFIG_MAC80211_RC_MINSTREL`).

| Function | When called | Purpose |
|----------|------------|---------|
| `rate_control_rate_init()` | STA added / link activated | Initialize rate table for the STA |
| `rate_control_get_rate()` | Each TX frame (in `tx_h_rate_ctrl`) | Select TX rate, retries, RTS/CTS |
| `rate_control_tx_status()` | TX status received | Feed ACK/timeout feedback to algorithm |
| `rate_control_rate_update()` | SMPS / channel width change | Notify algorithm of parameter changes |

---

## 7. Cryptographic Processing

### TX (software path, `tx_h_encrypt`)

| Cipher | Function |
|--------|----------|
| WEP | `ieee80211_wep_encrypt()` |
| TKIP | `ieee80211_tkip_encrypt()` + Michael MIC |
| CCMP / CCMP-256 | `ieee80211_ccmp_encrypt()` |
| GCMP | `ieee80211_gcmp_encrypt()` |
| AES-CMAC (BIP) | `ieee80211_aes_cmac_encrypt()` |
| BIP-GMAC | `ieee80211_aes_gmac_encrypt()` |

Hardware encryption: `tx_h_select_key` sets `info->control.hw_key`; driver
handles encryption.

### RX (software path, `rx_h_decrypt`)

Key selection priority: PTK (unicast) > BIGTK (beacon) > IGTK (multicast mgmt)
> GTK (multicast data) > WEP key index fallback.

Decryption functions mirror the TX equivalents. After decryption, IV/MIC are
stripped (or verified and stripped if hardware already decrypted).

---

## 8. Mesh Integration

Mesh networking (`CONFIG_MAC80211_MESH`) extends the data path:

**RX mesh path** (`ieee80211_rx_mesh_data()`):
1. Decrement TTL; drop if TTL <= 0.
2. If destination is local: deliver to stack.
3. If destination is another mesh STA: resolve next hop via
   `mesh_nexthop_lookup()` (HWMP path table).
4. Forward via `ieee80211_add_pending_skb()` or fast-forward via
   `ieee80211_rx_mesh_fast_forward()`.

**TX mesh path:**
- 4-address format (ToDS + FromDS) with mesh control element in frame body.
- QoS header sets mesh-control-present flag.

**Mesh peering state machine** (`mesh_plink.c`): `OPEN -> CNF_RCVD -> ESTAB ->
HOLDING -> OPEN` (simplified). Managed per-peer with plink timers.

**HWMP routing** (`mesh_hwmp.c`): PREQ/PREP/PERR generation and processing for
on-demand path discovery. Metric calculation based on airtime.

---

## 9. MLO (Multi-Link Operation) / 802.11be

mac80211 supports 802.11be Multi-Link Devices:

- **Per-link station data**: `link_sta_info` with per-link address, stats,
  bandwidth, capability.
- **Per-link BSS config**: `ieee80211_bss_conf` with per-link channel, TX power.
- **Link management**: `ieee80211_sta_allocate/activate/remove_link()`.
- **Driver notifications**: `change_sta_links()`, `change_vif_links()`,
  `can_activate_links()`.
- **TID-to-Link Mapping (TTLm)**: `ieee80211_process_neg_ttlm_req/res()`.
- **Fast TX/RX caches**: rebuilt when link configuration changes.
- **Statistics**: removed-link stats accumulated in `rem_link_stats` before
  link STA is freed.

---

## 10. Concurrency and Locking

| Lock / Mechanism | Scope |
|------------------|-------|
| **wiphy mutex** (`local->hw.wiphy->mtx`) | Primary serialization lock. Asserted by all `drv_*()` wrappers and most configuration paths. |
| **RCU** | STA hash table lookups, interface iteration, link STA access. STA freed via `kfree_rcu` after `synchronize_net()`. |
| **`sta->lock`** | Per-STA spinlock protecting aggregation state, PS transitions. |
| **`sta->ps_lock`** | Serializes power-save wakeup and frame delivery. |
| **`sta->rate_ctrl_lock`** | Rate control data access. |
| **`local->active_txq_lock[ac]`** | Per-AC TXQ scheduling list. |
| **`local->fq` lock** | Fair-queuing scheduler internal lock. |
| **`iflist_mtx`** | Interface list modifications. |

Locking hierarchy: RTNL -> wiphy mutex -> `iflist_mtx` -> per-STA locks.

---

## 11. Tracepoints

All driver callbacks and major internal events have tracepoints defined in
`trace.h`. Enable with `CONFIG_MAC80211_MESSAGE_TRACING` for message-level
tracing. Standard tracepoints include:

- `drv_start` / `drv_stop` / `drv_tx` / `drv_rx`
- `drv_sta_state` (old_state -> new_state)
- `drv_ampdu_action`
- `ieee80211_tx_hdr` / `ieee80211_rx_hdr`
- `ieee80211_bss_info_change_notify`

View via: `trace-cmd record -e mac80211 \*`
