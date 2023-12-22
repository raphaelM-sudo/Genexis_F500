# Anleitung für Root-Zugang und ändern des DNS
Stand: 22.12.2023, Genexis Pure F500, Firmware: PURE-F500-GNX-4.3.6.110-R 

## 1. JUCI Admin
Man kann sich mit dem Benutzernamen: "admin" und dem Passwort: "admin" auf dem JUCI Panel (http://192.168.1.1) des Modems anmelden. #security
(Falls die Zugangsdaten falsch sind: Ich musste mein Modem erst kürzlich auf die Werkseinstellungen zurücksetzen, d.h. kann es sein, dass das notwendig ist.)

![JUCI Admin Credentials](https://imgur.com/jaLbzsz.png "JUCI Admin Credentials")

## 2. SSH Key generieren
Es ist möglich sich über SSH, als root-User anzumelden, allerdings muss man vorher einen Schlüssel erzeugen.

### Linux
Um einen neuen RSA-Schlüssel unter Linux zu erstellen, muss der Befehl `ssh-keygen -t rsa` ausgeführt werden.
Man kann bei allen Optionen die Eingabetaste drücken. Optional kann man ein Passwort wählen, welches dann beim Aufbau der SSH-Verbindung abgefragt wird.

![Linux RSA Key](https://imgur.com/EpRbUnf.png "Linux RSA Key")

### Windows (PuTTYgen)
Putty ist ein SSH-Client für Windows und kann [hier](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html) heruntergeladen werden.

Mit dem PuTTY Installer kommt auch PuTTYgen.
Zum Erstellen eines SSH-RSA Schlüssels muss PuTTYgen geöffnet werden.

![PuTTY Key Generator](https://imgur.com/TuM3tFu.png "PuTTY Key Generator")
 - Die Bit-Anzahl rechts unten auf `3072` setzen.
 - "Generate" klicken.
 - Den Mauszeiger über die Fläche unterhalb des Fortschrittbalkens bewegen, bis dieser voll ist.

![PuTTY Key Generator](https://imgur.com/Zjaadi1.png "PuTTY Key Generator")
 - Optional kann in den Feldern "Key passphrase" und "Confirm passphrase" ein Passwort für den Schlüssel gesetzt werden.
 - "Save private key" klicken und den privaten Schlüssel an einem sicheren Ort speichern.
 - Den Public-Key aus dem Textfeld vollständig kopieren, in einen Texteditor einfügen und ebenfalls speichern - nicht "Save public key" klicken!

## 3. SSH in JUCI einrichten
Mit dem admin-Benutzer ist es nun möglich unter http://192.168.1.1/#!/settings-management-dropbear den generierten, öffentlichen SSH-RSA-Schlüssel zu hinterlegen.

![Dropbear SSH](https://imgur.com/rdA10YE.png "Dropbear SSH")
 - Unter dem Punkt "Accepted SSH keys" auf "Add" klicken.

![Dropbear SSH](https://imgur.com/eCdFziK.png "Dropbear SSH")
 - Den Text des Public-Keys entweder einfügen, oder die Datei (unter Linux befindet sie sich in `~/.ssh/id_rsa.pub`) auswählen.
 - "Apply" klicken.

## 4. Per SSH verbinden
### Linux
Der Befehl dazu lautet: `ssh -i ~/.ssh/id_rsa root@192.168.1.1`.

Unter Linux ist es noch notwendig folgenden Eintrag in der `~/.ssh/config` vorzunehmen, da das Modem den ssh-rsa Cypher verwendet:

```
Host 192.168.1.1
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedKeyTypes +ssh-rsa
```

### Windows (PuTTY)
Zum Verbinden PuTTY (nicht PuTTYgen) öffnen.

![PuTTY](https://imgur.com/FvcRpLg.png "PuTTY")
 - Unter Connection > SSH > Auth > Credentials den Pfad zum Private-Key angeben.

![PuTTY](https://imgur.com/ygGVGiF.png "PuTTY")
 - Unter Connection > Data "root" als Auto-Login-Usernamen angeben.

![PuTTY](https://imgur.com/vC03vS9.png "PuTTY")
 - Unter Session `192.168.1.1` als IP-Adresse angeben.
 - Einen Namen für die Verbindung im Feld unter "Saved Sessions" eingeben.
 - "Save" klicken - die Verbindung kann in Zukunft einfach ausgewählt werden.
 - "Open" klicken.

Die SSH-Verbindung sollte nun erfolgreich und man als root-User angemeldet sein:

![SSH](https://imgur.com/RJlR9qw.png "SSH")

## 5. DNSmasq DNS-Rolle deaktivieren
Folgende Befehle in GenXOS eingeben:
```
service dnsmasq stop
uci set dhcp.@dnsmasq[0].localuse="0"
uci set dhcp.@dnsmasq[0].port="0"
uci commit dhcp
service dnsmasq restart
```

## 6. Custom DNS über DHCP
Mein Raspberry Pi ist unter `192.168.1.31` erreichbar. Diese IP auf die eure abändern.
Ich habe zusätzlich den Cloudflare DNS `1.1.1.1` als Failsafe hinzugefügt.
```
uci -q delete network.wan.dns
uci add_list network.wan.dns="192.168.1.31"
uci add_list network.wan.dns="1.1.1.1"
uci set network.wan.peerdns="0"
uci commit network
service network restart
```

## 7. Optional: Firewall-Regel Gäste-Netzwerk
Damit man im Gästenetzwerk auf den DNS-Server im LAN Zugriff hat, muss folgende Regel erstellt werden:
```
uci set firewall.guest_rule_local_dns="rule"
uci set firewall.guest_rule_local_dns.name="Allow local DNS Queries"
uci set firewall.guest_rule_local_dns.src="guest"
uci set firewall.guest_rule_local_dns.dest="lan"
uci set firewall.guest_rule_local_dns.dest_port="53"
uci set firewall.guest_rule_local_dns.dest_ip="192.168.1.31"
uci set firewall.guest_rule_local_dns.target="ACCEPT"
uci set firewall.guest_rule_local_dns.proto="tcp udp"
uci set firewall.guest_rule_local_dns.family="ipv4"
uci commit firewall
service firewall restart
```
IP wie immer abändern.

# 🎉 Done!

![](https://imgur.com/ftOscRA.jpeg)

Mal eingeloggt, kann man das root-Passwort mit dem Befehl: `passwd root` ändern, wenn man nicht mehr mittels RSA-Key verbinden möchte.

Ich habe versucht die Anleitung möglichst einfach zu halten und wollte das root-Passwort cracken, damit der Schritt des RSA-Key-Erstellens entfällt. Jedoch hat Hashcat mit der RockYou Passwortliste und allen Kleinbuchstaben bis inkl. 8 Zeichen zu keinem Ergebnis geführt. Der Hash ist folgender: `$1$hixkj06D$465iCCMxkKbE6OW9NbcOV1`. Vielleicht hat ja jemand mehr Erfolg ;)

Auf der Suche nach dem Passwort bin ich auch auf folgenden Forenpost gestoßen, der ebenfalls interessant sein könnte: [IOPSYS EG400 - ein Modem vom selben Hersteller](https://forum.openwrt.org/t/iopsys-eg400-possible-to-run-up-to-date-openwrt/20673)

Mein nächstes Ziel ist es, auf dem F500 das originale OpenWRT zu installieren.
