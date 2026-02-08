.. SPDX-License-Identifier: GPL-2.0

==========================================
Wireless Network Stack Integration Guide
==========================================

:Author: Linux Kernel Wireless Development Community
:Date: February 2026

Introduction
============

This guide explains how the Linux kernel allows wireless network drivers to
integrate functionality through its wireless networking subsystem. The Linux
wireless stack provides a layered architecture that separates hardware-specific
driver code from protocol implementation and userspace interfaces.

Understanding this integration is crucial for:

* Wireless device driver developers
* System developers working with wireless functionality
* Contributors to the wireless networking stack
* Anyone wanting to understand Linux wireless architecture

Overview of the Wireless Stack Architecture
===========================================

The Linux wireless networking stack consists of several key layers:

::

    +------------------+
    | Userspace Apps   |  (iw, wpa_supplicant, NetworkManager)
    +------------------+
           |
    +------------------+
    |    nl80211       |  (Netlink-based configuration interface)
    +------------------+
           |
    +------------------+
    |    cfg80211      |  (Wireless Configuration API)
    +------------------+
           |
    +------------------+
    |    mac80211      |  (Software MAC layer - optional)
    +------------------+
           |
    +------------------+
    | Wireless Drivers |  (Hardware-specific implementations)
    +------------------+
           |
    +------------------+
    |    Hardware      |  (Wireless network cards)
    +------------------+

Key Components
--------------

1. **nl80211**: Netlink-based userspace interface for wireless configuration
2. **cfg80211**: Core wireless configuration and regulatory API
3. **mac80211**: Generic software implementation of IEEE 802.11 MAC layer
4. **Wireless Drivers**: Hardware-specific drivers

The cfg80211 Subsystem
=======================

Role and Purpose
----------------

cfg80211 is the wireless configuration API for 802.11 devices in Linux. It serves
as the bridge between userspace and drivers, providing:

* Device registration and management
* Regulatory domain enforcement
* Scanning and BSS (Basic Service Set) management
* Authentication and association handling
* Wireless extensions compatibility layer

Key Data Structures
-------------------

**struct wiphy**
    Represents a physical wireless device. Each wireless hardware device has
    one wiphy structure that describes its capabilities.

**struct wireless_dev**
    Represents a wireless network interface. A single wiphy can have multiple
    wireless_dev structures for virtual interfaces.

**struct cfg80211_ops**
    Operations structure that drivers must implement to provide functionality
    to cfg80211.

Device Registration Flow
-------------------------

A wireless driver integrates with cfg80211 through the following steps:

1. **Allocate wiphy structure**::

    struct wiphy *wiphy;
    wiphy = wiphy_new(&ops, sizeof(struct driver_priv));

   The wiphy structure contains:
   
   * Hardware capabilities (supported bands, channels, rates)
   * Supported interface modes (station, AP, monitor, etc.)
   * Maximum number of interfaces
   * Encryption capabilities
   * Feature flags

2. **Configure wiphy capabilities**::

    wiphy->interface_modes = BIT(NL80211_IFTYPE_STATION) |
                            BIT(NL80211_IFTYPE_AP);
    wiphy->bands[NL80211_BAND_2GHZ] = &driver_band_2ghz;
    wiphy->bands[NL80211_BAND_5GHZ] = &driver_band_5ghz;

3. **Register with cfg80211**::

    int ret = wiphy_register(wiphy);

4. **Create wireless interfaces**::

    struct wireless_dev *wdev;
    wdev = kzalloc(sizeof(*wdev), GFP_KERNEL);
    wdev->wiphy = wiphy;
    wdev->iftype = NL80211_IFTYPE_STATION;

Driver Callbacks
----------------

Drivers must implement callbacks defined in struct cfg80211_ops:

**Scanning Operations**:
    * `.scan()` - Initiate a scan for available networks
    * `.abort_scan()` - Cancel an ongoing scan

**Connection Management**:
    * `.connect()` - Connect to a network (for managed mode)
    * `.disconnect()` - Disconnect from a network
    * `.auth()` - Authenticate with an AP
    * `.assoc()` - Associate with an AP
    * `.deauth()` - Deauthenticate from an AP
    * `.disassoc()` - Disassociate from an AP

**Interface Management**:
    * `.add_virtual_intf()` - Create a virtual interface
    * `.del_virtual_intf()` - Remove a virtual interface
    * `.change_virtual_intf()` - Change interface type

**AP Mode Operations**:
    * `.start_ap()` - Start an access point
    * `.stop_ap()` - Stop an access point
    * `.change_beacon()` - Update beacon content

**Station Management**:
    * `.add_station()` - Add a station (for AP mode)
    * `.del_station()` - Remove a station
    * `.change_station()` - Modify station parameters
    * `.get_station()` - Retrieve station information

The mac80211 Subsystem
=======================

Role and Purpose
----------------

mac80211 is a software implementation of the IEEE 802.11 MAC layer. It provides:

* Frame transmission and reception handling
* Rate control algorithms
* Power management
* Aggregation (A-MPDU, A-MSDU)
* Block ACK handling
* Fragmentation and defragmentation
* Encryption/decryption (when not done in hardware)

When to Use mac80211
--------------------

Drivers should use mac80211 when:

* Hardware implements only PHY layer (software MAC)
* Hardware provides partial MAC implementation
* Driver wants to leverage common 802.11 functionality

Drivers should NOT use mac80211 when:

* Hardware provides full MAC implementation (fullmac devices)
* Hardware requires proprietary MAC implementation

In fullmac cases, drivers integrate directly with cfg80211.

Key Data Structures
-------------------

**struct ieee80211_hw**
    Represents hardware to mac80211. Contains hardware capabilities and state.

**struct ieee80211_ops**
    Hardware operations that drivers must implement for mac80211.

**struct ieee80211_vif**
    Represents a virtual interface in mac80211.

**struct ieee80211_sta**
    Represents a station (peer) in mac80211.

Driver Integration with mac80211
---------------------------------

1. **Allocate hardware structure**::

    struct ieee80211_hw *hw;
    hw = ieee80211_alloc_hw(sizeof(struct driver_priv), &driver_ops);

2. **Set hardware capabilities**::

    hw->flags = IEEE80211_HW_SIGNAL_DBM |
                IEEE80211_HW_SUPPORTS_PS |
                IEEE80211_HW_AMPDU_AGGREGATION;
    hw->wiphy->interface_modes = BIT(NL80211_IFTYPE_STATION);
    hw->queues = 4;  /* Number of hardware queues */

3. **Implement required callbacks**::

    static const struct ieee80211_ops driver_ops = {
        .tx = driver_tx,
        .start = driver_start,
        .stop = driver_stop,
        .add_interface = driver_add_interface,
        .remove_interface = driver_remove_interface,
        .config = driver_config,
        .configure_filter = driver_configure_filter,
        .bss_info_changed = driver_bss_info_changed,
    };

4. **Register with mac80211**::

    int ret = ieee80211_register_hw(hw);

mac80211 Driver Callbacks
--------------------------

**Core Operations**:
    * `.tx()` - Transmit a frame
    * `.start()` - Start the hardware
    * `.stop()` - Stop the hardware

**Interface Operations**:
    * `.add_interface()` - Add a virtual interface
    * `.remove_interface()` - Remove a virtual interface
    * `.config()` - Configure hardware parameters

**Frame Handling**:
    * `.configure_filter()` - Configure frame filtering
    * `.prepare_multicast()` - Prepare multicast address list

**BSS Operations**:
    * `.bss_info_changed()` - BSS parameters changed
    * `.set_key()` - Add/remove encryption key

**Advanced Features**:
    * `.ampdu_action()` - Handle A-MPDU operations
    * `.sta_add()` / `.sta_remove()` - Station management
    * `.get_stats()` - Retrieve statistics

Transmit and Receive Paths
===========================

Transmit Path (TX)
------------------

For mac80211-based drivers:

1. **Upper layers** submit packets through the network stack
2. **mac80211** receives packets, performs:
   
   * Rate selection
   * Aggregation (if supported)
   * Encryption (if not hardware-offloaded)
   * Fragmentation (if needed)
   * Adding 802.11 headers

3. **Driver's .tx() callback** receives prepared frames
4. **Driver** programs hardware DMA to transmit frames
5. **Hardware** transmits the frame
6. **Driver** receives TX completion interrupt
7. **Driver** calls `ieee80211_tx_status()` to report completion

For cfg80211-only (fullmac) drivers:

1. **cfg80211** receives packets from network stack
2. **Driver** receives data through netdev interface
3. **Driver/Hardware** handles MAC-layer processing
4. **Hardware** transmits the frame

Receive Path (RX)
-----------------

For mac80211-based drivers:

1. **Hardware** receives frame and generates interrupt
2. **Driver** retrieves frame from hardware
3. **Driver** fills `struct ieee80211_rx_status` with metadata:
   
   * Signal strength
   * Rate information
   * Channel information
   * Flags (encrypted, short preamble, etc.)

4. **Driver** calls `ieee80211_rx()` or `ieee80211_rx_irqsafe()`
5. **mac80211** processes frame:
   
   * Decryption (if needed)
   * Defragmentation
   * Deaggregation
   * Duplicate detection
   * Converting to 802.3 format

6. **Network stack** receives processed frame

For cfg80211-only (fullmac) drivers:

1. **Hardware** receives and processes frame
2. **Driver** receives processed frame from hardware
3. **Driver** passes frame to network stack via `netif_rx()`

Integration Example: Simplified Driver Flow
============================================

Here's a simplified example of how a driver integrates with mac80211:

Header and Includes
-------------------

::

    #include <linux/module.h>
    #include <linux/pci.h>
    #include <net/mac80211.h>

Driver Private Structure
------------------------

::

    struct mydriver_priv {
        struct pci_dev *pdev;
        void __iomem *regs;
        struct ieee80211_hw *hw;
        /* driver-specific fields */
    };

Hardware Operations
-------------------

::

    static int mydriver_tx(struct ieee80211_hw *hw,
                          struct ieee80211_tx_control *control,
                          struct sk_buff *skb)
    {
        struct mydriver_priv *priv = hw->priv;
        /* Program hardware to transmit frame */
        return 0;
    }

    static int mydriver_start(struct ieee80211_hw *hw)
    {
        struct mydriver_priv *priv = hw->priv;
        /* Initialize and start hardware */
        return 0;
    }

    static void mydriver_stop(struct ieee80211_hw *hw)
    {
        struct mydriver_priv *priv = hw->priv;
        /* Stop hardware */
    }

    static int mydriver_config(struct ieee80211_hw *hw, u32 changed)
    {
        /* Handle configuration changes (channel, power, etc.) */
        return 0;
    }

    static const struct ieee80211_ops mydriver_ops = {
        .tx = mydriver_tx,
        .start = mydriver_start,
        .stop = mydriver_stop,
        .config = mydriver_config,
        /* ... more callbacks ... */
    };

Driver Initialization
---------------------

::

    static int mydriver_probe(struct pci_dev *pdev,
                             const struct pci_device_id *id)
    {
        struct ieee80211_hw *hw;
        struct mydriver_priv *priv;
        int ret;

        /* Allocate hardware structure */
        hw = ieee80211_alloc_hw(sizeof(*priv), &mydriver_ops);
        if (!hw)
            return -ENOMEM;

        priv = hw->priv;
        priv->pdev = pdev;
        priv->hw = hw;

        /* Set hardware capabilities */
        hw->flags = IEEE80211_HW_SIGNAL_DBM;
        hw->wiphy->interface_modes = BIT(NL80211_IFTYPE_STATION);
        hw->queues = 4;

        /* Configure supported bands and rates */
        /* ... */

        /* Register with mac80211 */
        ret = ieee80211_register_hw(hw);
        if (ret) {
            ieee80211_free_hw(hw);
            return ret;
        }

        return 0;
    }

    static void mydriver_remove(struct pci_dev *pdev)
    {
        struct ieee80211_hw *hw = pci_get_drvdata(pdev);
        
        ieee80211_unregister_hw(hw);
        ieee80211_free_hw(hw);
    }

Regulatory Framework
====================

The Linux wireless stack includes a regulatory framework to ensure compliance
with local regulations regarding wireless spectrum usage.

Key Concepts
------------

**Regulatory Domain**
    A set of rules defining allowed channels, power limits, and other
    restrictions for a geographic region.

**World Regulatory Domain**
    A default, conservative regulatory domain used when the actual region
    is unknown.

Regulatory Hints
----------------

The regulatory framework receives hints from multiple sources:

1. **Driver hints**: `regulatory_hint(wiphy, alpha2_code)`
2. **User hints**: Via nl80211 from userspace
3. **Country IE**: From access point beacons
4. **Built-in database**: Kernel-compiled regulatory rules

Driver Integration
------------------

::

    /* Provide regulatory hint */
    regulatory_hint(wiphy, "US");

    /* For self-managed devices */
    wiphy->regulatory_flags |= REGULATORY_WIPHY_SELF_MANAGED;
    regulatory_set_wiphy_regd(wiphy, custom_regdomain);

Power Management Integration
=============================

The wireless stack supports various power management features:

Hardware Power Save (PS)
------------------------

For mac80211 drivers:

::

    hw->flags |= IEEE80211_HW_SUPPORTS_PS;
    hw->flags |= IEEE80211_HW_PS_NULLFUNC_STACK;

mac80211 will handle:

* Entering/exiting PS mode
* Sending NULL frames to AP
* Buffering frames during PS

Wake-on-WLAN (WoWLAN)
---------------------

::

    struct wiphy_wowlan_support wowlan = {
        .flags = WIPHY_WOWLAN_MAGIC_PKT |
                WIPHY_WOWLAN_DISCONNECT,
        .n_patterns = 4,
        .pattern_min_len = 1,
        .pattern_max_len = 128,
    };
    
    wiphy->wowlan = &wowlan;

Implement callbacks:

* `.suspend()` - Configure hardware for suspend
* `.resume()` - Restore hardware after resume

Advanced Features
=================

Rate Control
------------

mac80211 provides rate control algorithms:

* **Minstrel**: Default algorithm, optimal for most cases
* **Minstrel HT**: For 802.11n and later
* **PID**: Legacy algorithm

Drivers can use built-in algorithms or implement custom ones:

::

    hw->flags |= IEEE80211_HW_HAS_RATE_CONTROL;

Aggregation Support
-------------------

**A-MPDU (Aggregated MAC Protocol Data Unit)**:

::

    hw->flags |= IEEE80211_HW_AMPDU_AGGREGATION;

Implement `.ampdu_action()` callback to handle:

* RX_START / RX_STOP
* TX_START / TX_STOP_CONT / TX_STOP_FLUSH
* TX_OPERATIONAL

**A-MSDU (Aggregated MAC Service Data Unit)**:

::

    ieee80211_hw_set(hw, SUPPORTS_AMSDU_IN_AMPDU);

Monitor Mode
------------

Support for packet capture:

::

    wiphy->interface_modes |= BIT(NL80211_IFTYPE_MONITOR);

Implement frame injection via `.tx()` callback with proper radiotap header
handling.

Mesh Networking
---------------

802.11s mesh support:

::

    wiphy->interface_modes |= BIT(NL80211_IFTYPE_MESH_POINT);

Requires implementing mesh-specific callbacks in cfg80211_ops.

Testing and Debugging
======================

Virtual WiFi Driver (mac80211_hwsim)
-------------------------------------

The mac80211_hwsim driver provides a virtual wireless interface for testing:

::

    modprobe mac80211_hwsim radios=2

This creates virtual radios that can communicate with each other, useful for:

* Testing wireless stack changes
* Developing and testing user applications
* Protocol analysis
* Teaching and demonstrations

See `Documentation/networking/mac80211_hwsim/` for details.

Debugging Tools
---------------

**debugfs interface**:
    * `/sys/kernel/debug/ieee80211/phyX/`
    * Exposes internal state and statistics

**trace events**:
    * Enable with ftrace
    * `trace-cmd record -e mac80211 -e cfg80211`

**kernel logs**:
    * Dynamic debug: `echo 'module cfg80211 +p' > /sys/kernel/debug/dynamic_debug/control`

Userspace Tools
---------------

* **iw**: Modern nl80211-based configuration tool
* **wpa_supplicant**: WPA/WPA2/WPA3 authentication
* **hostapd**: Software access point
* **wireshark**: Packet capture and analysis

Best Practices for Driver Development
======================================

1. **Use mac80211 when possible**
   
   Unless hardware provides full MAC, leverage mac80211's tested implementation.

2. **Implement all required callbacks**
   
   At minimum: tx, start, stop, add_interface, remove_interface, config

3. **Report accurate RX status**
   
   Provide correct signal strength, rate, and channel information.

4. **Handle errors gracefully**
   
   Return appropriate error codes and clean up resources.

5. **Test thoroughly**
   
   Use mac80211_hwsim for initial development and testing.

6. **Follow regulatory requirements**
   
   Properly integrate with regulatory framework.

7. **Support power management**
   
   Implement suspend/resume callbacks.

8. **Use workqueues for deferred work**
   
   Don't perform long operations in interrupt context.

9. **Protect shared data**
   
   Use appropriate locking (spinlocks for IRQ context, mutexes otherwise).

10. **Document hardware quirks**
    
    Comment any workarounds for hardware issues.

Common Pitfalls to Avoid
========================

1. **Calling mac80211 from IRQ context inappropriately**
   
   Use `ieee80211_rx_irqsafe()` and `ieee80211_tx_status_irqsafe()` from
   interrupt handlers.

2. **Not handling all interface types**
   
   Test driver with different interface modes declared as supported.

3. **Incorrect locking**
   
   Follow mac80211/cfg80211 locking requirements documented in headers.

4. **Memory leaks**
   
   Always free allocated skbs, especially on error paths.

5. **Not updating TX status**
   
   Always call status reporting functions after transmission attempts.

6. **Ignoring return values**
   
   Check and handle errors from mac80211/cfg80211 calls.

References and Further Reading
===============================

Kernel Documentation
--------------------

* `Documentation/driver-api/80211/` - Comprehensive API documentation
* `Documentation/networking/regulatory.rst` - Regulatory framework
* `include/net/cfg80211.h` - cfg80211 API and documentation
* `include/net/mac80211.h` - mac80211 API and documentation

Source Code
-----------

* `net/wireless/` - cfg80211 implementation
* `net/mac80211/` - mac80211 implementation  
* `drivers/net/wireless/virtual/mac80211_hwsim.c` - Example virtual driver
* `drivers/net/wireless/` - Real hardware drivers

External Resources
------------------

* Linux Wireless Wiki: https://wireless.wiki.kernel.org/
* nl80211 family documentation
* IEEE 802.11 standards specifications

Conclusion
==========

The Linux wireless stack provides a flexible, layered architecture for
integrating wireless network functionality. By separating concerns between
cfg80211 (configuration), mac80211 (software MAC), and drivers (hardware),
the kernel achieves:

* Code reuse across different hardware
* Consistent userspace interfaces
* Centralized regulatory enforcement
* Easier driver development
* Better testability

Understanding this architecture is essential for anyone working with wireless
networking in Linux, whether developing drivers, implementing protocols, or
building applications.

For questions and contributions, contact the linux-wireless mailing list:
linux-wireless@vger.kernel.org
