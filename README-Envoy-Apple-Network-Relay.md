# Apple Managed Network Relay mit Envoy, OPNsense und Proxmox

Diese Anleitung beschreibt den Aufbau eines selbst betriebenen Apple Managed Network Relay auf Basis von:

- Proxmox VE
- Debian 13 als virtuelle Maschine
- Envoy Proxy 1.39.1
- OPNsense mit IPv4 Destination NAT
- HTTP/2 und HTTP/3 (QUIC)
- Let's Encrypt DNS-01 über Infomaniak
- Microsoft Intune zur Verteilung des Apple-Relayprofils

Das Beispiel verwendet zwei voneinander getrennte Relay-Instanzen:

| Instanz | DMZ | interne Adresse | öffentlicher Port | interner Port |
|---|---|---|---:|---:|
| Adults | VLAN 30 | `172.16.30.10` | `59443` TCP/UDP | `443` TCP/UDP |
| Kids | VLAN 40 | `172.16.40.10` | `58443` TCP/UDP | `443` TCP/UDP |

Beide Instanzen verwenden denselben öffentlichen Hostnamen `relay.example.ch`. Die unterschiedlichen öffentlichen Ports bestimmen, an welche VM OPNsense den Verkehr weiterleitet.

> **Hinweis:** Hohe Ports sind kein Sicherheitsmechanismus. Die eigentliche Absicherung erfolgt durch aktuelle Software, DMZ-Isolation, restriktive Firewallregeln und optional Clientauthentifizierung.

## Funktionsweise

```text
Apple-Gerät
    |
    | HTTPS / HTTP/3
    v
relay.example.ch:59443 oder :58443
    |
    v
OPNsense Destination NAT
    |
    +--> Adults-VM 172.16.30.10:443
    |
    +--> Kids-VM   172.16.40.10:443
            |
            v
        Envoy MASQUE
            |
            v
        Internet über getrennte DMZ/Zenarmor-Policy
```

## Voraussetzungen

- Öffentlicher DNS-Name für das Relay
- Öffentliche IPv4-Adresse
- OPNsense mit separatem DMZ-Interface oder VLAN
- Proxmox-Bridge, welche das jeweilige VLAN transportiert
- Funktionierende DNS-Auflösung aus der DMZ
- Apple-Gerät mit unterstützter Betriebssystemversion und MDM-Profil
- Für HTTP/3 muss TCP **und UDP** freigegeben werden

Diese Anleitung setzt voraus, dass VLANs, Routing und DNS bereits eingerichtet sind.

## 1. Debian-VM in Proxmox erstellen

Empfohlene VM-Einstellungen:

| Einstellung | Wert |
|---|---|
| Betriebssystem | Debian 13 amd64 netinst |
| Machine | q35 |
| CPU | 2 Cores, Typ `host` |
| RAM | 2048 MiB |
| Disk | 16 GiB, SCSI, VirtIO SCSI single |
| Netzwerk | VirtIO |
| MTU | 1492 bei PPPoE |
| QEMU Guest Agent | aktiviert |

Im Debian-Installer werden nur **SSH server** und **standard system utilities** benötigt. Eine Desktopumgebung ist nicht erforderlich.

Pakete installieren:

```bash
apt update
apt install -y ca-certificates curl python3-pip certbot ethtool qemu-guest-agent tcpdump
systemctl enable --now qemu-guest-agent
```

## 2. Statische Netzwerkkonfiguration

Beispiel für die Adults-VM in `/etc/network/interfaces`:

```ini
# Network configuration for EnvoyDMZadults

source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

auto ens18
iface ens18 inet static
        address 172.16.30.10/24
        gateway 172.16.30.1
        dns-nameservers 10.0.1.2 10.0.1.3
        dns-search example.ch
        mtu 1492
        post-up /usr/sbin/ethtool -K ens18 gro off

iface ens18 inet6 static
        address 2001:db8:30::10/64
        gateway fe80::1
        pre-up /sbin/sysctl -w net.ipv6.conf.ens18.autoconf=0
        pre-up /sbin/sysctl -w net.ipv6.conf.ens18.accept_ra=0
```

Die IPv6-Adressen sind Beispiele und müssen durch das eigene Präfix sowie die tatsächliche Link-Local-Adresse des OPNsense-Gateways ersetzt werden.

Nach einem Neustart prüfen:

```bash
ip -br address
ip -4 route
ip -6 route
ping -4 -c 2 1.1.1.1
ping -6 -c 2 2606:4700:4700::1111
```

## 3. QUIC-Puffer des Kernels

Datei `/etc/sysctl.d/90-envoy-quic.conf` erstellen:

```ini
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.core.netdev_max_backlog = 8192
```

Anwenden und prüfen:

```bash
sysctl --system
sysctl net.core.rmem_max
sysctl net.core.wmem_max
sysctl net.core.netdev_max_backlog
```

## 4. VirtIO-GRO deaktivieren

In der getesteten VM führte `generic-receive-offload` zu verworfenen QUIC-Datagrammen. GRO wurde daher deaktiviert:

```bash
ethtool -K ens18 gro off
ethtool -k ens18 | grep generic-receive-offload
```

Erwartet:

```text
generic-receive-offload: off
```

Die Zeile in `/etc/network/interfaces` macht die Einstellung dauerhaft:

```ini
post-up /usr/sbin/ethtool -K ens18 gro off
```

## 5. Envoy installieren

Beispiel für Envoy 1.39.1:

```bash
curl -fL \
  https://github.com/envoyproxy/envoy/releases/download/v1.39.1/envoy-1.39.1-linux-x86_64 \
  -o /usr/local/bin/envoy
chmod 755 /usr/local/bin/envoy
/usr/local/bin/envoy --version
```

Vor produktiver Verwendung sollten Download und Prüfsumme mit der offiziellen Envoy-Release-Seite verglichen werden.

## 6. Certbot-Plugin für Infomaniak

Das Infomaniak-Plugin installieren:

```bash
python3 -m pip install --break-system-packages certbot-dns-infomaniak
certbot plugins | grep -A4 infomaniak
```

Die API-Zugangsdaten in `/etc/infomaniak.ini` ablegen:

```ini
dns_infomaniak_token = HIER_API_TOKEN_EINTRAGEN
```

Datei schützen:

```bash
chmod 600 /etc/infomaniak.ini
```

Zertifikat ausstellen:

```bash
certbot certonly \
  --authenticator dns-infomaniak \
  --dns-infomaniak-credentials /etc/infomaniak.ini \
  --dns-infomaniak-propagation-seconds 300 \
  --cert-name relay.example.ch \
  --key-type ecdsa \
  -d relay.example.ch
```

Prüfen:

```bash
certbot certificates
```

> API-Token, private Schlüssel und vollständige Firewall-Exporte dürfen niemals in ein öffentliches Git-Repository hochgeladen werden.

## 7. Vollständige Envoy-Konfiguration

Verzeichnis erstellen:

```bash
mkdir -p /etc/envoy
```

Die folgende Konfiguration ist das Adults-Beispiel für den öffentlichen Port `59443`. Envoy selbst lauscht intern auf TCP und UDP 443.

Datei `/etc/envoy/envoy.yaml`:

```yaml
admin:
  address:
    socket_address:
      address: 127.0.0.1
      port_value: 9901

layered_runtime:
  layers:
  - name: static_layer
    static_layer:
      "envoy.reloadable_features.FLAGS_envoy_quiche_reloadable_flag_quic_enable_mtu_discovery_at_server": true
  - name: admin_layer
    admin_layer: {}

overload_manager:
  refresh_interval: 0.25s
  resource_monitors:
  - name: envoy.resource_monitors.global_downstream_max_connections
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.resource_monitors.downstream_connections.v3.DownstreamConnectionsConfig
      max_active_downstream_connections: 4096

static_resources:
  listeners:

  - name: listener_http3
    address:
      socket_address:
        address: "::"
        port_value: 443
        protocol: UDP
        ipv4_compat: true

    socket_options:
    - description: QUIC receive buffer
      level: 1
      name: 8
      int_value: 16777216
      state: STATE_PREBIND
    - description: QUIC send buffer
      level: 1
      name: 7
      int_value: 16777216
      state: STATE_PREBIND

    udp_listener_config:
      downstream_socket_config:
        max_rx_datagram_size: 2048
        prefer_gro: false
      quic_options:
        idle_timeout: 3600s
        quic_protocol_options:
          max_packet_length: 1440

    filter_chains:
    - transport_socket:
        name: envoy.transport_sockets.quic
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.quic.v3.QuicDownstreamTransport
          downstream_tls_context:
            common_tls_context:
              alpn_protocols:
              - h3
              tls_certificates:
              - certificate_chain:
                  filename: "/etc/letsencrypt/live/relay.example.ch/fullchain.pem"
                private_key:
                  filename: "/etc/letsencrypt/live/relay.example.ch/privkey.pem"

      filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: h3_relay
          codec_type: HTTP3
          common_http_protocol_options:
            idle_timeout: 3600s
          http3_protocol_options:
            allow_extended_connect: true
          stream_idle_timeout: 900s
          request_timeout: 0s
          delayed_close_timeout: 10s
          upgrade_configs:
          - upgrade_type: CONNECT
          - upgrade_type: CONNECT-UDP

          route_config:
            name: masque_routing
            virtual_hosts:
            - name: relay
              domains:
              - "*"
              response_headers_to_add:
              - header:
                  key: alt-svc
                  value: 'h3=":59443"; ma=2592000'
              routes:
              - match:
                  path: "/.well-known/masque"
                direct_response:
                  status: 200
                  body:
                    inline_string: '{"proxy-ports":[59443]}'
                response_headers_to_add:
                - header:
                    key: content-type
                    value: application/json
                  append_action: OVERWRITE_IF_EXISTS_OR_ADD
              - match:
                  connect_matcher: {}
                route:
                  cluster: dynamic_forward_proxy_cluster
                  timeout: 0s
                  upgrade_configs:
                  - upgrade_type: CONNECT
                    connect_config: {}
                  - upgrade_type: CONNECT-UDP
                    connect_config: {}
              - match:
                  prefix: "/"
                direct_response:
                  status: 200
                  body:
                    inline_string: "MASQUE Relay Active\n"

          http_filters:
          - name: envoy.filters.http.dynamic_forward_proxy
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.dynamic_forward_proxy.v3.FilterConfig
              dns_cache_config:
                name: dynamic_forward_proxy_cache_config
                dns_lookup_family: ALL
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  - name: listener_tcp
    address:
      socket_address:
        address: "::"
        port_value: 443
        protocol: TCP
        ipv4_compat: true

    filter_chains:
    - transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
          common_tls_context:
            tls_params:
              tls_minimum_protocol_version: TLSv1_3
              tls_maximum_protocol_version: TLSv1_3
            alpn_protocols:
            - h2
            tls_certificates:
            - certificate_chain:
                filename: "/etc/letsencrypt/live/relay.example.ch/fullchain.pem"
              private_key:
                filename: "/etc/letsencrypt/live/relay.example.ch/privkey.pem"

      filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: h2_relay
          codec_type: AUTO
          common_http_protocol_options:
            idle_timeout: 3600s
          http2_protocol_options:
            allow_connect: true
          stream_idle_timeout: 900s
          request_timeout: 0s
          delayed_close_timeout: 10s
          upgrade_configs:
          - upgrade_type: CONNECT
          - upgrade_type: CONNECT-UDP

          route_config:
            name: masque_routing_tcp
            virtual_hosts:
            - name: relay
              domains:
              - "*"
              response_headers_to_add:
              - header:
                  key: alt-svc
                  value: 'h3=":59443"; ma=2592000'
              routes:
              - match:
                  path: "/.well-known/masque"
                direct_response:
                  status: 200
                  body:
                    inline_string: '{"proxy-ports":[59443]}'
                response_headers_to_add:
                - header:
                    key: content-type
                    value: application/json
                  append_action: OVERWRITE_IF_EXISTS_OR_ADD
              - match:
                  connect_matcher: {}
                route:
                  cluster: dynamic_forward_proxy_cluster
                  timeout: 0s
                  upgrade_configs:
                  - upgrade_type: CONNECT
                    connect_config: {}
                  - upgrade_type: CONNECT-UDP
                    connect_config: {}
              - match:
                  prefix: "/"
                direct_response:
                  status: 200
                  body:
                    inline_string: "MASQUE Relay Active\n"

          http_filters:
          - name: envoy.filters.http.dynamic_forward_proxy
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.dynamic_forward_proxy.v3.FilterConfig
              dns_cache_config:
                name: dynamic_forward_proxy_cache_config
                dns_lookup_family: ALL
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
  - name: dynamic_forward_proxy_cluster
    connect_timeout: 10s
    lb_policy: CLUSTER_PROVIDED
    cluster_type:
      name: envoy.clusters.dynamic_forward_proxy
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.clusters.dynamic_forward_proxy.v3.ClusterConfig
        dns_cache_config:
          name: dynamic_forward_proxy_cache_config
          dns_lookup_family: ALL
```

Für die Kids-Instanz müssen alle Vorkommen von `59443` durch `58443` ersetzt werden. Außerdem sind Hostname, IP-Adressen und DMZ-Zuordnung anzupassen.

Konfiguration prüfen:

```bash
envoy --mode validate -c /etc/envoy/envoy.yaml
```

Die Warnung zu `allow_extended_connect` weist darauf hin, dass diese Envoy-Funktion als Work in Progress gekennzeichnet ist. Die Konfiguration kann trotzdem gültig sein, sollte aber nach Envoy-Updates erneut getestet werden.

## 8. systemd-Service

Datei `/etc/systemd/system/envoy.service`:

```ini
[Unit]
Description=Envoy Proxy for Apple Managed Network Relay
Documentation=https://www.envoyproxy.io/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStartPre=/usr/local/bin/envoy --mode validate -c /etc/envoy/envoy.yaml
ExecStart=/usr/local/bin/envoy -c /etc/envoy/envoy.yaml --service-cluster h3_relay
Restart=on-failure
RestartSec=5s
LimitNOFILE=1048576

NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictSUIDSGID=true
LockPersonality=true
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

Aktivieren:

```bash
systemctl daemon-reload
systemctl enable --now envoy
systemctl status envoy --no-pager -l
ss -lntup | grep ':443'
```

## 9. Automatische Zertifikatserneuerung

Deploy-Hook `/etc/letsencrypt/renewal-hooks/deploy/restart-envoy.sh`:

```sh
#!/bin/sh
/usr/bin/systemctl restart envoy
```

Aktivieren:

```bash
chmod 750 /etc/letsencrypt/renewal-hooks/deploy/restart-envoy.sh
systemctl enable --now certbot.timer
systemctl is-enabled certbot.timer
systemctl is-active certbot.timer
```

## 10. OPNsense-Aliase

Beispiel:

| Alias | Typ | Inhalt |
|---|---|---|
| `Envoy_DMZAdults_v4` | Host | `172.16.30.10` |
| `Envoy_DMZKids_v4` | Host | `172.16.40.10` |
| `Net_DMZAdults_ipv6` | Network | eigenes Adults-/64 |
| `Net_DMZKids_ipv6` | Network | eigenes Kids-/64 |

Alle DMZ-Präfixe müssen zusätzlich im Alias für sämtliche internen IPv6-Netze enthalten sein.

## 11. OPNsense Destination NAT

Adults:

```text
Interface:            WAN
Version:              IPv4
Protocol:             TCP/UDP
Source:               any
Destination:          WAN address
Destination port:     59443
Redirect target IP:   Envoy_DMZAdults_v4
Redirect target port: 443
```

Kids:

```text
Interface:            WAN
Version:              IPv4
Protocol:             TCP/UDP
Source:               any
Destination:          WAN address
Destination port:     58443
Redirect target IP:   Envoy_DMZKids_v4
Redirect target port: 443
```

Für beide Regeln muss eine passende WAN-Passregel vorhanden sein. TCP wird für HTTP/2 benötigt, UDP für HTTP/3.

### Hinweis zu eingehendem IPv6

Das oben beschriebene Port-Mapping ist IPv4 Destination NAT. Bei nativem IPv6 wird normalerweise nicht auf unterschiedliche interne Hosts übersetzt. Ein gemeinsamer AAAA-Record kann deshalb nicht allein anhand der Ports `58443` und `59443` auf zwei verschiedene VMs zeigen.

Mögliche Lösungen sind getrennte Hostnamen/AAAA-Records, ein zusätzlicher IPv6-Frontend-Proxy oder zunächst ausschließlich eingehendes IPv4. Der ausgehende Verkehr der Envoy-VMs kann trotzdem dual-stack über IPv4 und IPv6 erfolgen.

### Option: dynamische WAN-IP mit DynDNS (ungetestet)

> **Hinweis:** Diese Variante wurde in der beschriebenen Umgebung nicht praktisch getestet. Sie ist als mögliche Erweiterung für Internetanschlüsse mit wechselnder öffentlicher IPv4-Adresse dokumentiert.

Apple Network Relay verwendet in der Konfiguration eine Relay-URL mit einem DNS-Namen. Deshalb sollte sich grundsätzlich auch ein per DynDNS aktualisierter Hostname verwenden lassen. Die `mobileconfig` verweist weiterhin auf denselben Namen, beispielsweise:

```xml
<!-- Adults -->
<string>https://relay.example.ch:59443/</string>

<!-- Kids -->
<string>https://relay.example.ch:58443/</string>
```

Bei einem Wechsel der öffentlichen IPv4-Adresse aktualisiert OPNsense beziehungsweise `os-ddclient` den A-Record von `relay.example.ch`. Die Destination-NAT-Regeln bleiben unverändert, da sie weiterhin an die aktuelle WAN-Adresse gebunden sind. Auch das TLS-Zertifikat muss nicht wegen des IP-Wechsels ersetzt werden, weil es für den DNS-Namen ausgestellt ist.

Voraussetzungen und Einschränkungen:

- Der Anschluss benötigt eine von außen erreichbare öffentliche IPv4-Adresse. Hinter CGNAT funktioniert das eingehende Portforwarding nicht.
- Der DynDNS-A-Record sollte eine kurze TTL besitzen, beispielsweise 300 Sekunden.
- TCP und UDP müssen für die externen Relay-Ports erreichbar sein.
- Nach einem IP-Wechsel können bestehende Relay-Verbindungen kurzzeitig ausfallen, bis DNS-Caches die neue Adresse übernommen haben.
- Ein AAAA-Record darf nur veröffentlicht werden, wenn eingehendes IPv6 zuverlässig zum richtigen Relay gelangt. Ein veralteter oder falsch gerouteter AAAA-Record kann dazu führen, dass Apple-Geräte den nicht erreichbaren IPv6-Pfad bevorzugen.
- Die Zertifikatserneuerung per DNS-01 ist unabhängig von der jeweils aktuellen WAN-IP, sofern die DNS-API und die zuständige DNS-Zone erreichbar bleiben.

Vor einem produktiven Einsatz sollten mindestens ein erzwungener WAN-IP-Wechsel, die anschließende Aktualisierung des A-Records sowie neue HTTP/2- und HTTP/3-Verbindungen von einem externen Mobilfunknetz getestet werden.

## 12. DMZ-Firewallregeln

Empfohlene Reihenfolge pro Envoy-DMZ:

1. DNS IPv4 ausschließlich zu den internen DNS-Servern erlauben
2. DNS IPv6 ausschließlich zu den internen DNS-Servern erlauben
3. Zugriff auf `This Firewall` blockieren
4. RFC1918-Netze über IPv4 blockieren
5. sämtliche internen IPv6-Netze blockieren
6. RFC4193/ULA blockieren
7. Internetzugriff für das jeweilige DMZ-Netz erlauben

Bei kopierten Regeln unbedingt kontrollieren, dass **Interface** und **Source** zur neuen DMZ gehören. Eine kopierte Kids-Regel darf beispielsweise nicht weiterhin `DMZAdults net` als Quelle verwenden.

## 13. Lokale Funktionstests

HTTP/2:

```bash
curl --http2 \
  --resolve relay.example.ch:443:127.0.0.1 \
  https://relay.example.ch/.well-known/masque
```

HTTP/3:

```bash
curl --http3-only \
  --resolve relay.example.ch:443:127.0.0.1 \
  --max-time 10 \
  https://relay.example.ch/.well-known/masque
```

Erwartete Adults-Antwort:

```json
{"proxy-ports":[59443]}
```

## 14. Externer Test

Auf der VM:

```bash
tcpdump -ni ens18 'tcp port 443 or udp port 443'
```

Von einem externen Anschluss:

```text
https://relay.example.ch:59443/.well-known/masque
```

Ein Browser verwendet beim ersten Aufruf möglicherweise HTTP/2. Der tatsächliche Relaybetrieb sollte anschließend auch eingehende UDP-Pakete zeigen.

## 15. Apple-Mobileconfig für Intune

Der zentrale Teil des Relay-Payloads:

```xml
<key>Relays</key>
<array>
    <dict>
        <key>HTTP2RelayURL</key>
        <string>https://relay.example.ch:59443/</string>
        <key>HTTP3RelayURL</key>
        <string>https://relay.example.ch:59443/</string>
    </dict>
</array>
```

Für Kids wird Port `58443` verwendet. Adults- und Kids-Profil müssen jeweils eigene `PayloadIdentifier` und `PayloadUUID` besitzen.

Beispiel für On-Demand:

```xml
<key>OnDemandRules</key>
<array>
    <dict>
        <key>Action</key>
        <string>Disconnect</string>
        <key>SSIDMatch</key>
        <array>
            <string>Mein Heim-WLAN</string>
        </array>
    </dict>
    <dict>
        <key>Action</key>
        <string>Connect</string>
    </dict>
</array>
```

Das Profil kann in Intune als benutzerdefiniertes Apple-Konfigurationsprofil verteilt werden. Zunächst sollte immer nur ein Testgerät zugewiesen werden.

## 16. Datagram-Drops überwachen

Zähler zurücksetzen:

```bash
curl -sS -X POST http://127.0.0.1:9901/reset_counters
```

Prüfen:

```bash
curl -s http://127.0.0.1:9901/stats |
grep 'downstream_rx_datagram_dropped'
```

Logs prüfen:

```bash
journalctl -u envoy --since '-10 min' --no-pager |
grep -Ei 'dropped|error|critical'
```

Erwartet:

```text
listener.[__]_443.udp.downstream_rx_datagram_dropped: 0
```

Falls der Zähler in einer Proxmox-VM trotz großer Socket-Puffer steigt, zuerst VirtIO-GRO kontrollieren und testweise deaktivieren.

## 17. Proxmox-Startreihenfolge

Beispiel:

```text
Order 1: OPNsense, anschließend ausreichend Startzeit
Order 2: DNS-Server
Order 3: weitere Infrastruktur
Order 4: Envoy-VMs
```

Beispielbefehl:

```bash
qm set VMID --onboot 1 --startup order=4,up=10,down=20
```

## 18. Sicherheitshinweise

- Envoy gehört in eine isolierte DMZ.
- Die DMZ darf nicht auf interne IPv4- oder IPv6-Netze zugreifen.
- Envoy, Debian und Proxmox regelmäßig aktualisieren.
- API-Tokens ausschließlich in Dateien mit restriktiven Rechten speichern.
- Keine privaten Schlüssel, echten API-Tokens oder vollständigen Firewall-Exporte veröffentlichen.
- Den Admin-Port `9901` nur auf `127.0.0.1` binden.
- Alte Container nach erfolgreicher Migration deaktivieren oder entfernen.
- Profile vor breiter Intune-Zuweisung immer auf einem einzelnen Gerät testen.
- Nach Envoy-Upgrades HTTP/2, HTTP/3, CONNECT und CONNECT-UDP erneut testen.

## 19. Fehlerdiagnose

### Endpoint funktioniert lokal, aber extern nicht

- öffentlichen DNS-A-Record prüfen
- OPNsense Destination NAT kontrollieren
- WAN-Regel für TCP und UDP prüfen
- auf der VM mit `tcpdump` kontrollieren

### HTTP/2 funktioniert, HTTP/3 nicht

- UDP-Port in NAT und Firewall prüfen
- `ss -lntup | grep ':443'`
- QUIC-Drop-Zähler prüfen
- GRO auf VirtIO deaktivieren
- MTU und `max_packet_length` kontrollieren

### Nach dem Klonen ist die VM nicht erreichbar

- VLAN auf Proxmox, UniFi und OPNsense anlegen
- vor dem ersten parallelen Start IP-Adresse und Hostname ändern
- keine geklonte VM mit derselben IP im selben VLAN starten
- geklonte Certbot-Konfigurationen kontrollieren; parallele Erneuerungen desselben Zertifikats sollten bewusst geplant werden

### Certbot meldet NXDOMAIN für den TXT-Record

- autoritative Nameserver direkt abfragen
- prüfen, ob Änderungen im DNS-Portal zusätzlich gespeichert werden müssen
- Propagationszeit erhöhen
- keine wiederholten Zertifikatsanfragen in kurzer Zeit erzeugen

## Lizenz und Haftung

Diese Anleitung ist ein technisches Beispiel. Netzbereiche, Präfixe, Ports, DNS-Namen und Sicherheitsanforderungen müssen an die eigene Umgebung angepasst werden. Vor Änderungen sollte ein aktuelles Backup der OPNsense- und Proxmox-Konfiguration erstellt werden.
