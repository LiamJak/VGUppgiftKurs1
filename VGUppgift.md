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
sudo chmod 700 /Projekt/Gemensamt
sudo setfacl -m g:g_ledare:rwx,g:g_personal:rwx /Projekt/Gemensamt
```
första raden tar bort allas behörigheter förutom roots. andra ger båda grupperna behörighet att skriva, titta och skapa filer och mappar men ingen övrig kan det (förutom root)

sen måste vi göra så att nya filer och undermappar ärver gruppbehörigheten
```
sudo setfacl -d -m u::rwx,g::---,g:g_ledare:rwx,g:g_personal:rwx,m::rwx,o::--- /Projekt/Gemensamt
```
detta gör så att vanliga nya textfiler får normalt läs och skrivrättigheter för båda grupperna, utan körbehörighet

nu ska vi testa att det faktiskt funkar
```
sudo -iu alice
echo "hello world! > /Group/Gemensamt/test.txt
cat /Group/Gemensamt/test.txt
Hello world!
```
Den funkar. vi gick in i alice profil och skapade en test fil. sedan körde vi den och det funkade

nu går vi istället in i bob och ser att han inte kommer åt Ledningsmappen samt kan läsa test filen
```
exit
sudo -ui bob
ls /Projekt/Ledning
ls: cannot open directory ´/Projekt/Ledning´: Permission denied
cat /Projekt/Gemensamt/test.txt
Hello world!
echo "Bob kan också skriva" >> /Projekt/Gemensamt/test.txt
cat /Projekt/Gemensamt/test.txt
Hello world! 
Bob kan också skriva
```

och alice kan skriva och fixa i sin egen Ledningsmapp
```
exit
sudo -iu alice
echo "hello world! > /Group/Ledning/test.txt
cat /Group/Ledning/test.txt
Hello world!
```

<br><br>

**Windows OS**<br>
Skapar grupperna och användarna samt placera in dom rätt.
```
net localgroup g_ledare /add
net localgroup g_personal /add

net user alice /add
net user bob /add

net localgroup g_ledare alice /add
net localgroup g_personal bob /add
```

skapa huvudmappen
```
mkdir C:\Projekt
```

sätter behörighet på mappen
```
icacls C:\Projekt /inheritance:r
icacls C:\Projekt /grant:r "*S-1-5-32-544:(OI)(CI)F" "*S-1-5-18:(OI)(CI)F" "g_ledare:RX" "g_personal:RX"
```
Raden icacls C:\Projekt /inheritance:r stänger av arvet från C:\ och tar bort alla ärvda behörigheter.

Raden icacls C:\Projekt /grant:r ... ger sedan att g_ledare och g_personal har bara rätt att öppna och lista C:\Projekt, utan arv nedåt.

Nu skapar vi nudermapparna och sätter deras behörighet
```
mkdir C:\Projekt\Gemensamt
mkdir C:\Projekt\Ledning

icacls C:\Projekt\Gemensamt /grant:r "g_ledare:(OI)(CI)M" "g_personal:(OI)(CI)M"
icacls C:\Projekt\Ledning /grant:r "g_ledare:(OI)(CI)M"
```
M betyder Modify, alltså ändra. Det ger rätt att läsa, skriva, skapa och ta bort innehåll. Det motsvarar behörigheterna som Linux-mappar med rwx ger.

Då testar vi med alice profil och skapar en test fil i Ledning samt Gemensamt
```
runas /user:.\alice cmd
echo Hello world! > C:\Projekt\Gemensamt\test.txt
type C:\Projekt\Gemensamt\test.txt
Hello world!

echo Alice har tillgang > C:\Projekt\Ledning\ledningstest.txt
type C:\Projekt\Ledning\ledningstest.txt
Alice har tillgang
```

Nu testar vi med bob, om jag kommer in i Ledning och om vi har behörighet i test filen
```
runas /user:.\bob cmd

dir C:\Projekt\Ledning
Access is denied

type C:\Projekt\Gemensamt\test.txt
Hello world!

echo Bob kan ocksa skriva >> C:\Projekt\Gemensamt\test.txt
type C:\Projekt\Gemensamt\test.txt
Hello world!
Bob kan ocksa skriva
```

Vi testar om behörigheten är ärvd
```
icacls C:\Projekt\Gemensamt\test.txt
C:\Projekt\Gemensamt\test.txt LIAMWINDOWS\g_ledare:(I)(M)
LIAMWINDOWS\g_personal:(I)(M)
BUILTIN\Administrators:(I)(F)
NT AUTHORITY\SYSTEM:(I)(F)

Successfully processed 1 files; Failed processing 0 files
```

**Jämförelsen mellan Linux och Windows**

| Fråga | Linux | Windows |
| ----- | ----- | ------- |
| Hur tilldelas rättigheterna? | Ägare, ägargrupp och övriga samt ACL för flera grupper | NTFS använder åtkomstlistor med rättigheter för användare och grupper |
| Vad ärver nya filer i Gemensamt | Standard-ACL ger båda grupperna läs och skrivrättigheter. |	Filen ärver gruppernas behörigheter från mappen genom (OI)(CI).|
| Viktig skillnad i arv | Vanlig chmod ger inte automatiskt motsvarande rättigheter på nya filer. Här används standard-ACL. |Ärftliga NTFS-poster kan föras vidare till nya filer och undermappar|
| Vem blir ägare av en ny fil? | Skaparen | Skaparen |
| Vilken grupp får den nya filen? | Normalt ägarens primära grupp | Ingen enskild "ägargrupp" styr åtkomst, ACL:en gör det |

### Moment C: Spårbarhet & Överlämningsdokukentation

**omfattning**
Dokumentation om behörighetstrukturen i Linux (Ubuntu) Miljön (POSIX-behörigheter och ACL) och Windows Miljön (NTFS ACL med icacls). Detta ska räcka för att man ska kunna bygga om och verifiera utan att ställa frågor.

**Miljööversikt**
| Egenskap | Linux | Windows |
| --- | --- | --- |
| Operativsystem | Ubuntu 25.04 | Windows 11 |
| Datornamn | LiamsUbuntu | LiamWindows |
| Plattform | Oracle Virtualbox | Oracle Virtualbox |
| Rotmap | /Projekt | C:\Projekt |

Kommandon för Windows körs i CMD som administratör och Linux Bash med hjälp utav sudo. Behörighetstesterna körs därefter med alice respektive bobs profil


Krav
| Typ | namn | Detalj |
| --- | --- | --- |
| Grupp | g_ledare | chefer/ledare |
| Grupp | g_personal | Övrig personal |
| Användare | alice | Medlem i g_ledare |
| Användare | bob | Medlem i g_personal |

| Map | g_personal (bob) | g_ledare (alice) |
| --- | --- | --- |
| Projekt/Gemensamt | Läsa och skriva | Läsa och skriva |
| Projekt/Ledning | Ingen åtkomst | Läsa och Skriva |
särkrav: en testfil i Gemensamt ska kunna redigeras av båda. g_personal ska nekas åtkomst till Ledning.

**Uppbygnad eller återställning**
Om mapparna, grupperna eller användarna försvinner kan strukturen återskapas med instruktionerna i Moment B. Kontrollera först vilka delar som finns kvar och gör sedan de steg som saknas. Existernade användare och grupper behöver inte skapas igen.
<br>
Om bara behörigheterna har blivit fel återställs de enligt behörighetskommandona i Moment B. Kontrollera även rättigheterna på existerande filer, eftersom standard-ACL i Linux endast påverkar nya filer och undermappar.

Om du återskapar ska då:
- alice och bob ska kunna läsa pch redogera samma fil i Gemesamt.
- alice ska kunna skapa och läsa filer i Ledining.
- bob ska nekas åtkomst till ledning.
- Nya filer i Gemensamt ska få avsedda ärvda behörigheter.
<br>

**Problem som kan uppstå**
| Problem | Kontroll |
| --- | --- |
| Användare nekas åtkomst till Gemensamt | Kontrollera gruppmedlemskap och behörighter på både Projket, Gemensamt och filen. på Linux (ls -l / för mapp ls -ld) och windows (icalcs) |
| bob kommer åt ledningen | Kontrollera att bob inte tillhör g_ledare och att inga andra rättigheter ger honom åtkomst |
| En ny fil kan inte redigeras av båda | Kontrollera Standard-ACL i linux eller ärvda NTFS-behörigheter i windows. |

Ange kontrollkommandon: id alice (id visar användar-id, primära grupp och alla andra grupper användaren tillhör), id bob, getfacl (ägare för mapp och ägargruppen) i Linux och net localgroup (vilka som är medlemar i den gruppen) g_ledare, net localgroup g_personal, icalcs i Windows.