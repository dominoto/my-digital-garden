---
{"dg-publish":true,"permalink":"/resources/a-secure-port-free-homelab-with-cloudflare-tunnels-cosmos-and-split-dns/","created":"2024-07-30","updated":"2025-10-12"}
---


**Topics:** ==[[Resources/Homelab\|Homelab]]==

### My Homelab: Background

I run a small homelab server to host a handful of containerized services. Initially, I accessed these services directly using their local IP addresses and port numbers.

- **CPU:** Intel N100
    
- **RAM:** 16 GB DDR5
    
- **Boot Drive:** 128 GB NVMe
    
- **Storage:** ~5 TB pooled with mergerfs and SnapRAID
    

To improve security, ease of use, and enable remote access, I decided to switch from IP-based access to a proper domain-based setup. This guide details that process.

---

### Guide Introduction

This guide provides a walkthrough for setting up a secure, self-hosted application environment. The goal is to create an architecture that:

1. Allows **secure remote access** to your services from anywhere.
    
2. Requires **no open ports** on your home router, significantly boosting security.
    
3. Provides the **fastest possible connection** when you're on your local network.

This guide provides a comprehensive walkthrough for setting up a secure, self-hosted application environment. The final architecture allows for secure remote access to your services from anywhere in the world without opening any ports on your router, while also providing the fastest possible connection when you are at home. 

#### **Core Technologies:**

*   **Host OS:** **OpenMediaVault (OMV)**
*   **Application Hosting:** **Docker**, managed via the **Cosmos Cloud** UI.
*   **Domain & Public DNS:** A custom domain managed by **Cloudflare**.
*   **Secure Remote Access:** **Cloudflare Tunnel** & **Cloudflare Access** (from the Zero Trust platform).
*   **Local DNS:** **Technitium DNS Server**.

---

### **Part 1: The Cloudflare Foundation (The Public-Facing Layer)**

1.  **Obtain a Custom Domain** and add it to a free Cloudflare account.
2.  **Secure Your Cloudflare Account:**
    *   Enable **Two-Factor Authentication (2FA)**.
    *   Set SSL/TLS encryption mode to **Full (Strict)** under **SSL/TLS -> Overview**.
    *   Enable **DNSSEC** under **DNS -> Settings**.
3.  **Set Up DNS Security Records:** In the **DNS -> Records** section, add the following `TXT` and `CAA` records.
    *   **SPF (for email):** **Type:** `TXT`, **Name:** `@`, **Content:** `"v=spf1 -all"`
    *   **DMARC (for email):** **Type:** `TXT`, **Name:** `_dmarc`, **Content:** `"v=DMARC1; p=reject;"`
    *   **CAA (for certificates):** Create two records to allow Let's Encrypt to issue both standard and wildcard certificates.
        *   **Record 1:** **Type:** `CAA`, **Name:** `@`, **Tag:** `Only allow specific hostnames`, **CA domain name:** `letsencrypt.org`
        *   **Record 2:** **Type:** `CAA`, **Name:** `@`, **Tag:** `Only allow wildcards`, **CA domain name:** `letsencrypt.org`

---

### **Part 2: Server-Side Foundation (Cosmos & Local DNS)**

1.  **Install Cosmos Cloud** on your OMV server.
2.  **Deploy Technitium DNS Server** via the Cosmos App Store. Ensure its ports `53/tcp` and `53/udp` are correctly mapped in the Docker settings.
3.  **Configure Technitium for Split-DNS and Public Resolution:**
    *   Log in to the Technitium web UI.
    *   **Configure Upstream Forwarders:** Go to **Settings -> Proxy & Forwarders**. In the **Forwarders** section, add reliable public DNS servers (e.g., `https://dns.quad9.net/dns-query`, `https://cloudflare-dns.com/dns-query`). This allows Technitium to resolve public domains like `google.com`.
    *   **Create a Local Zone:** Go to the main **Zones** page and **Add Zone**. Name it after your domain (e.g., `kosto.top`) and set the type to `Primary Zone`.
    *   **Create a Wildcard `A` Record:** Inside the new zone, create a record with the name `*` pointing to your server's **local IP address** (e.g., `192.168.0.12`).
4.  **Configure Your Router's DHCP:**
    *   Log in to your router's admin page.
    *   Find the **DHCP Server** settings.
    *   Set the **Primary DNS Server** to the local IP address of your Technitium container. This directs all devices on your network to use it.

---

### **Part 3: The Cloudflare Tunnel (The Secure Gateway)**

1.  **Create the Tunnel in Cloudflare:**
    *   In the **Zero Trust Dashboard** (`one.dash.cloudflare.com`) under **Networks -> Tunnels**, create a new tunnel.
    *   Copy the **Docker** command's token.
2.  **Deploy the `cloudflared` Connector in Cosmos:**
    *   Deploy the `cloudflare/cloudflared:latest` image. **Do not worry about the command or other settings at this stage; we will fix them in the Troubleshooting section.** The initial deployment may fail or stop, which is expected.

---

### **Part 4: Exposing & Securing an Application (Example: Immich)**

1.  **Identify Application Network & Name:** In Cosmos, find the Immich container's **Network Name** (e.g., `cosmos-Immich-default`) and its **Service Name** from its compose file (e.g., `Immich`).
2.  **Connect the Tunnel:** In the Network Settings for your `cloudflared` container in Cosmos, **connect it to the application's network** (`cosmos-Immich-default`).
3.  **Create the Public Hostname in the Tunnel:** In the Zero Trust dashboard, add a public hostname to your tunnel.
    *   **Subdomain:** `immich`, **Domain:** `kosto.top`
    *   **Service -> URL:** `http://Immich:2283` (Use the exact `http://<Service Name>:<Internal Port>`).
4.  **Set Up a Cloudflare Access Login Method:** In the Zero Trust dashboard, go to **Settings -> Authentication** and enable the **One-Time Pin** method.
5.  **Secure the Application (The Dual-Application Method for Mobile App Compatibility):**
    *   **Create the "Bypass" Application for the API:** In **Access -> Applications**, create a new **Self-hosted** application.
        *   **Application name:** `Immich API (Bypass)`
        *   **Subdomain:** `immich`, **Domain:** `kosto.top`
        *   **Path:** `/api` *(This targets only API traffic)*.
        *   **Policy:** Create a policy with **Action:** `Bypass` and a rule for **Selector:** `Everyone`.
    *   **Create the "Secure" Application for the Web UI:** Create a second **Self-hosted** application.
        *   **Application name:** `Immich Web UI`
        *   **Subdomain:** `immich`, **Domain:** `kosto.top`
        *   **Path:** (Leave this blank to act as a catch-all).
        *   **Policy:** Create a policy with **Action:** `Allow` and a rule for **Selector:** `Emails`, **Value:** Your personal email address.

---

### **Part 5: Critical Host-Level Configuration (OMV & Docker)**

1.  **Fixing the OMV Firewall vs. Docker Conflict:**
    *   **The Problem:** The OMV firewall service (`nftables.service`) and the Docker engine fight for control of the system's firewall rules. Restarting `nftables` will wipe Docker's networking, breaking all your containers' connectivity.
    *   **The Solution:**
        1.  If your Docker networking is broken, fix it **one last time** by running `sudo systemctl restart docker` via SSH.
        2.  **Golden Rule:** From now on, **NEVER** manage the firewall from the command line. **ONLY** use the **OpenMediaVault Web UI** for any firewall changes.

2.  **Fixing the OMV Server's Own DNS Resolution:**
    *   **The Problem:** The OMV server itself does not use the Technitium DNS service it hosts, causing inconsistencies.
    *   **The Solution:**
        1.  Log in to the **OMV web panel**.
        2.  Go to **System -> Network -> Interfaces**.
        3.  **Edit** your main network interface.
        4.  In the **"DNS Servers"** field, add **only one**: `127.0.0.1`.
        5.  Save and apply the changes.

---

### **Part 6: Troubleshooting Appendix (Essential Fixes)**

*   **Snag: The `cloudflared` Tunnel Container is Stopping or Restarting**
    *   **Symptom:** The container is not "Healthy." The logs show a DNS error like `server misbehaving` or `Couldn't resolve SRV record`.
    *   **Cause:** The container is inheriting the host's inconsistent DNS and cannot reach the internet. It may also have an incorrect command or restart policy.
    *   **Solution:** You must manually correct the container's configuration. Go to the container's **Compose** editor in Cosmos and **replace the entire content** with the following correct JSON structure.
        ```json
        {
          "container_name": "cloudflared-tunnel",
          "image": "cloudflare/cloudflared:latest",
          "restart": "unless-stopped",
          "command": "tunnel run <PASTE_YOUR_TUNNEL_TOKEN_HERE>",
          "networks": [
            "cosmos-Immich-default",
            "cosmos-Jellyfin-default"
          ],
          "dns": [
            "1.1.1.1",
            "8.8.8.8"
          ]
        }
        ```
        **You must:**
        1.  Replace `<PASTE_YOUR_TUNNEL_TOKEN_HERE>` with your actual token.
        2.  Update the `"networks"` list to include the **exact network names** of all services you want to expose.

*   **Snag: The 522 Error at Home**
    *   **Cause:** NAT Loopback failure. Your device (e.g., a phone with its own DNS) is not using your local DNS server.
    *   **Solution:** Your Split-DNS is working correctly. The issue is with the client device's configuration.

*   **Snag: The 502 Bad Gateway**
    *   **Cause:** Docker networking issue. The `cloudflared` container cannot find the application container.
    *   **Solution:** In Cosmos, ensure the `cloudflared` container is connected to the application's specific Docker network (as defined in the corrected compose JSON above).

*   **Snag: Total DNS Failure (Timeouts) for All Devices**
    *   **Cause:** The Docker networking layer is broken, likely because the `nftables` service was restarted from the command line.
    *   **Solution:** SSH into OMV and run `sudo systemctl restart docker`. Then, follow the Golden Rule from Part 5.