# Individuell Fördjuppning <br>
## Av: <br> Liam Jakbosson ISCX2026 <br>

### Moment A: Avancerad Nätverksanalys & Trafikflöden

![Datatrafikflöde](Bilder/Trafikflöde.png) <br>

**Klienten tar reda på serverns IP-Adress** <br>
Först o främst måste klienten veta vart den ska. Med hjälp av en DNS-Server så skickar klienten ett domännamn (det som är lättläsligt för gemeneman), i detta fallet ``chasacademy.se``, som DNS översätter till en ipadress (IP: 75.2.60.5) och skickar tillbaka, den skickar normalt fram och sedan tillbaks med UDP. Nu så vet vi vart ska, tack vare DNS, och kan börja skicka trafiken. 

**Klienten skickar trafiken till sin gateway**<br>
Klienten har adressen ``192.168.10.10/24``. Den jämför desitanitonens IP-Adress med sitt eget subnät och ser att webbservern befinner sig utanför det. Därför skickas trafiken till standardgatewayen ``192.168.10.1`` Vilket är det samma som routern. Man kan likna det med dörren ut på ett hus, ut till internetet.

Klienten användre ARP för att ta reda på gatewayens MAC-Adress, om inte den redan finns sparad.

Den första ramen mot webbservern innehåller då:
| Fäl | Värde |
| --- | ---- |
| Avsändar-MAC| 08:00:27:97:60:cd |
| Mottagar-MAC | Gatewayens MAC-Adress (VM saknar Gateway) |
| Avsändar-IP | 192.168.10.10 |
| Mottagar-IP | 75.2.60.5 |
| Avsändarport |Tillfälig port |
| Mottagarport | 443 för HTTPS |

Motagarens MAC-Adress tillhör gatewayn, men mottagarens IP-Adress tillhör webbservern.

**Switch och router skickar vidare trafiken**<br>
Switchen använder motagarens MAC-Adress för att skicka ramen till rätt port i det lokala nätverket. Routern tar bort den inkommande länklagerramen och läser destinationens IP-Adress för att välja nästa våg. det skapar en ny ram för nästa länk/router. På Ethernet för ramen routerns utgående MAC-Adress som avsändaren och nästa hopps MAC-Adress som mottagare.

vid vanlig internetåtkomst översätter NAT klientens avsändar-IP till en publik IP-Adress. PAT kan även ändra avsändarporten och håller reda på vilken klient svarstrafiken ska tillbaka till.

**Trafiken når webbservern**
Routarna på vägen anävnder desinations IP-Adressen för att föra paketet vidare, IP-adresserma är normalt oförändrade längs vägen förutom när exempelvis NAT används.

Vid vanlig HTTPS över TCP upprättar klinenten först en TCP-anslutning till serverns port, alltså 443. Därefter sker en TLS handshake och HTTP-trafken skickas krypterat.

**Svaret kommer tillbaka**
 Servern svara med sin IP-adress som avsändaren och port 443 som avsändarporten. klientens tillfälliga port blir destinationsporten. Om NAT/PAT används så översätter routern svaret tillbka till klientens privata IP-Adress och ursprungliga prot med hjälp av sin NAT-Tabell.

 **Koppling till TCP/IP-Modellen:**

 | skikt|  Vad som händer |
 | --- | --- |
 | Applikationsslagret | DNS slår upp namnet. HTTP används för webbförfrågan, skyddas av TLS vid HTTPS|
 | Transportslagret | TCP/UDP och portar identifierar vilken tjänst eller anslutning trafiken hör till.
 | Internetlagret | IP-Adresser används för adressering och routing mellan nätverk |
 | Nätverksåtkomstlagret | MAC-Adresser används för leverans över den lokala länken. Nya ramar skapas när trafiken passerar en router. |

### Moment B: Jämförande OS- och behörighetsanalys

**Linux OS (Ubuntu)**
Först skappar vi grupperna
```
sudo groupadd g_ledare
sudo groupadd g_personal
```
och användarna
```
sudo adduser alice
sudo adduser bob
```
För att detta ska gå igenom måste vi använda root-behörigheter/administrationsrättigheter (sudo)

Därefter lägger vi till användarna i grupperna
```
sudo usermod -G g_ledare alice
sudo usermod -G g_personal bob
```
samma här använder vi root-behörigheter men även flaggan -G för att den ska ange användarnas extra grupper <br>

Nu skapar vi mappstrukturen
```
sudo mkdir -p /Projekt/Gemensamt
sudo mkdir /Projekt/Ledning
```
Använder vi root och -p för att det inte ska vara bara Gemensant som skapas utan också Projekt. sen i andra kommandoraden så behövs inte -p för att projekt finns redan <br>

Nu ger vi behörigheter med hjälp av ACL
```
sudo chmod 700 /Projekt
sudo setfacl -m g:g_ledare:rx,g:g_personal:rx /Projekt
```
först, behörighet endast för root, sen behörighet till båda grupperna att skriva och lsita mappen.<br><br>
sedan Ledningsmappen
```
sudo chown root:g_ledare /Projekt/Ledning
sudo chmod 770 /Projekt/Ledning
```
här gör vi g_ledare till mappens ägargruppen och andra raden så att endast root och medlemmarna i g_ledare har tillgång.

och nu Gemensamt mappen
```
sudo chmod 700 /projekt/Gemensamt
sudo setfacl -m g:g_ledare:rwx,g:g_personal:rwx /Projekt/Gemensamt
```
första raden tar bort allas behörigheter förutom roots. andra ger båda grupperna behörighet att skriva, titta och skapa filer och mappar men ingen övrig kan det (förutom root)

sen måste vi göra så att nya filer och undermappar ärver gruppbehörigheten
```
sudo setfacl -d -m u::rwx,g::---,g:g_ledare:rwx,g:g_personal:rwx,m::rwx,o::--- /Projekt/Gemensamt
```
detta gör så att vanliga nya textfiler får normalt läs och skrivrättigheter för båda grupperna, utan körbehörighet