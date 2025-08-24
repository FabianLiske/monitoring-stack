```bash
scp .env faba@192.168.10.100:~/monitoring/.env
```

```yaml
monitoring_spec:
  version: 1
  notes:
    - "Primär: Prometheus + Grafana. Ergänzend möglich: InfluxDB via Telegraf."
    - "SNMP nach Möglichkeit v3 (AuthPriv). Für UniFi: API via Unpoller."
    - "Uptime Kuma → /metrics aktivieren, von Prometheus scrapen."
    - "Cloudflare Free: Analytics via GraphQL/Exporter (nicht Echtzeit)."
    - "Platzhalter wie <host>, <ip>, <token> später ersetzen."
    - "Tapo P110 ist explizit out-of-scope (bereits vorhanden)."

  # Einheitliche Definitionen der Metrik-Gruppen (semantische 'Bundles')
  metric_groups:
    system_basic:        {desc: "CPU, Load, RAM, Swap, Uptime, FS, Disk-IO, Net-IF"}
    linux_rpi:           {desc: "RasPi: CPU-Temp, Throttling, Voltage, Clocks"}
    docker_containers:   {desc: "Container CPU/RAM/Net/IO je Container"}
    nginx:               {desc: "Nginx active/reading/writing/waiting, conns, reqs"}
    http_probe:          {desc: "HTTP Status, Latenz, TLS-Infos (Blackbox http)"}
    icmp_probe:          {desc: "Ping RTT, Packet Loss (Blackbox icmp)"}
    tcp_probe:           {desc: "TCP Connect Latenz/Up (Blackbox tcp)"}
    vpn_openvpn:         {desc: "Clients, Bytes, Sessions (Status-File/Exporter)"}
    vpn_wireguard:       {desc: "Peers, Rx/Tx-Bytes je Peer"}
    dns_pihole:          {desc: "Queries total, blocked, clients, upstream"}
    mqtt_mosquitto:      {desc: "Clients, Msg/s, Bytes, $SYS-Topics"}
    qbittorrent:         {desc: "Torrents, Rates, Session-Stats (API/Exporter)"}
    modem_dsl:           {desc: "DSL/VDSL Sync, SNR, Attenuation, Errors, Resyncs"}
    router_core:         {desc: "CPU/RAM, ifMIB traffic/errors, conntrack/NAT (optional)"}
    switch_ports:        {desc: "Port up/down, speed, throughput, errors/discards"}
    poe:                 {desc: "PoE per Port, total budget/used"}
    wifi_unifi_ap:       {desc: "AP health, clients, traffic, airtime/channel usage"}
    ipmi_sensors:        {desc: "Temps, Fans, Voltages, PSU/Power (BMC/IPMI)"}
    zfs:                 {desc: "Pool health, capacity, ARC size/hit, vdev stats"}
    truenas:             {desc: "TrueNAS system/datasets via SNMP/API"}
    smart_disks:         {desc: "Drive temps, SMART attrs, reallocated, power-on-hrs"}
    windows_os:          {desc: "CPU, Memory, Disk, NIC (PerfCounter)"}
    nvidia_gpu:          {desc: "GPU util, mem, temp, power (nvidia-smi/DCGM)"}
    proxmox_host:        {desc: "VM CPU/RAM/State, host bridges, storage (API)"}
    kuma_metrics:        {desc: "Uptime Kuma checks, durations, status, cert age"}
    cloudflare_analytics:{desc: "Requests, bandwidth, cache hit, codes, threats, geo"}

  # Welche Exporter/Erheber gehören zu welcher Gruppe (mit Default-Ports & Hinweisen)
  exporters_catalog:
    node_exporter:          {port: 9100,  groups: [system_basic],                 hint: "Linux host agent"}
    rpi_exporter:           {port: 9101,  groups: [linux_rpi],                    hint: "oder Textfile via vcgencmd"}
    cadvisor:               {port: 8080,  groups: [docker_containers],            hint: "cAdvisor http; ggf. 8080/metrics"}
    windows_exporter:       {port: 9182,  groups: [windows_os],                   hint: "Windows PerfCounter"}
    nginx_exporter:         {port: 9113,  groups: [nginx],                        hint: "requires nginx stub_status"}
    blackbox_exporter:      {port: 9115,  groups: [http_probe, icmp_probe, tcp_probe], hint: "module http_2xx/icmp/tcp_connect"}
    pihole_exporter:        {port: 9617,  groups: [dns_pihole],                   hint: "Pi-hole API"}
    mosquitto_exporter:     {port: 9234,  groups: [mqtt_mosquitto],               hint: "$SYS topics"}
    openvpn_exporter:       {port: 9176,  groups: [vpn_openvpn],                  hint: "reads status file"}
    wireguard_exporter:     {port: 9586,  groups: [vpn_wireguard],                hint: "wg show dump"}
    snmp_exporter:          {port: 9116,  groups: [modem_dsl, router_core, switch_ports, poe], hint: "SNMP v2c/v3; generator.yml"}
    unifi_poller_prom:      {port: 9130,  groups: [wifi_unifi_ap, switch_ports],  hint: "read from UniFi Controller API"}
    ipmi_exporter:          {port: 9290,  groups: [ipmi_sensors],                 hint: "BMC host:623 → exporter host"}
    zfs_exporter:           {port: 9134,  groups: [zfs],                          hint: "arcstats/zpool iostat"}
    truenas_via_snmp:       {port: 161,   groups: [truenas],                      hint: "net-snmp service on NAS"}
    smart_via_telegraf:     {port: null,  groups: [smart_disks],                  hint: "smartctl → Influx or textfile"}
    proxmox_exporter:       {port: 9221,  groups: [proxmox_host],                 hint: "Proxmox API (token/user)"}
    nvidia_smi_exporter:    {port: 9400,  groups: [nvidia_gpu],                   hint: "Linux/Win; requires drivers"}
    dcgm_exporter:          {port: 9400,  groups: [nvidia_gpu],                   hint: "Alternative for NVIDIA GPUs"}
    kuma_builtin_metrics:   {port: 3001,  groups: [kuma_metrics],                 hint: "Uptime Kuma /metrics"}
    cloudflare_exporter:    {port: 8080,  groups: [cloudflare_analytics],         hint: "GraphQL Analytics; token scopes"}

  # Dein Inventar als Geräte-Liste mit geplanten Gruppen und Erhebern
  devices:

    # -------- Netzwerk --------
    - id: modem-vigor167
      kind: modem
      host: <ip_modem>
      labels: {site: home, vendor: draytek, model: vigor167, role: wan}
      collectors:
        - {exporter: snmp_exporter, profile: "dsl+ifmib", groups: [modem_dsl, switch_ports]}
        - {exporter: blackbox_exporter, module: icmp, target: <ip_isp_gateway>, groups: [icmp_probe]}
      dashboards: [network/wan, snmp/dsl]
      prereqs: ["SNMP v3 aktivieren", "SNMP OIDs für DSL/VDSL im generator.yml ergänzen"]

    - id: router-edgerouter12
      kind: router
      host: <ip_router>
      labels: {site: home, vendor: ubiquiti, model: edgerouter12, role: core-router}
      collectors:
        - {exporter: snmp_exporter, profile: "edgerouter+ifmib", groups: [router_core, switch_ports]}
        - {exporter: blackbox_exporter, module: icmp, target: <ip_router>, groups: [icmp_probe]}
        - {exporter: node_exporter, optional: true, groups: [system_basic], hint: "nur falls Agent auf EdgeOS"}
        - {exporter: textfile, optional: true, groups: [router_core], hint: "conntrack/NAT via cron script"}
      dashboards: [network/router, snmp/interfaces]
      prereqs: ["SNMP v3 auf EdgeRouter", "optional: conntrack script"]

    - id: switch-unifi-flex-mini
      kind: switch
      host: <ip_unifi_flex_mini>
      labels: {site: home, vendor: ubiquiti, model: usw-flex-mini, role: access}
      collectors:
        - {exporter: unifi_poller_prom, controller: <ip_unifi_controller>, groups: [switch_ports]}
      dashboards: [unifi/switch-ports]
      prereqs: ["Unifi Controller erreichbar, Unpoller mit API-Keys"]

    - id: switch-brocade-icx7250-48p
      kind: switch
      host: <ip_icx7250>
      labels: {site: home, vendor: brocade, model: icx7250-48p, role: distribution}
      collectors:
        - {exporter: snmp_exporter, profile: "icx+ifmib+poe+entity", groups: [switch_ports, poe]}
        - {exporter: snmp_exporter, profile: "entity-sensor", groups: [system_basic], hint: "CPU/RAM/Temp/Fans sofern verfügbar"}
        - {exporter: blackbox_exporter, module: icmp, target: <ip_icx7250>, groups: [icmp_probe]}
      dashboards: [snmp/poe, snmp/interfaces, snmp/hardware]
      prereqs: ["SNMP v3", "POWER-ETHERNET-MIB im generator.yml"]

    - id: ap-unifi-ac-lite
      kind: access_point
      host: <ip_unifi_ap>
      labels: {site: home, vendor: ubiquiti, model: uap-ac-lite, role: wifi}
      collectors:
        - {exporter: unifi_poller_prom, controller: <ip_unifi_controller>, groups: [wifi_unifi_ap]}
      dashboards: [unifi/ap, unifi/clients]
      prereqs: ["Unpoller angebunden an Controller"]

    # -------- SBCs / Stacks --------
    - id: rpi5-main
      kind: sbc
      host: <ip_rpi5>
      labels: {site: home, hw: rpi5, os: ubuntu, role: monitoring+services}
      collectors:
        - {exporter: node_exporter, groups: [system_basic]}
        - {exporter: rpi_exporter, groups: [linux_rpi]}
        - {exporter: cadvisor, groups: [docker_containers]}
        - {exporter: kuma_builtin_metrics, port: 3001, groups: [kuma_metrics]}
        - {exporter: nginx_exporter, groups: [nginx], hint: "Nginx Proxy Manager stub_status"}
        - {exporter: pihole_exporter, groups: [dns_pihole]}
        - {exporter: mosquitto_exporter, groups: [mqtt_mosquitto]}
        - {exporter: blackbox_exporter, module: http_2xx, target: "https://pihole.<domain>", groups: [http_probe]}
        - {exporter: blackbox_exporter, module: icmp, target: <ip_rpi5>, groups: [icmp_probe]}
      services_http:
        - {name: "NPM admin", url: "https://npm.<domain>/"}
        - {name: "Grafana",   url: "https://grafana.<domain>/"}
        - {name: "UptimeKuma",url: "https://status.<domain>/"}
      dashboards: [linux/host, docker/containers, nginx/overview, pihole/main, mqtt/main, kuma/overview]
      prereqs: ["nginx stub_status", "Pi-hole API token falls nötig"]

    - id: rpi4-dmz
      kind: sbc
      host: <ip_rpi4_dmz>
      labels: {site: home, hw: rpi4, os: ubuntu, role: dmz}
      collectors:
        - {exporter: node_exporter, groups: [system_basic]}
        - {exporter: rpi_exporter, groups: [linux_rpi]}
        - {exporter: cadvisor, groups: [docker_containers]}
        - {exporter: nginx_exporter, groups: [nginx], hint: "NPM in DMZ"}
        - {exporter: blackbox_exporter, module: http_2xx, target: "https://vaultwarden.<domain>", groups: [http_probe]}
      services_http:
        - {name: "Vaultwarden", url: "https://vaultwarden.<domain>/"}
        - {name: "Hugo site",  url: "https://based.<domain>/"}
      dashboards: [linux/host, docker/containers, nginx/overview]
      prereqs: ["Vaultwarden Prometheus optional: ENABLE_PROMETHEUS=true"]

    - id: rpi4-lite
      kind: sbc
      host: <ip_rpi4_lite>
      labels: {site: home, hw: rpi4, role: generic}
      collectors:
        - {exporter: node_exporter, groups: [system_basic]}
        - {exporter: rpi_exporter, groups: [linux_rpi]}
      dashboards: [linux/host]

    - id: rpi3-pivpn
      kind: sbc
      host: <ip_rpi3_pivpn>
      labels: {site: home, hw: rpi3, role: vpn}
      collectors:
        - {exporter: node_exporter, groups: [system_basic]}
        - {exporter: wireguard_exporter, optional: true, groups: [vpn_wireguard]}
        - {exporter: openvpn_exporter,  optional: true, groups: [vpn_openvpn]}
        - {exporter: blackbox_exporter, module: tcp_connect, target: "<wan_ip>:<wg_port>", groups: [tcp_probe]}
      dashboards: [linux/host, vpn/wireguard, vpn/openvpn]

    # -------- Server / Hypervisor / NAS --------
    - id: srv-supermicro-h12ssl-proxmox
      kind: hypervisor
      host: <ip_proxmox>
      labels: {site: home, vendor: supermicro, board: h12ssl-i, role: proxmox}
      collectors:
        - {exporter: node_exporter, groups: [system_basic]}
        - {exporter: proxmox_exporter, groups: [proxmox_host]}
        - {exporter: ipmi_exporter, bmc_host: <ip_bmc_h12>, groups: [ipmi_sensors]}
        - {exporter: blackbox_exporter, module: icmp, target: <ip_proxmox>, groups: [icmp_probe]}
      dashboards: [proxmox/host, ipmi/hardware, linux/host]
      prereqs: ["Proxmox API token", "BMC User (read-only)"]

    - id: nas-supermicro-h11ssl-truenas
      kind: nas
      host: <ip_truenas_h11>
      labels: {site: home, vendor: supermicro, board: h11ssl-i, os: truenas-scale}
      collectors:
        - {exporter: node_exporter, optional: true, groups: [system_basic]}
        - {exporter: truenas_via_snmp, groups: [truenas]}
        - {exporter: zfs_exporter, groups: [zfs]}
        - {exporter: ipmi_exporter, bmc_host: <ip_bmc_h11>, groups: [ipmi_sensors]}
        - {exporter: smart_via_telegraf, optional: true, groups: [smart_disks]}
      dashboards: [truenas/overview, zfs/arc, ipmi/hardware]

    - id: nas-msi-x99a-truenas
      kind: nas
      host: <ip_truenas_x99>
      labels: {site: home, vendor: msi, board: x99a-sli-plus, os: truenas-scale}
      collectors:
        - {exporter: truenas_via_snmp, groups: [truenas]}
        - {exporter: zfs_exporter, groups: [zfs]}
        - {exporter: smart_via_telegraf, optional: true, groups: [smart_disks]}
        - {exporter: node_exporter, optional: true, groups: [system_basic]}
      dashboards: [truenas/overview, zfs/arc]

    - id: pc-asrock-b850m-win11
      kind: workstation
      host: <ip_winpc>
      labels: {site: home, vendor: asrock, os: windows11, gpu: rtx5080}
      collectors:
        - {exporter: windows_exporter, groups: [windows_os]}
        - {exporter: nvidia_smi_exporter, optional: true, groups: [nvidia_gpu]}
      dashboards: [windows/host, nvidia/gpu]

    - id: vm-endeavouros-primary
      kind: vm
      host: <ip_primary_vm>
      labels: {site: home, os: endeavour, role: desktop-vm, gpu_passthrough: 1080ti}
      collectors:
        - {exporter: node_exporter, groups: [system_basic]}
        - {exporter: nvidia_smi_exporter, optional: true, groups: [nvidia_gpu]}
      dashboards: [linux/host, nvidia/gpu]

    # -------- Cloud / Public --------
    - id: cloudflare-account
      kind: cloud
      host: api.cloudflare.com
      labels: {site: cloud, provider: cloudflare, plan: free}
      collectors:
        - {exporter: cloudflare_exporter, groups: [cloudflare_analytics], token: "<cf_token>", zones: ["<zone1>", "<zone2>"]}
      dashboards: [cloudflare/overview]
      prereqs: ["API Token (Analytics:Read), Rate-Limits beachten, nicht Echtzeit"]

  # Optionale Standard-Scrape-Jobs (Template) – wird bei Bedarf konkretisiert
  prometheus_job_templates:
    - name: node_exporter
      job: |
        - job_name: 'node'
          static_configs: [{ targets: ['<host>:9100'] }]

    - name: snmp_exporter
      job: |
        - job_name: 'snmp'
          static_configs: [{ targets: ['<host>'] }]
          metrics_path: /snmp
          params:
            module: ['<module>']   # z.B. 'if_mib','edgerouter','brocade_poe','dsl'
          relabel_configs:
            - source_labels: [__address__]
              target_label: __param_target
            - source_labels: [__param_target]
              target_label: instance
            - target_label: __address__
              replacement: '<snmp_exporter_host>:9116'

    - name: blackbox_http
      job: |
        - job_name: 'blackbox-http'
          metrics_path: /probe
          params: { module: [http_2xx] }
          static_configs: [{ targets: ['https://example.org/'] }]
          relabel_configs:
            - source_labels: [__address__]
              target_label: __param_target
            - target_label: __address__
              replacement: '<blackbox_host>:9115'
            - source_labels: [__param_target]
              target_label: instance

    - name: unifi_poller
      job: |
        - job_name: 'unpoller'
          static_configs: [{ targets: ['<unpoller_host>:9130'] }]

    - name: proxmox_exporter
      job: |
        - job_name: 'proxmox'
          static_configs: [{ targets: ['<proxmox_exporter_host>:9221'] }]

    - name: ipmi_exporter
      job: |
        - job_name: 'ipmi'
          static_configs: [{ targets: ['<ipmi_exporter_host>:9290'] }]

    - name: zfs_exporter
      job: |
        - job_name: 'zfs'
          static_configs: [{ targets: ['<host>:9134'] }]

    - name: windows_exporter
      job: |
        - job_name: 'windows'
          static_configs: [{ targets: ['<host>:9182'] }]

    - name: kuma_metrics
      job: |
        - job_name: 'uptime-kuma'
          static_configs: [{ targets: ['<kuma_host>:3001'] }]

    - name: cloudflare_exporter
      job: |
        - job_name: 'cloudflare'
          static_configs: [{ targets: ['<cf_exporter>:8080'] }]

    - name: nginx_exporter
      job: |
        - job_name: 'nginx'
          static_configs: [{ targets: ['<nginx_exporter_host>:9113'] }]

    - name: pihole_exporter
      job: |
        - job_name: 'pihole'
          static_configs: [{ targets: ['<host>:9617'] }]

    - name: mosquitto_exporter
      job: |
        - job_name: 'mosquitto'
          static_configs: [{ targets: ['<host>:9234'] }]

    - name: wireguard_exporter
      job: |
        - job_name: 'wireguard'
          static_configs: [{ targets: ['<host>:9586'] }]

    - name: openvpn_exporter
      job: |
        - job_name: 'openvpn'
          static_configs: [{ targets: ['<host>:9176'] }]

  # (Optional) Sinnvolle Folderstruktur in Grafana
  grafana_folders:
    - name: "00 - Platform"
      dashboards: ["prometheus/self", "kuma/overview", "cloudflare/overview"]
    - name: "10 - Network"
      dashboards: ["snmp/interfaces", "snmp/poe", "unifi/ap", "unifi/switch-ports", "network/wan", "nginx/overview"]
    - name: "20 - Hosts"
      dashboards: ["linux/host", "windows/host", "ipmi/hardware", "proxmox/host"]
    - name: "30 - Storage"
      dashboards: ["truenas/overview", "zfs/arc"]
    - name: "40 - Services"
      dashboards: ["pihole/main", "mqtt/main", "docker/containers"]
```