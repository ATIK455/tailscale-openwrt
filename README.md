# Lightweight Tailscale for 16MB OpenWrt Routers (Runs from RAM)

[![Build and Release Tailscale for OpenWrt](https://github.com/ATIK455/tailscale-openwrt/actions/workflows/build.yml/badge.svg)](https://github.com/ATIK455/tailscale-openwrt/actions/workflows/build.yml)

A heavily optimized, stripped, and UPX-compressed **Tailscale ARM64 (arch64)** daemon tailored specifically for flash-constrained OpenWrt routers (such as the **Cudy WR3000 v1** with only 16MB flash storage).

> [!NOTE]
> **100% RAM-Disk Execution:** This binary is designed to be downloaded directly into **RAM (/tmp)** on every boot. **Zero bytes** are written to your router's permanent flash overlay (/overlay), completely eliminating the risk of flash storage exhaustion or overlay corruption while retaining full **Subnet Routing**.

---

## ?? Why This Exists

* **The Problem:** Standard Tailscale installations on OpenWrt require **25MB+** of storage, but a 16MB flash router typically only has **1.5MB - 5.5MB** of free overlay space. Installing official packages directly to flash will instantly overflow the partition.
* **The Solution:** 
  1. We strip unnecessary server/cloud bloat (AWS, Kubernetes, Bird, etc.).
  2. We **explicitly retain 
etstack** so full **Subnet Routing** works seamlessly.
  3. We compress the executable with **UPX (LZMA)** down to **~6 MB**.
  4. The router downloads this binary directly into its **RAM-Disk (/tmp)** on boot, leaving 100% of your router's permanent flash storage untouched.
  5. The authentication state (/etc/tailscale/tailscaled.state) is preserved in flash (<3 KB), so you never have to re-authenticate on reboot!

---

## ?? Direct Download URL for OpenWrt

This direct URL always points to the latest compiled binary:
`ash
wget -q https://raw.githubusercontent.com/ATIK455/tailscale-openwrt/main/tailscale.combined -O /tmp/tailscaled
chmod +x /tmp/tailscaled
`

---

## ?? Router Startup Script (/etc/rc.local)

To make your router automatically download and run Tailscale in RAM on boot, add the following to /etc/rc.local:

`sh
(
    # 1. Restore OpenWrt default DNS symlink (Prevents MagicDNS deadlock)
    rm -f /etc/resolv.conf
    ln -sf /tmp/resolv.conf.d/resolv.conf.auto /etc/resolv.conf

    # 2. Wait for internet connection
    while true; do
        if ping -c 1 8.8.8.8 >/dev/null 2>&1; then
            logger -t tailscale "Internet is up! Syncing time and starting Tailscale in RAM..."
            
            # 3. Force time sync (Prevents WireGuard timestamp rejections after power loss)
            ntpd -n -q -p 0.openwrt.pool.ntp.org -p time.google.com
            
            # 4. Download directly into RAM (/tmp) and launch Tailscale
            wget -q https://raw.githubusercontent.com/ATIK455/tailscale-openwrt/main/tailscale.combined -O /tmp/tailscaled
            if [ -s /tmp/tailscaled ]; then
                chmod +x /tmp/tailscaled
                /tmp/tailscaled --statedir=/etc/tailscale >/dev/null 2>&1 &
                break
            fi
        fi
        sleep 5
    done
) >/dev/null 2>&1 &

exit 0
`

---

## ?? How Updates Work

This repository features an automated **GitHub Actions Workflow**:

1. **Automated Schedule:** Checks and compiles the latest stable Tailscale release on the 1st of every month automatically.
2. **On-Demand (Maintainer / Forks):** The repository owner (or anyone who forks this repo) can trigger an on-demand build anytime with 1 click under the **Actions** tab by selecting Run workflow.
3. The workflow compiles the binary on a GitHub cloud runner, compresses it with UPX, pushes 	ailscale.combined directly to main, and publishes a formal release under [Releases](https://github.com/ATIK455/tailscale-openwrt/releases).
4. Any router reboot will automatically fetch the latest build directly into RAM!

---

## 📄 License

This project is licensed under the [BSD 3-Clause License](LICENSE), matching upstream [Tailscale](https://github.com/tailscale/tailscale/blob/main/LICENSE).
