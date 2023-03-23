# Készítse el saját SMTP levélküldő szerverét

## preambulum

Az SMTP közvetlenül vásárolhat szolgáltatásokat a felhőszolgáltatóktól, például:

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Ali felhő e-mail push](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

Saját levelezőszervert is készíthet – korlátlan küldés, alacsony összköltség.

Az alábbiakban lépésről lépésre bemutatjuk, hogyan építsük fel saját levelezőszerverünket.

## Szerver kiválasztása

A saját üzemeltetésű SMTP-kiszolgálóhoz nyilvános IP-cím szükséges a 25-ös, 456-os és 587-es portokkal.

Az általánosan használt nyilvános felhők alapértelmezés szerint blokkolták ezeket a portokat, és munkaparancs kiadásával lehet megnyitni őket, de ez végül is nagyon kellemetlen.

Azt javaslom, hogy olyan gazdagéptől vásároljon, amelyen nyitva vannak ezek a portok, és támogatja a fordított tartománynevek beállítását.

Itt ajánlom [a Contabot](https://contabo.com) .

A Contabo egy müncheni székhelyű tárhelyszolgáltató, amelyet 2003-ban alapítottak rendkívül versenyképes árakkal.

Ha az eurót választja a vásárlás pénznemeként, akkor az ár olcsóbb lesz (egy 8 GB memóriával és 4 CPU-val rendelkező szerver évente kb. 529 jüanba kerül, a kezdeti telepítési díj pedig egy évig ingyenes).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

Rendeléskor vegye figyelembe, hogy `prefer AMD` , és az AMD CPU-val rendelkező szerver jobb teljesítményt fog nyújtani.

A következőkben a Contabo VPS-jét fogom példaként bemutatni, hogy bemutassam, hogyan építs fel saját levelezőszervert.

## Ubuntu rendszerkonfiguráció

Az operációs rendszer itt az Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Ha az ssh-n lévő kiszolgálón megjelenik `Welcome to TinyCore 13!` (ahogy az alábbi ábrán látható), ez azt jelenti, hogy a rendszer még nincs telepítve. Kérjük, válassza le az ssh-t, és várjon néhány percet az újbóli bejelentkezéshez.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

Amikor megjelenik `Welcome to Ubuntu 22.04.1 LTS` , az inicializálás befejeződött, és folytathatja a következő lépésekkel.

### [Opcionális] Inicializálja a fejlesztői környezetet

Ez a lépés nem kötelező.

A kényelem kedvéért az ubuntu szoftver telepítését és rendszerkonfigurálását [a github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) címre tettem.

Futtassa a következő parancsot a telepítéshez egy kattintással.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Kínai felhasználók, kérjük, használja inkább a következő parancsot, és a nyelv, az időzóna stb. automatikusan beáll.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### A Contabo engedélyezi az IPV6-ot

Engedélyezze az IPV6-ot, hogy az SMTP IPV6-címekkel is tudjon e-maileket küldeni.

szerkessze `/etc/sysctl.conf`

Módosítsa vagy adja hozzá a következő sorokat

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Kövesse [a contabo oktatóanyagot: IPv6-kapcsolat hozzáadása a szerverhez](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Szerkessze `/etc/netplan/01-netcfg.yaml` fájlt, adjon hozzá néhány sort az alábbi ábrán látható módon (a Contabo VPS alapértelmezett konfigurációs fájlja már tartalmazza ezeket a sorokat, csak törölje a megjegyzéseket).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

Ezután `netplan apply` , hogy a módosított konfiguráció életbe lépjen.

A sikeres konfiguráció után `curl 6.ipw.cn` segítségével megtekintheti a külső hálózat ipv6-címét.

## A konfigurációs lerakat klónozása műveletek

```
git clone https://github.com/wactax/ops.soft.git
```

## Hozzon létre ingyenes SSL-tanúsítványt a domain nevéhez

A levélküldéshez SSL-tanúsítvány szükséges a titkosításhoz és az aláíráshoz.

[Az acme.sh-t](https://github.com/acmesh-official/acme.sh) használjuk a tanúsítványok generálására.

Az acme.sh egy nyílt forráskódú automatikus tanúsítvány-aláíró eszköz,

Lépjen be az ops.soft konfigurációs raktárba, futtassa `./ssl.sh` , és egy `conf` mappa jön létre **a felső könyvtárban** .

Keresse meg DNS-szolgáltatóját [az acme.sh dnsapi webhelyről](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , szerkessze `conf/conf.sh` fájlt.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Ezután futtassa `./ssl.sh 123.com` `123.com` és `*.123.com` tanúsítványok létrehozásához a domain nevéhez.

Az első futtatás automatikusan telepíti [az acme.sh fájlt](https://github.com/acmesh-official/acme.sh) , és hozzáad egy ütemezett feladatot az automatikus megújításhoz. Láthatod `crontab -l` , van egy ilyen sor a következőképpen.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

A generált tanúsítvány elérési útja valami ilyesmi: `/mnt/www/.acme.sh/123.com_ecc。`

A tanúsítvány megújítása meghívja `conf/reload/123.com.sh` szkriptet, szerkessze ezt a szkriptet, és hozzáadhat olyan parancsokat, mint például `nginx -s reload` a kapcsolódó alkalmazások tanúsítvány-gyorsítótárának frissítéséhez.

## Építsen SMTP szervert chasquid segítségével

[A chasquid](https://github.com/albertito/chasquid) egy Go nyelven írt nyílt forráskódú SMTP-kiszolgáló.

Az ősi levelezőszerver-programok, például a Postfix és a Sendmail helyettesítőjeként a chasquid egyszerűbb és könnyebben használható, valamint a másodlagos fejlesztéshez is könnyebb.

Futtassa `./chasquid/init.sh 123.com` egy kattintással (a 123.com helyére cserélje ki a küldő tartomány nevét).

## E-mail aláírás DKIM konfigurálása

A DKIM-et e-mail-aláírások küldésére használják, hogy megakadályozzák a levelek spamként való kezelését.

A parancs sikeres lefutása után a rendszer felkéri a DKIM-rekord beállítására (lásd alább).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Csak adjon hozzá egy TXT rekordot a DNS-hez (az alábbiak szerint).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Tekintse meg a szolgáltatás állapotát és naplóit

 `systemctl status chasquid` A szolgáltatás állapotának megtekintése.

A normál működés állapota az alábbi ábrán látható

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` vagy `journalctl -xeu chasquid` megtekintheti a hibanaplót.

## Fordított tartománynév-konfiguráció

A fordított tartománynév lehetővé teszi az IP-cím feloldását a megfelelő tartománynévvé.

A fordított tartománynév beállítása megakadályozhatja, hogy az e-maileket spamként azonosítsák.

Amikor a levél megérkezik, a fogadó szerver fordított tartománynévelemzést hajt végre a küldő szerver IP-címén, hogy megbizonyosodjon arról, hogy a küldő szerver rendelkezik-e érvényes fordított tartománynévvel.

Ha a küldő szerver nem rendelkezik fordított tartománynévvel, vagy ha a fordított tartománynév nem egyezik a küldő szerver IP-címével, a fogadó szerver spamként ismerheti fel az e-mailt, vagy elutasíthatja azt.

Látogassa meg [a https://my.contabo.com/rdns webhelyet](https://my.contabo.com/rdns) , és konfigurálja az alábbiak szerint

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

A fordított tartománynév beállítása után ne felejtse el beállítani az ipv4 és ipv6 tartománynév továbbítási felbontását a kiszolgálóhoz.

## Szerkessze a chasquid.conf gazdagépnevét

Módosítsa `conf/chasquid/chasquid.conf` a fordított tartománynév értékére.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Ezután futtassa `systemctl restart chasquid` a szolgáltatás újraindításához.

## A conf biztonsági mentése a git tárolóba

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

Például biztonsági másolatot készítek a conf mappáról a saját github folyamatomra a következőképpen

Először hozzon létre egy privát raktárt

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Lépjen be a conf könyvtárba, és küldje el a raktárba

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Feladó hozzáadása

fuss

```
chasquid-util user-add i@wac.tax
```

Feladó hozzáadható

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Ellenőrizze, hogy a jelszó helyesen van-e beállítva

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

A felhasználó hozzáadása után `chasquid/domains/wac.tax/users` frissül, ne felejtse el beküldeni a raktárba.

## DNS hozzáadása SPF rekord

Az SPF (Sender Policy Framework) egy e-mail-ellenőrzési technológia, amelyet az e-mailes csalások megelőzésére használnak.

Ellenőrzi a levélküldő személyazonosságát azáltal, hogy ellenőrzi, hogy a feladó IP-címe egyezik-e az állítólagos tartománynév DNS-rekordjaival, így megakadályozza, hogy a csalók hamis e-maileket küldjenek.

Az SPF-rekordok hozzáadásával a lehető legnagyobb mértékben megakadályozhatja, hogy az e-maileket spamként azonosítsák.

Ha a domain névszervere nem támogatja az SPF típust, csak adjon hozzá TXT típusú rekordot.

Például a `wac.tax` SPF-je a következő

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF `_spf.wac.tax` számára

`v=spf1 a:smtp.wac.tax ~all`

Ne feledje, hogy itt `include:_spf.google.com` adtam meg, ez azért van, mert később az `i@wac.tax` címet fogom beállítani küldési címként a Google postafiókjában.

## DNS konfiguráció DMARC

A DMARC a (Domain-based Message Authentication, Reporting & Conformance) rövidítése.

Az SPF visszapattanások rögzítésére szolgál (lehet, hogy konfigurációs hibák okozzák, vagy valaki más adja ki magát, hogy spamet küldjön).

TXT rekord hozzáadása `_dmarc` ,

A tartalom a következő

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

Az egyes paraméterek jelentése a következő

### p (Irányelv)

Azt jelzi, hogyan kell kezelni az SPF- (Sender Policy Framework) vagy DKIM- (DomainKeys Identified Mail)-ellenőrzést sikertelen e-maileket. A p paraméter három érték egyikére állítható be:

* nincs: Nem történik semmilyen művelet, csak az ellenőrzés eredménye kerül visszajelzésre a feladónak az e-mailes jelentési mechanizmuson keresztül.
* Karantén: Az ellenőrzésen át nem eső leveleket helyezze a spam mappába, de nem utasítja el közvetlenül.
* elutasít: Közvetlenül elutasítja azokat az e-maileket, amelyek ellenőrzése sikertelen.

### fo (hibabeállítások)

Meghatározza a jelentési mechanizmus által visszaküldött információ mennyiségét. A következő értékek egyikére állítható be:

* 0: Jelentse az összes üzenet érvényesítési eredményét
* 1: Csak azokat az üzeneteket jelentse, amelyek ellenőrzése sikertelen
* d: Csak a tartománynév-ellenőrzési hibákat jelentse
* s: csak az SPF-ellenőrzési hibákat jelenti
* l: Csak a DKIM-ellenőrzési hibákat jelentse

### rua & ruf

* rua (Reporting URI for Aggregate Reports): E-mail cím az összesített jelentések fogadására
* ruf (Reporting URI for Forensic reports): e-mail cím a részletes jelentések fogadására

## Adjon hozzá MX rekordokat az e-mailek Google Mailbe való továbbításához

Mivel nem találtam olyan ingyenes vállalati postafiókot, amely támogatná az univerzális címeket (Catch-All, minden erre a domain névre küldött e-mailt fogadhat, az előtagok korlátozása nélkül), ezért chasquid segítségével minden e-mailt a Gmail postafiókomba továbbítottam.

**Ha van saját fizetős üzleti postafiókja, kérjük, ne módosítsa az MX-et, és hagyja ki ezt a lépést.**

`conf/chasquid/domains/wac.tax/aliases` szerkesztése, átirányítási postafiók beállítása

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` az összes e-mailt jelöli, `i` a küldő felhasználó e-mail címének fent létrehozott előtagja. A levelek továbbításához minden felhasználónak hozzá kell adnia egy sort.

Ezután adja hozzá az MX rekordot (itt közvetlenül a fordított domain név címére mutatok, ahogy az alábbi ábra első sorában látható).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

A konfiguráció befejezése után más e-mail címekkel is küldhet e-maileket `i@wac.tax` és `any123@wac.tax` címekre, hogy megnézze, fogadhat-e e-maileket a Gmailben.

Ha nem, ellenőrizze a chasquid naplót ( `grep chasquid /var/log/syslog` ).

## Küldjön e-mailt az i@wac.tax címre a Google Mail segítségével

Miután a Google Mail megkapta a levelet, természetesen reméltem, hogy az i.wac.tax@gmail.com helyett `i@wac.tax` címmel válaszolok.

Látogasson el [a https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) oldalra, és kattintson a "Másik e-mail cím hozzáadása" lehetőségre.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Ezután írja be a továbbított e-mailben kapott ellenőrző kódot.

Végül beállítható alapértelmezett feladó címként (a válaszadás lehetőségével együtt).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

Ezzel befejeztük az SMTP levelezőszerver létrehozását, és ezzel egyidejűleg a Google Mail szolgáltatást használjuk e-mailek küldésére és fogadására.

## Küldjön teszt e-mailt, hogy ellenőrizze, hogy a konfiguráció sikeres-e

Írja be: `ops/chasquid`

`direnv allow` a függőségek telepítését (a direnv az előző egykulcsos inicializálási folyamatban lett telepítve, és egy hook került hozzáadásra a shellhez)

majd fuss

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

A paraméterek jelentése a következő

* felhasználó: SMTP felhasználónév
* pass: SMTP jelszó
* címzett: címzett

Küldhetsz teszt e-mailt.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Javasoljuk, hogy a Gmailt használja teszt e-mailek fogadására, hogy ellenőrizze, hogy a konfigurációk sikeresek-e.

### TLS szabványos titkosítás

Ahogy az alábbi ábrán is látható, ott van ez a kis zár, ami azt jelenti, hogy az SSL-tanúsítvány sikeresen engedélyezve lett.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Ezután kattintson az "Eredeti e-mail megjelenítése" gombra.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

Amint az alábbi ábrán látható, a Gmail eredeti levelei oldalon megjelenik a DKIM, ami azt jelenti, hogy a DKIM konfiguráció sikeres volt.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Ellenőrizd az eredeti e-mail fejlécében a Received-et, és láthatod, hogy a feladó címe IPV6, ami azt jelenti, hogy az IPV6 is sikeresen konfigurálva van.
