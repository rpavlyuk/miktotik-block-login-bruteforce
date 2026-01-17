# block-login-attempts.script

Brute-force protection script for [MikroTik RouterOS](https://mikrotik.com/software) 
**You failed — you’re jailed.**

This script monitors RouterOS authentication logs and automatically blocks IP addresses that generate failed login attempts (API / SSH / WinBox / WebFig), by adding them to a firewall address-list with an expiration timeout.

Think of it as **Fail2Ban for MikroTik**, implemented using native RouterOS primitives: logs, scripts, and firewall address-lists.

## Features
- Blocks IPs after configurable number of failed login attempts
- Uses RouterOS logs as the single source of truth
- Automatic timeout with refresh on repeated attempts
- Supports trusted / local IP whitelist
- No external dependencies
- Transparent and auditable (everything is visible in `/log` and firewall)

## Tested Environment
- **RouterOS:** 7.2.x  
- **Platforms:** x86, CHR, RouterBOARD (should work on any ROS 7.x)


## How It Works (High Level)
1. Script scans RouterOS logs for messages starting with:
```
login failure for user
```
2. Extracts the source IP address from the log entry
3. Checks whether the IP belongs to a trusted (whitelisted) subnet
4. Counts failed attempts per IP
5. Once the threshold is reached:
- IP is added to a firewall address-list (`list_failed_attempt`)
- Timeout is set (default: `72h`)
- If the IP already exists in the list, its timeout is refreshed

## Prerequisite: Ensure System Authentication Logs Are Enabled

This script relies on **RouterOS system authentication logs**. Authenitication failures are being logged by default. If login failures are not written to the log, the script will have nothing to process. So let's ensure you have logging that this script needs to function.

### Required Log Topics
You must have at least the following log topics enabled:
- `system`
- `error`

### Verify Current Logging Configuration
Run:
```
/system logging print
```
You should see an entry similar to:
```
Topics: system,error
```

### Enable Required Logging (if missing)
If no suitable rule exists, add one:
```
/system logging add topics=system,error action=memory
```
> `action=memory` is sufficient and recommended for most setups.
### Verify That Login Failures Are Logged
Trigger a failed login (for example, wrong password via Winbox or Web), then run:
```
/log print where message~"^login failure for user"
```
You should see entries like:
```
login failure for user admin from 203.0.113.45 via api
```
If you **do not** see such messages, the script **will not work**.
### Notes
- The script intentionally matches only log messages starting with  `login failure for user` to avoid self-triggering and false positives.
- Custom logging actions or disabled system logs can break detection.

**General Advise**: Do **not** disable system authentication logs unless you fully understand the impact. We bet you do, but just in case :)


## Installation
1. Upload the [script](https://github.com/rpavlyuk/miktotik-block-login-bruteforce/blob/main/block-login-attempts.script) to your router:
```
/system script add name=block-login-attempts source=block-login-attempts.script
```
**OR** use [Winbox](https://mikrotik.com/winbox) for better use experience.
2. Schedule it (example: every 60 seconds):
```
/system scheduler add name=block-login-attempts interval=60s on-event=block-login-attempts
```
3. Create a firewall rule to actually drop the traffic:
```
/ip firewall filter
add chain=input src-address-list=list_failed_attempt action=drop comment="Drop brute-force login attempts"
```
## Updating
1. Just copy and paste a code of the [script](https://github.com/rpavlyuk/miktotik-block-login-bruteforce/blob/main/block-login-attempts.script) using [Winbox](https://mikrotik.com/winbox)

## Script Tuning
### Maximum Failed Attempts
At the top of the script:
```mikrotik
:global maxattampt 2
```
1 → instant ban on first failure
2 or 3 → safer if you have unstable connections or automation
**⚠️ VERY IMPORTANT WARNING**
If you set maxattampt to 1 and do NOT whitelist your own IP ranges, you CAN lock yourself out.
Examples of dangerous situations:
* VPN reconnect glitches
* NAT changes on your ISP
* API automation with a typo
* Testing from dynamic IPs
Always whitelist your trusted subnets before using `maxattampt = 1`.

### Whitelisting Trusted / Local IP Subnets
The script uses explicit subnet checks, not CIDR math, to stay compatible and predictable.
Example whitelist logic from the script:
```
# Skip trusted networks
:local trusted false

# 192.168.0.0/16
:if ([:pick $ipinside 0 8] = "192.168.") do={
    :set trusted true
}

# 10.10.0.0/20 -> 10.10.0.0 – 10.10.15.255
:if ([:pick $ipinside 0 6] = "10.10.") do={
    :local thirdOctet [:pick $ipinside 6 [:find $ipinside "." 6]]
    :if ($thirdOctet < 16) do={
        :set trusted true
    }
}
```
How to add your own subnets?
Example: `172.16.5.0/24`
```
:if ([:pick $ipinside 0 7] = "172.16.") do={
    :local thirdOctet [:pick $ipinside 7 [:find $ipinside "." 7]]
    :if ($thirdOctet = 5) do={
        :set trusted true
    }
}
```
Example: single static IP `203.0.113.42`
```
:if ($ipinside = "203.0.113.42") do={
    :set trusted true
}
```
__Tip__: Be explicit. Routers should never guess your intent.

### Address List Output
Blocked IPs appear in:
```
/ip firewall address-list
```
Example:
```
list_failed_attempt  150.107.245.99   timeout=2d23h58m
list_failed_attempt  103.168.234.58   timeout=2d23h58m
```
Each repeated failed login refreshes the timeout — the more they try, the longer they stay jailed.

## Why This Script Exists
MikroTik RouterOS does not provide native brute-force protection for management services. This script fills that gap using only supported, documented features. It is:
* deterministic
* explicit
* upgrade-safe and far less error-prone than “magic” security features

## Disclaimer
Use at your own risk.
Test carefully on non-production routers first.

Always ensure you have (standard precautions):
* console access
* MAC access
* or an out-of-band recovery path

## License
GPL v3

Kick botnets hard. 🔨


