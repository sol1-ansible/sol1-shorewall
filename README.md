
# Ansible Role: Shorewall

## Description

Ansible role which installs and configures [Shorewall](http://shorewall.org/) and Shorewall6.

## Installation

Install this role in your Ansible roles path.

Example using git:

```bash
git clone <role-repo-url> roles/sol1-shorewall
```


## Requirements

- Ansible 2.10 or better
- `ansible.posix` collection (used by `ansible.posix.sysctl`)
- Shorewall / Shorewall6 5.2.0 or better on destination hosts

Operational safety note:

- The role validates configured `shorewall_interfaces` / `shorewall6_interfaces` names against detected host interfaces.
- If unknown interface names are found, a warning is printed and service restart is automatically skipped for safety.
- Interfaces matching `shorewall_allowed_missing_interface_regex` are treated as transient and allowed to be absent (default: `^(tun|tap|wg|ppp|vti|ip_vti|xfrm|tailscale).*`).

## Role Handlers

Name | Description
--- | ---
`enable shorewall`, `enable shorewall6` | Enables and starts Shorewall / Shorewall 6
`restart shorewall`, `restart shorewall6` | Restarts Shorewall / Shorewall6

## Role Variables

*Note:* The Shorewall (IPv4) variables are prefixed by `shorewall_`, whereas the Shorewall6 (IPv6) variables are prefixed by `shorewall6_`.

Variable | Dictionary / Options
--- | ---
shorewall_run | "false", "true"
shorewall_install_packages | "true", "false"
shorewall_startup | "1" or "0"
shorewall_conf | *this variable uses standard option / value pairs*
shorewall_interfaces | `zone`, `interface`, `options`, `comment`
shorewall_policies | `source`, `dest`, `policy`, `log_level`, `burst_limit`, `connlimit`, `comment`
shorewall_rules | **sections**: `section`, **rules**: `rule`.  For each **rule**: `action`, `source`, `dest`, `proto`, `dest_port`, `source_port`, `original_dest`, `rate_limit`, `user_group`, `mark`, `connlimit`, `time`, `headers`, `switch`, `helper`, `when`, `comment`
shorewall_snat | `action`, `action_param`, `source`, `dest`, `proto`, `port`, `ipsec`, `mark`, `user`, `switch`, `original_dest`, `probability`, `comment`
shorewall_zones | `zone`, `type`, `options`, `options_in`, `options_out`, `comment`
shorewall_hosts | `zone`, `hosts`, `options`, `comment`
shorewall_params | [ `import` | `comment` | `name`, `value` ] **imports are processed first then name/value pairs**
shorewall_actions | `name`, `options`, `description`, `comment`
shorewall_mangle | `action`, `source`, `dest`, `proto`, `dport`, `sport`, `user`, `test`, `comment`
shorewall_providers | `name`, `number`, `mark`, `duplicate`, `interface`, `gateway`, `options`, `copy_iface`, `comment`
shorewall_rtrules | `source`, `dest`, `provider`, `priority`, `mask`, `comment`
shorewall_routes | `provider`, `dest`, `gateway`, `device`, `options`, `comment`
shorewall_stoppedrules | `action`, `source`, `dest`, `proto`, `dest_port`, `source_port`, `comment`
shorewall_tcinterfaces | `interface`, `type`, `in_bandwidth`, `out_bandwidth`
shorewall_tunnels | `type`, `zone`, `gateway`, `gateway_zone`, `comment`



### shorewall_run - Ansible Shorewall role run or not

This allows you to protect against overwriting an existing Shorewall configuration on a host if you no longer wish Ansible to run this role against a specific host.
If this host var is not set to true for a specific host then the Shorewall role will not run against that host.

#### Example

```yaml
shorewall_run: true
```

### shorewall_install_packages - Install package toggle

Controls whether package installation tasks run for Shorewall and Shorewall6.
Allows for faster re-runs if only updating rules on an existing firewall.

- `true` (default): install required Shorewall and Shorewall6 packages
- `false`: skip package installation tasks (useful for faster rules-only updates)

#### Example

```yaml
shorewall_install_packages: false
```


### shorewall_startup - Shorewall startup behaviour

This updates the `/etc/default/shorewall` file's `startup` option to either enable (*"1"*) startup (using the `service` or `systemctl` commands) or disable it (*"0"*).


### shorewall_conf - Shorewall Configuration

Specify values for global Shorewall options in the `/etc/shorewall/shorewall.conf` file. See the Shorewall [shorewall.conf man page](http://shorewall.org/manpages/shorewall.conf.html) for more details.

Each shorewall.conf option may be written in lower-case, such as `ACCEPT_DEFAULT=none
` can be written as `accept_default: "none"` in the variables.

#### Example

```yaml
shorewall_conf:
  verbosity:
    verbosity: "1"
  logging:
    log_verbosity: "2"
    logfile: "/var/log/messages"
  fw_options:
    blacklist: "\"NEW,INVALID,UNTRACKED\""
  packet_disposition:
    blacklist_disposition: "DROP"
```

### shorewall_interfaces - Interfaces

Define the interfaces on the system and optionally associate them with zones in the `/etc/shorewall/interfaces` file. See the Shorewall [interfaces man page](http://www.shorewall.net/manpages/shorewall-interfaces.html) for more details.

#### Example

```yaml
shorewall_interfaces:
  - { zone: net, interface: enp1s0, options: "dhcp,logmartians", comment: "Primary WAN Interface" }
```

### shorewall_policies - Policies

Define high-level policies for connections between zones in the `/etc/shorewall/policies`. See the Shorewall [policy man page](http://www.shorewall.net/manpages/shorewall-policy.html) for more details.

#### Example

```yaml
shorewall_policies:
  - { source: "fw", dest: all, policy: ACCEPT }
  - { source: net, dest: all, policy: DROP }
  - { source: all, dest: all, policy: REJECT, log_level: info }
```


### shorewall_rules - Rules

Specify exceptions to policies, including DNAT and REDIRECT in the `/etc/shorewall/rules` file. See the Shorewall [rules man page](https://shorewall.org/manpages/shorewall-rules.html) for more details.

***WARNING***: Please be sure to include a rule for SSH on the correct port, to avoid locking Ansible - and yourself - out from the remote host.


#### Examples

```yaml
shorewall_rules:
  - section: NEW
    rules:
    - { action: ACCEPT, source: net, dest: "fw", proto: tcp, dest_port: ssh, comment: "Allow SSH Into the firewall" }
    - { action: ACCEPT, source: all, dest: "fw", proto: icmp, dest_port: echo-request, comment: "Allow pings" }
```


### shorewall_snat - SNAT

Define dynamic NAT (Masquerading) and to define Source NAT (SNAT) in the `/etc/shorewall/snat` file. See the Shorewall [snat man page](https://shorewall.org/manpages/shorewall-snat.html) for more details.

#### Example

```yaml
shorewall_snat:
  - { action: SNAT, action_param: "203.0.113.10", source: "10.0.0.0/24", dest: "0.0.0.0/0", proto: "-", port: "-", comment: "SNAT LAN traffic to WAN IP" }
```


### shorewall_zones - Zones

Declare Shorewall zones in the `/etc/shorewall/zones` file. See the Shorewall [zones man page](http://www.shorewall.net/manpages/shorewall-zones.html) for more details.

#### Example

```yaml
shorewall_zones:
  - { zone: fw, type: firewall }
  - { zone: net, type: ipv4, comment: "Public Internet" }
```


### shorewall_hosts - Hosts

Define multiple zones accessed through a single interface in the `/etc/shorewall/hosts` file. See the Shorewall [hosts man page](https://shorewall.org/manpages/shorewall-hosts.html) for more details.

#### Example

```yaml
shorewall_hosts:
  - { zone: net, hosts: "203.0.113.0/24", options: "routeback", comment: "Public subnet on WAN" }
  - { zone: loc, hosts: "10.0.10.0/24", comment: "Internal VLAN" }
```


### shorewall_params - Parameters

Assign any shell variables that you need in the `/etc/shorewall/params` file. See the Shorewall [params man page](https://shorewall.org/manpages/shorewall-params.html) for more details.

#### Example
```yaml
shorewall_params:
  - { import: "/path/file" }
  - { name: "web_server", value: "10.0.0.1" }
  - { name: "some_var", value: "{{ SOME_ANSIBLE_GROUP_VAR }}" }
```


### shorewall_actions - Actions

Define Shorewall actions to allow a symbolic name to be associated with a series of one or more iptables rules `/etc/shorewall/actions` file. See the Shorewall [actions man page](https://shorewall.org/Actions.html) for more details.

#### Example

```yaml
shorewall_actions:
  - { name: "LogDrop", options: "-", description: "Log and drop packets" }
```


### shorewall_mangle - Mangle

Define packets to be marked as a means of classifying them for traffic control or policy routing in the `/etc/shorewall/mangle` file. See the Shorewall [mangle man page](https://shorewall.org/manpages/shorewall-mangle.html) for more details.

#### Example
```yaml
shorewall_mangle:
  - { comment: "Change the DSCP traffic prioritization marking on all traffic from 192.168.3.240 to be remarked as TC-4 bulk." }
  - { action: "DSCP(CS0):T", source: "192.168.3.240/32", dest: "0.0.0.0/0" }
```


### shorewall_providers - Providers

Define additional routing tables, eg for ISP failover connections using Foolsm. `/etc/shorewall/providers` file. See the Shorewall [providers man page](https://shorewall.org/manpages/shorewall-providers.html) for more details.

#### Example
```yaml
shorewall_providers:
  - { name: "nbn", number: "1", mark: "1", interface: "enp1s0", gateway: "detect", options: "track,balance=1" }
  - { name: "fourg", number: "2", mark: "2", interface: "enp2s0", gateway: "detect", options: "track,balance=2" }
```


### shorewall_rtrules - RTRules

Define entries to cause traffic to be routed to one of the providers listed in shorewall_providers. `/etc/shorewall/rtrules` file. See the Shorewall [rtrules man page](https://shorewall.org/manpages/shorewall-rtrules.html) for more details.

#### Example
```yaml
shorewall_rtrules:
  - { source: "10.0.0.0/24", provider: "nbn", priority: "1510", comment: "Route LAN to NBN connection" }
  - { source: "lo", provider: "nbn", priority: "1510", comment: "Route firewall traffic to NBN connection" }

  - { source: "10.0.0.0/24", provider: "fourg", priority: "2010", comment: "Route LAN to 4G connection with lower route weight than NBN connection for failover" }
  - { source: "lo", provider: "fourg", priority: "2010", comment: "Route firewall traffic to 4G connection with lower route weight than NBN connection for failover" }
```


### shorewall_routes - Routes

Define routes to be added to provider routing tables in the `/etc/shorewall/routes` file.  See the Shorewall [routes man page](https://shorewall.org/manpages/shorewall-routes.html) for more details.

#### Example

```yaml
shorewall_routes:
  - { provider: "nbn", dest: "0.0.0.0/0", gateway: "203.0.113.1", device: "enp1s0", options: "persistent" }
  - { provider: "fourg", dest: "0.0.0.0/0", gateway: "198.51.100.1", device: "enp2s0", options: "persistent" }
```


### shorewall_stoppedrules - Stopped Rules

Define the hosts that are accessible when the firewall is stopped or is being stopped in the `/etc/shorewall/stoppedrules` file.  See the Shorewall [stoppedrules man page](https://shorewall.org/manpages/shorewall-stoppedrules.html) for more details.

#### Example

```yaml
shorewall_stoppedrules:
  - { action: ACCEPT, source: enp1s0 }
  - { action: ACCEPT, dest: enp1s0 }
```


### shorewall_tcinterfaces - TCInterfaces

Define the interfaces that are subject to simple traffic shaping in the `/etc/shorewall/tcinterfaces` file.  See the Shorewall [tcinterfaces man page](https://shorewall.org/manpages/shorewall-tcinterfaces.html) for more details.

#### Example

```yaml
shorewall_tcinterfaces:
  - { interface: "enp1s0", type: "external", out_bandwidth: "50984kbit:2048kb:1250ms", comment: "Limit upload to 50Mbit to avoid hitting NBN policer" }
```


### shorewall_tunnels - Tunnels

Define VPN connections with endpoints on the firewall in the `/etc/shorewall/tunnels` file.  See the Shorewall [tunnels man page](https://shorewall.org/manpages/shorewall-tunnels.html) for more details.

#### Example

```yaml
shorewall_tunnels:
  - { type: ipsec, zone: net, gateway: "0.0.0.0/0", gateway_zones: "vpn1,vpn2" }
```


 
## Example Playbook

```yml
- hosts: all
  roles:
     - sol1-shorewall
```

## Changelog

### v3.0
- Drop legacy compatibility paths for old Shorewall versions and standardize on current behavior.
- Remove deprecated masq handling and migrate role flow to snat-only semantics.
- Add interface preflight safety checks to detect invalid configured interfaces.
- Skip service restarts when interface validation fails, with explicit end-of-run warnings.
- Add transient interface allowlist support for common dynamic links (tun/tap/wg/ppp/vti/ip_vti/xfrm/tailscale).
- Add optional-file cleanup so removed vars purge stale generated config files.
- Simplify package management flow to install-toggle control only.
- Remove obsolete CI/test artifacts and other dead weight.
- Refresh README and metadata: fix inaccuracies/typos, expand examples, and document current requirements.

### v2.0
- Massive rewrite and cleanup of Templates, and Vars files and Tasks for both Shorewall and Shorewall6
- Add templates for missing Shorewall(6) config files eg tcinterfaces rtrules
- Fix up broken Shorewall6 tasks
- Add working dead mans switch for both Shorewall and Shorewall6
- Shorewall(6) Templates now generate templates like the factory Shorewall templates instead of wall of text
- Update Readme with new functions added

### v1.0.3

- Added: The `shorewall_rules` has an added option `when` for each rule, which acts similar to Ansible's `when` statement and allows rules to be conditional.
- Added: role variable `shorewall_tunnels` for use with VPNs.

### v1.0

- Added: `ipset` as a package dependency;
- Added: role variable `shorewall_conf`, allowing each option in the shorewall.conf file to be defined;
- *Changed:* The default for `shorewall_interface` now detects the default network interface rather than fixed at `eth0` (though `eth0` is still a fall-back default);
- **Removed:** role variables: `shorewall_verbosity`, `shorewall_log_verbosity`.  Use the `shorewall_conf` role variable to configure these instead.




## Author
* Lindsay Harvey
* Sol1 Pty Ltd (http://sol1.com.au)
* [Michael Green](http://myatus.com)
* [Simon Bärlocher](https://sbaerlocher.ch)
* Farhad Shahbazi
* Sascha Biberhofer
 
## License

This project is under the MIT License. See the LICENSE file for the full license text.

## Copyright

- Copyright (c) 2022 Lindsay Harvey
- Copyright (c) 2022 Sol1 Pty Ltd
- Copyright (c) 2017 Michael Green
- Copyright (c) 2016 Simon Bärlocher

