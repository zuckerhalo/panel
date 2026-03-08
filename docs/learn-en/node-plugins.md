---
sidebar_position: 9
title: Node Plugins β
---

Node Plugins are additional modules that can be activated on a Remnawave Node to enable extra functionality: **Torrent Blocker**, **Blacklist** and **Connection Drop**.

<!-- truncate -->

:::warning

🚧 This page is still under development.

---

Node Plugins are available starting from Remnawave Panel and Remnawave Node **2.7.0**.

:::

<img src={require('/node-plugins/plugins-preview.webp').default} width="100%" style={{borderRadius: '8px'}} alt="Node Plugins" />

## Requirements

For Plugins to function correctly, you must add the `cap_add: NET_ADMIN` directive to the Remnawave Node configuration.

```yaml title="docker-compose.yml"
services:
    remnanode:
        container_name: remnanode
        hostname: remnanode
        image: remnawave/node:latest
        network_mode: host
        restart: always
        // highlight-next-line-green
        cap_add:
            // highlight-next-line-green
            - NET_ADMIN
        ulimits:
            nofile:
                soft: 1048576
                hard: 1048576
        environment:
            - NODE_PORT=<NODE_PORT>
            - SECRET_KEY=<SECRET_KEY>
```

Also, ensure that `nftables` is installed on the host machine. It is generally pre-installed by default on most distributions. 

```bash title="Check the nftables version"
nft --version
```

:::warning

Be cautious when configuring the Plugins. The Torrent Blocker and Blacklist Plugins will interact directly with the firewall on your host machine.

:::

<details>
<summary>What table will Remnawave Node create for Plugin operation?</summary>

```nftables title="nftables"
table ip remnanode {
        set blacklist {
                type ipv4_addr
                flags timeout
        }

        set torrent-blocker {
                type ipv4_addr
                flags timeout
        }

        chain input {
                type filter hook input priority filter - 10; policy accept;
                ip saddr @blacklist log prefix "blacklist: " drop
                ip saddr @torrent-blocker log prefix "torrent-blocker: " drop
        }

        chain forward {
                type filter hook forward priority filter - 10; policy accept;
                ip saddr @blacklist log prefix "blacklist: " drop
                ip saddr @torrent-blocker log prefix "torrent-blocker: " drop
        }
}
table ip6 remnanode6 {
        set blacklist6 {
                type ipv6_addr
                flags timeout
        }

        set torrent-blocker6 {
                type ipv6_addr
                flags timeout
        }

        chain input {
                type filter hook input priority filter - 10; policy accept;
                ip6 saddr @blacklist6 log prefix "blacklist: " drop
                ip6 saddr @torrent-blocker6 log prefix "torrent-blocker: " drop
        }

        chain forward {
                type filter hook forward priority filter - 10; policy accept;
                ip6 saddr @blacklist6 log prefix "blacklist: " drop
                ip6 saddr @torrent-blocker6 log prefix "torrent-blocker: " drop
        }
}

```

</details>

## Configuration Structure

<img src={require('/node-plugins/configuration.webp').default} width="100%" style={{borderRadius: '8px'}} alt="Node Plugins" />

```json title="Plugin configuration"
{
    "blacklist": {
        "ip": [],
        "enabled": false
    },
    "torrentBlocker": {
        "enabled": false,
        "ignoreLists": {
            "ip": [],
            "userId": []
        },
        "blockDuration": 3600
    },
    "connectionDrop": {
        "enabled": false,
        "whitelistIps": []
    },
    "sharedLists": []
}
```

:::tip

The Plugin configuration editor supports tooltips (hover over an object to view its description) and also offers autocompletion.

:::

## Torrent Blocker

The Torrent Blocker Plugin blocks the IP address from which BitTorrent traffic has been detected.

:::warning

This Plugin requires a minimum version of Xray-core – **??.??.??**. This version of the core comes with Remnawave Node **2.7.0**.

:::

### Configuration

```json
"torrentBlocker": {
    "enabled": false,
    "ignoreLists": {
        "ip": [],
        "userId": []
    },
    "blockDuration": 3600
},
```

| Field           | Type    | Description                                                                 |
| --------------- | ------- | --------------------------------------------------------------------------- |
| `enabled`       | boolean | Enables or disables the Plugin. Default is `false`.              |
| `ignoreLists`   | object  | A list of IP addresses and user IDs to be ignored by the Plugin. |
| `blockDuration` | number  | The block duration in seconds.                                   |

```jsonc title="ignoreLists"
{
    "ip": [],
    "userId": []
}
```

| Field    | Type  | Description                                                                                                       |
| -------- | ----- | ----------------------------------------------------------------------------------------------------------------- |
| `ip`     | array | A list of IP addresses that the Plugin will ignore. You can also use entries from the `Shared Lists` configuration. |
| `userId` | array | A list of user IDs that the Plugin will ignore.                                                   |

### How It Works

Xray-core **cannot fully block BitTorrent traffic**, it can only detect roughly 10–30% of it.  

However, we can still take action based on that information. With the **??.??.??** version of Xray-core a new feature, `webhook`, has been added (see [#5722](https://github.com/XTLS/Xray-core/pull/5722)). When Xray-core detects a BitTorrent packet, it informs Remnawave Node through a webhook, which then:

- Blocks the IP in `nftables`
- Terminates any active connections (similar to `ss -k` or `conntrack -D`)

Step-by-step workflow:

1. A user generates BitTorrent traffic.
2. The traffic triggers a rule added by Remnawave Node in Xray-core.
3. Xray-core sends packet info to the Node via webhook.
4. Remnawave Node terminates the connection and blocks the IP.
5. Within ~15–30 seconds, this information is sent to Remnawave Panel.
6. Remnawave Panel notifies the administrator via Telegram and also triggers a webhook (`torrent_blocker.report`, scope: `torrent_blocker`).

This allows you to reasonably block the majority of BitTorrent connections, making torrenting largely unusable.

:::warning

Enabling this Plugin automatically handles the Xray-core configuration.  
**Do not modify the configuration manually.**

:::

<details>
<summary>How enabling the Plugin affects the Xray-core configuration?</summary>

```json
{
    "inbounds": [],
    "outbounds": [
        {
            "tag": "DIRECT",
            "protocol": "freedom"
        },
        {
            "tag": "BLOCK",
            "protocol": "blackhole"
        },
        // highlight-next-line-green
        {
            // highlight-next-line-green
            "tag": "RW_TB_OUTBOUND_BLOCK",
            // highlight-next-line-green
            "protocol": "blackhole"
            // highlight-next-line-green
        }
    ],
    "routing": {
        "rules": [
            // highlight-next-line-green
            {
                // highlight-next-line-green
                "protocol": [
                    // highlight-next-line-green
                    "bittorrent"
                    // highlight-next-line-green
                ],
                // highlight-next-line-green
                "outboundTag": "RW_TB_OUTBOUND_BLOCK",
                // highlight-next-line-green
                "webhook": {
                    // highlight-next-line-green
                    "url": "<REPLACED IN RUNTIME BY REMNAWAVE NODE>",
                    // highlight-next-line-green
                    "deduplication": 30
                    // highlight-next-line-green
                }
            }
        ]
    }
}
```
The rule is always inserted at the beginning of the `rules` array, before existing rules, while the outbound entry is always appended at the end of the `outbounds` array.

</details>

:::tip

Further actions regarding the abuser must be handled manually by the administrator. The Panel provides all the tools needed for this. For example, you can change the user's status to `DISABLED`, if you wish to block the abuser entirely.  

Since the Panel sends out a webhook, you can also automate these actions based on your own logic.

:::

<details>
<summary>What will the `torrent_blocker.report` webhook contain?</summary>

The webhook will include the full objects for `node` and `user`, along with a `report` object containing all available information:

```json
{
    "scope": "torrent_blocker",
    "event": "torrent_blocker.report",
    "timestamp": "2026-03-07T16:02:50.564Z",
    "data": {
        "node": {},
        "user": {},
        "report": {
            "actionReport": {
                "blocked": true,
                "ip": "<omitted>",
                "blockDuration": 60,
                "willUnblockAt": "2026-03-07T16:03:48.986Z",
                "userId": "2",
                "processedAt": "2026-03-07T16:02:48.986Z"
            },
            "xrayReport": {
                "email": "2",
                "level": 0,
                "protocol": "bittorrent",
                "network": "tcp",
                "source": "<omitted>:51431",
                "destination": "<omitted>:59755",
                "routeTarget": null,
                "originalTarget": "tcp:<omitted>:59755",
                "inboundTag": "VLESS_TCP_REALITY",
                "inboundName": "vless",
                "inboundLocal": "<omitted>:443",
                "outboundTag": "RW_TB_OUTBOUND_BLOCK",
                "ts": 1772899368
            }
        }
    }
}
```
</details>

## Blacklist

The Blacklist Plugin permanently blocks IP addresses that are included in it's list.

:::warning

The Blacklist Plugin is a powerful and potentially dangerous tool. Use it with caution when adding IPs to the list.

:::

### Configuration

```json
"blacklist": {
    "ip": [],
    "enabled": false
}
```

| Field     | Type    | Description                                                                                                      |
| --------- | ------- | ---------------------------------------------------------------------------------------------------------------- |
| `enabled` | boolean | Enables or disables the Plugin. Default is `false`.                                                           |
| `ip`      | array   | A list of IP addresses to be blocked by the Plugin. You can use lists from the `Shared Lists` configuration. |

### How It Works

IP addresses specified in the list will be blocked through `nftables`.

## Connection Drop

The Connection Drop feature is not a full plugin per se, but rather a small enhancement that allows you to whitelist IP addresses for the Connection Drop functionality.

Starting from Remnawave Node **2.6.0** – when the `cap_add: NET_ADMIN` directive is enabled – Remnawave Node will automatically drop connections when a user is removed from Xray-core. And when using [bridges](/docs/learn-en/server-routing), this could result in the bridge connection being dropped.

From Remnawave Node **2.7.0** onwards, you can whitelist IPs for the Connection Drop functionality.

:::tip

Connection Drop functionality will **always be active**. Here, you only control whether or not a whitelist is enabled for this feature.

:::

### Configuration

```json
"connectionDrop": {
    "enabled": false,
    "whitelistIps": []
}
```

| Field          | Type    | Description                                                                             |
| -------------- | ------- | --------------------------------------------------------------------------------------- |
| `enabled`      | boolean | Enables or disables the Plugin. Default is `false`.                                    |
| `whitelistIps` | array   | A list of IP addresses to be whitelisted. |

## Shared Lists

Shared Lists are collections of IP addresses that can be referenced and used by other Plugins.

### Configuration

```json
"sharedLists": [
    {
        "name": "ext:my-list",
        "type": "ipList",
        "items": ["127.0.0.1", "127.0.0.2"]
    }
]
```

| Field   | Type   | Description                            |
| ------- | ------ | -------------------------------------- |
| `name`  | string | The list name. Must start with `ext:`  |
| `type`  | string | The list type. Must be `ipList`        |
| `items` | array  | A list of IP addresses                 |

```json title="Example Usage in Torrent Blocker Configuration"
"torrentBlocker": {
    "enabled": false,
    "ignoreLists": {
        "ip": ["ext:my-list"]
    },
    "blockDuration": 3600
}
```

## Executor

<img src={require('/node-plugins/executor.webp').default} width="100%" style={{borderRadius: '8px'}} alt="Executor" />

**Executor** lets you send commands for execution, such as temporarily blocking IP addresses, unblocking IP addresses, or resetting the `nftables` table.

:::warning
Executor does not require any Plugins to function, but the `cap_add: NET_ADMIN` directive must be enabled.
:::

### Block IPs

<img src={require('/node-plugins/executor-ip-block.webp').default} width="100%" style={{borderRadius: '8px'}} alt="Node Plugins" />

This command is used to **temporarily block** IP addresses. You can specify multiple IPs and the block duration in seconds. A value of `0` means the block will last until Remnawave Node is restarted or the configuration of a Plugin associated with this Node is changed.

:::tip

Do **not** use this command for permanent IP blocks. The Blacklist Plugin was designed for that purpose.

:::

### Unblock IPs

<img src={require('/node-plugins/executor-ip-unblock.webp').default} width="100%" style={{borderRadius: '8px'}} alt="Node Plugins" />

This command is used to **remove IP blocks**. Executing it sends a request to remove the specified IP(s) from the `nftables` table.

### Reset nftables

This command **resets the nftables table**. When executed, it sends a request to recreate the `nftables` table from scratch.
