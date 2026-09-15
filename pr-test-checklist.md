# PR teszt checklist

## 0. Elokeszites

- [ ] Tesztadatbazis vagy kulon tesztkornyezet hasznalata
- [ ] `pnpm install`
- [ ] `pnpm db:generate`    
- [ ] `pnpm lint`
- [ ] `pnpm build`
- [ ] Ellenorizd, hogy ne keruljenek be veletlenul a `... 2.ts` duplikalt fajlok
- [ ] Tesztfiokok letrehozasa:
  - [ ] normal tag
  - [ ] trainee
  - [ ] driver
  - [ ] dispatcher
  - [ ] inspector
  - [ ] instructor
  - [ ] admin
  - [ ] alacsonyabb es magasabb rangu admin

## 1. Dashboard es profil

- [ ] Profil szerkesztese, handle es lathatosag modositasa
- [ ] Profil megnyitasa publikus, tagoknak lathato es privat allapotban
- [ ] Elrejtett statisztikak tenyleg nem jelennek meg
- [ ] Profil osszehasonlitasa ket felhasznaloval
- [ ] Sajat adatok exportalasa
- [ ] Csatolt Discord- es Roblox-fiok levalasztasa
- [ ] Az utolso identitas levalasztasat a rendszer megakadalyozza
- [ ] Beallitasok oldalon a profil, badge-ek, ertesitesek es fiok tab mukodik
- [ ] Heti es havi cel letrehozasa, modositasa es torlese
- [ ] Tul nagy cel ertekenek elutasitasa
- [ ] Duty-emlekezteto idopontok beallitasa es kikapcsolasa
- [ ] Nyelv mentese es ujratoltes utan is megmarad

## 2. Duty es menetrend

- [ ] Driver-, dispatcher- es inspector-duty inditasa/leallitasa
- [ ] Duty ujrakezdese sajat, korabban megszakadt szolgatalaton belul
- [ ] Lejart departure grace utan az ujrakezdes tiltott
- [ ] Foglalt vagy idoben utkozo vonat kivalasztasa tiltott
- [ ] Inspector szolgaltat vonatparral es vonatpar nelkul
- [ ] Inspector booking letrehozasa, torlese es dupla booking tiltasa
- [ ] Duty announcement letrejon Discordon
- [ ] Duty lezarasakor ugyanaz az announcement szerkesztodik
- [ ] Discord-hiba nem akadalyozza a duty muveletet
- [ ] Napi duty reset lezarja az aktiv duty-kat es announcementeket
- [ ] Heti duty minimum szamolasa minden relevans aggal
- [ ] Duty streak helyes, az aktualis het nem szamit bele
- [ ] Teljesen kivett het nem tori meg a streaket
- [ ] Heti es havi cel progressze futó duty kozben is frissul

## 3. Admin es jogosultsagok

- [ ] Bulk ban, unban es logout mukodik
- [ ] Senki nem tudja sajat magat modositani
- [ ] Alacsonyabb rangu admin nem tud magasabb rangu tagot modositani
- [ ] Reszleges bulk eredmenynel kulon latszik a siker es az elutasitas
- [ ] A sikertelenul kihagyott kivalasztas megmarad
- [ ] 50 cimzettnel nagyobb bulk keres elutasitva
- [ ] Bulk ban indok nelkul elutasitva
- [ ] Driver-, dispatcher- es inspector-suspension kulon mukodik
- [ ] Suspension alatt az erintett duty nem indithato
- [ ] A masik duty-ag tovabbra is hasznalhato
- [ ] Lejarat utan a suspension automatikusan hatastalan
- [ ] Suspension feloldasa audit logba kerul
- [ ] Minden uj permission flag lathato es helyesen vedett
- [ ] Jogosulatlan felhasznalo kozvetlen API-hivassal sem kerulheti meg az UI-t

## 4. Wiki es support assistant

- [ ] Wiki oldal szerkesztese es mentese
- [ ] Revision letrejon valodi tartalmi modositasnal
- [ ] Valtozatlan mentes nem hoz letre felesleges revisiont
- [ ] Revision visszaallitasa mukodik
- [ ] Visszaallitas uj revisionkent jelenik meg
- [ ] Wiki review statusza es lejarata helyes
- [ ] Support assistant wiki-talalatot is hasznal
- [ ] Wiki-talalat nelkuli kerdesre nem talal ki valaszt
- [ ] Wiki gap riport csak megfelelo ismetlodes utan jelez
- [ ] A mar lefedett kerdes kulon jelolest kap
- [ ] Machine translation mukodik wiki, training es exam tartalmon
- [ ] Kodblokkok, linkek es Markdown-szerkezet nem serul
- [ ] Translation-szolgaltato hibaja eseten az eredeti szoveg jelenik meg

## 5. Vizsgak es training

- [ ] Training programme-ok listazasa es szerkesztese
- [ ] Trainee szerep es programme-tagsag helyesen mukodik
- [ ] Material nelkuli programme is tud closing examet inditani
- [ ] Closing exam slot regisztracio, lemondas es tokenes link
- [ ] Napi closing-exam vote: reggeli igen/nem szavazas
- [ ] Napi closing-exam vote: ora kivalasztasa
- [ ] Dontetlennel a korabbi ora nyer
- [ ] Szavazat nelkuli naphoz nem keszul slot
- [ ] A slot automatikusan megnyilik
- [ ] No-show csak committed hour vote eseten buntet
- [ ] Vizsga lezarasakor eredmenyemail es rank queue mukodik
- [ ] Ismetelt cron-futas nem duplikal slotot, levelet vagy rankot
- [ ] Exam question statistics helyesen kezeli a pending valaszokat
- [ ] Trainee risk lista helyesen jeloli a resetelt es lejart trainee-ket
- [ ] Supervision fault pontozas, warning es automatikus fail mukodik
- [ ] Hianyzo lapdokumentacio eseten a lezaras tiltott
- [ ] Trainee supervision summary helyesen csoportositja a hibakat

## 6. Discord bot es automod

- [ ] `/profil` mukodik sajat es mas felhasznaloval
- [ ] Privat profil Discordbol sem olvashato
- [ ] `/poll` limitjei es opciovalidacioja mukodik
- [ ] Invite attribution: normal, vanity, consumed es ambiguous eset
- [ ] Ismert slur, profanity es custom word kezelese
- [ ] Allowlist megfeleloen felulirja a talalatot
- [ ] AI veto csak engedelyezett szabalyokra vonatkozik
- [ ] AI timeout eseten a deterministic automod tovabb mukodik
- [ ] Dry-run nem torol es nem buntet
- [ ] AI ignored, veto es valodi detection kulon logkent latszik
- [ ] Warning pontok es escalation szintek helyesek
- [ ] Warning torlese/allowlistelese visszaveszi a kapcsolodo pontokat
- [ ] Automod komponensek admin jogosultsag nelkul nem hasznalhatok

## 7. Email, ertesitesek es cronok

- [ ] `pnpm email:check`
- [ ] Email bekerul az outbox queue-ba
- [ ] Sikeres kuldes `SENT` allapotot kap
- [ ] Atmeneti hiba retry-t indit
- [ ] Vegleges cimhiba nem retry-zik vegtelenul
- [ ] Queue ujrainditas utan folytathato
- [ ] Duty reminder a kivalasztott idoben erkezik
- [ ] Dokumentum reminder nem kuld duplikalt levelet
- [ ] Daily duty report csak dolgozo tagoknak megy
- [ ] Weekly leadership digest helyes elozo heti osszehasonlitast mutat
- [ ] Cron endpointek `CRON_API_KEY` nelkul tiltottak
- [ ] Cronok tobbszori futasa idempotens

## 8. Statisztikak es megfigyelhetoseg

- [ ] Activity log keresese action, user, datum es `details` alapjan
- [ ] Ticket SLA statisztikak helyesek
- [ ] Support answer feedback es coverage riport mukodik
- [ ] Community statistics Discord/web osszekapcsolasa helyes
- [ ] Retention es forecast keves adattal sem torik el
- [ ] Status oldal dependency-hibanal is betolt
- [ ] Admin diagnostics mutat memoriat, queue-t es SSE-kapcsolatokat
- [ ] SSE kapcsolat bezarasakor nincs timer- vagy listener-leak

## 9. Reszponziv es regresszios ellenorzes

- [ ] Desktop nezet
- [ ] Mobil nezet 375 px szelessegen
- [ ] Admin tablazatok vizszintesen gorgethetok
- [ ] Modalok rovid kijelzon is bezarhatok es gorgethetok
- [ ] Tabok nem tornek tobb sorba
- [ ] Touch drag nem akadalyozza az oldal gorgeteset
- [ ] Bejelentkezes, logout, session refresh es middleware redirect mukodik
- [ ] `pnpm lint`
- [ ] `pnpm build`
- [ ] PR diff atnezese, kulonosen az uj es duplikalt fajloknal

## Teszteles kozben talalt hibak

| Prioritas | Funkcio | Leiras | Reprodukcio | Allapot |
|---|---|---|---|---|
| P0/P1/P2/P3 |  |  |  |  |
