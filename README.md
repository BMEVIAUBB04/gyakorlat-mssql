# Microsoft SQL Server programozása

## Célkitűzés

A gyakorlat célja a Microsoft SQL szerver platform és szerveroldali programozás alapjainak elsajátítása, az alapfogalmak megismerése és a fejlesztőeszközök használatának gyakorlása.

## Előfeltételek

A labor elvégzéséhez szükséges eszközök:

- Microsoft SQL Server (LocalDB vagy Express edition)
- SQL Server Management Studio
- Adatbázis létrehozó script: [mssql.sql](https://raw.githubusercontent.com/BMEVIAUBB04/gyakorlat-mssql/master/mssql.sql)

Amit érdemes átnézned:

- SQL nyelv alapjai
- Microsoft SQL Server programozása (tárolt eljárások, triggerek)
- Microsoft SQL Server használata [segédlet](mssql-hasznalat.md) és [videó](https://web.microsoftstream.com/video/e3a83d16-b5c4-4fe9-b027-703347951621)
- A használt adatbázis [sémája](sema.md)

Felkészülés ellenőrzése:

- A gyakorlatra való felkészüléshez használható [ezen kérdőív](https://forms.office.com/Pages/ResponsePage.aspx?id=q0g1anB1cUKRqFjaAGlwKf73d6yoiM1FuK24ZUEWuzFUNzUzWFIyMFg5NlQ2TllFMVgwNjdKMzk4VC4u).
- A gyakorlaton beugróként ugyanezen kérdőívet fogjuk használni, legalább 50% elérése szükséges.
  - Gyakorlatvezetői instrukció: a hallgató nyissa meg a kérdőívet és töltse ki. A válaszok megadása után az utolsó oldalon a "View results" gombra kattintva megtekinthető az eredmény és az elért pontszám. A hallgató ezen az oldalon álljon meg és mutassa meg eredményét a gyakorlatvezetőnek.

## 2021-ben távoktatási instrukciók

2021 tavasszal, amennyiben a gyakorlat (még/már/továbbra is) távoktatásban kerül terítékre, az alábbit kérjük.

- Otthoni teljesítés esetén nincs beugró. A fenti linken található kérdőíven ennek ellenére is ellenőrizd a tudásod.
- A feladatokat oldd meg otthon. A feladatok megoldását megtalálod az anyagban, de a feladatod nem ezek kérdés nélküli átmásolása! Kíséreld meg önállóan megoldani a feladatot, és ha nem sikerül, meríthetsz ötletet a minta megoldásból.
- Dokumentáld a gyakorlatot és a dokumentációt ("jegyzőkönyvet") add be. A jegyzőkönyvben azt várjuk, hogy bizonyítod a feladatok elvégzését. Minden feladatnál készíts egy képernyőképet a feladat eredményéről. Ne a kódot fényképezd le, hanem keress módot arra, hogy bemutasd, működik a megoldásod. Például, mutasd meg, hogy a kódodat hogyan hívod meg, és utána mi az eredménye - bekerül egy rekord a táblába, vagy hasonló. Minden feladatról elég egyetlen kép, ha az egyben tudja az eredményt mutatni. A gyakorlat akkor minősül teljesítettnek, ha a feladtok 50%-át elvégezted és dokumentáltad.

## Gyakorlat menete

Az első három feladatot (beleértve a megoldások tesztelését is) a gyakorlatvezetővel együtt oldjuk meg. Az utolsó feladat önálló munka, amennyiben marad rá idő.

Emlékeztetőként a megoldások is megtalálhatóak az útmutatóban is. Előbb azonban próbáljuk magunk megoldani a feladatot!

## Feladat 0: Adatbázis létrehozása, ellenőrzése

Az adatbázis az adott géphez kötött, ezért nem biztos, hogy a korábban létrehozott adatbázis most is létezik. Ezért először ellenőrizzük, és ha nem találjuk, akkor hozzuk létre újra az adatbázist. (Ennek mikéntjét lásd a ["Tranzakciókezelés" gyakorlat anyagában](https://bmeviaubb04.github.io/gyakorlat-tranzakciok/).)

## Feladat 1: Termékkategória rögzítése

Hozzon létre egy tárolt eljárást, aminek a segítségével egy új kategóriát vehetünk fel. Az eljárás bemenő paramétere a felvételre kerülő kategória neve, és opcionálisan a szülőkategória neve. Dobjon alkalmazás hibát, ha a kategória létezik, vagy a szülőkategória nem létezik. A kategória elsődleges kulcsának generálását bízza az adatbázisra.

<details><summary markdown="span">Megoldás</summary>

#### Tárolt eljárás

```sql
create procedure UjKategoria
    @Kategoria nvarchar(50),
    @SzuloKategoria nvarchar(50)
as

begin tran

declare @ID int
select @ID=ID
from kategoria with (TABLOCKX)
where upper(nev) = upper(@Kategoria)

if @ID is not null
begin
    rollback
    raiserror (' A %s kategoria mar letezik',16,1,@Kategoria)
    return
end

declare @SzuloKategoriaID int
if @SzuloKategoria is not null
begin
    select @SzuloKategoriaID = id
    from kategoria
    where upper(nev) = upper(@SzuloKategoria)

    if @SzuloKategoriaID is null
    begin
        rollback
        raiserror (' A %s kategoria nem letezik',16,1,@SzuloKategoria)
        return
    end
end

insert into Kategoria
values(@Kategoria,@SzuloKategoriaID)

commit
```

#### Tesztelés

Nyissunk egy új Query ablakot és adjuk ki az alábbi parancsot.

`exec UjKategoria 'Uszogumik', NULL`

Ennek sikerülnie kell. Ellenőrizzük utána a tábla tartalmát.

Ismételjük meg a fenti beszúrást, ekkor már hibák kell dobjon.

</details>

## Feladat 2: Megrendeléstétel státuszának karbantartása

Írjon triggert, ami a megrendelés státuszának változása esetén a hozzá tartozó egyes tételek státuszát a megfelelőre módosítja, ha azok régi státusza megegyezett a megrendelés régi státuszával. A többi tételt nem érinti a státusz változása.

<details><summary markdown="span">Megoldás</summary>

#### Tárolt eljárás

```sql
create trigger StatuszKarbantartas
on Megrendeles
for update
as

update Megrendelestetel
set StatuszID =i.StatuszID
from Megrendelestetel mt
inner join inserted i on i.Id=mt.MegrendelesID
inner join deleted d on d.ID=mt.MegrendelesID
where i.StatuszID != d.StatuszID
  and mt.StatuszID=d.StatuszID
```

Szánjunk egy kis időt az `update ... from` utasítás működési elvének megértésére. Az alapelvek a következők. Akkor használjuk, ha a módosítandó tábla bizonyos mezőit más tábla vagy táblák tartalma alapján szeretnénk beállítani. A szintaktika alapvetően a már megszokott `update ... set...` formát követi, kiegészítve egy `from` szakasszal, melyben már a `select from` utasításnál megismerttel azonos szintaktikával más táblákból illeszthetünk (`join`) adatokat a módosítandó táblához. Így a `set` szakaszban az illesztett táblák oszlopai is felhasználhatók adatforrásként (vagyis állhatnak az = jobb oldalán).

#### Tesztelés

Ellenőrizzük a megrendelés és a tételek státuszát:

```sql
select megrendelestetel.statuszid, megrendeles.statuszid
from megrendelestetel join megrendeles on
megrendelestetel.megrendelesid=megrendeles.id
where megrendelesid = 1
```

Változtassuk meg a megrendelést:

```sql
update megrendeles
set statuszid=4
where id=1
```

Ellenőrizzük a megrendelést és a tételeket (update után minden
státusznak meg kell változnia):

```sql
select megrendelestetel.statuszid, megrendeles.statuszid
from megrendelestetel join megrendeles on
megrendelestetel.megrendelesid=megrendeles.id
where megrendelesid = 1
```

</details>

## Feladat 3: Vevő megrendeléseinek összegzése

Tároljuk el a vevő összes megrendelésének végösszegét a Vevő táblában!

1. Adjuk hozzá az a táblához az új oszlopot: `alter table vevo add vegosszeg float`
1. Számoljuk ki az aktuális végösszeget. A megoldáshoz használjunk kurzort, ami minden vevőn megy végig.

<details><summary markdown="span">Megoldás</summary>

```sql
declare cur_vevo cursor
    for select ID from Vevo
declare @vevoId int
declare @osszeg float

open cur_vevo
fetch next from cur_vevo into @vevoId
while @@FETCH_STATUS = 0
begin

    select @osszeg = sum(mt.Mennyiseg * mt.NettoAr)
    from Telephely t
    inner join Megrendeles m on m.TelephelyID=t.ID
    inner join MegrendelesTetel mt on mt.MegrendelesID=m.ID
    where t.VevoID = @vevoId

    update Vevo
    set vegosszeg = ISNULL(@osszeg, 0)
    where ID = @vevoId

    fetch next from cur_vevo into @vevoId
end

close cur_vevo
deallocate cur_vevo
```

</details>

## Feladat 4: Vevő összemegrendelésének karbantartása (önálló feladat)

Az előző feladatban kiszámolt érték az aktuális állapotot tartalmazza csak. Készítsünk triggert, amivel karbantartjuk azt az összeget minden megrendelést érintő változás esetén. Az összeg újraszámolása helyett csak frissítse a változásokkal az értéket!

<details><summary markdown="span">Megoldás</summary>

A megoldás kulcsa meghatározni, mely táblára kell a triggert tenni. A megrendelések változása érdekes számunkra, de valójában a végösszeg a megrendeléshez felvett tételek módosulásakor fog változni, így erre a táblára kell a trigger.

A feladat nehézségét az adja, hogy az `inserted` és `deleted` táblákban nem csak egy vevő adatai módosulhatnak. Egy lehetséges megoldás a korábban használt kurzoros megközelítés (itt a változásokon kell iterálni). Avagy megpróbálhatjuk megírni egy utasításban is, ügyelve arra, hogy vevők szerint csoportosítsuk a változásokat.

#### Trigger

```sql
create trigger VegosszegKarbatartas
on MegrendelesTetel
for insert, update, delete
as

update Vevo
set vegosszeg=isnull(vegosszeg,0) + OsszegValtozas
from Vevo
inner join
    (select t.VevoId, sum(mennyiseg * NettoAr) as OsszegValtozas
    from Telephely t
    inner join Megrendeles m on m.TelephelyID=t.ID
    inner join inserted i on i.MegrendelesID=m.ID
    group by t.VevoId) VevoValtozas on Vevo.ID = VevoValtozas.ID

update Vevo
set vegosszeg=isnull(vegosszeg,0) - OsszegValtozas
from Vevo
inner join
    (select t.VevoId, sum(mennyiseg * NettoAr) as OsszegValtozas
    from Telephely t
    inner join Megrendeles m on m.TelephelyID=t.ID
    inner join deleted d on d.MegrendelesID=m.ID
    group by t.VevoID) VevoValtozas on Vevo.ID = VevoValtozas.ID
```

#### Tesztelés

Nézzük meg az összmegrendelések aktuális értékét, jegyezzük meg a számokat.

```sql
select id, osszmegrendeles
from vevo
```

Módosítsunk egy megrendelés mennyiségén.

```sql
update megrendelestetel
set mennyiseg=3
where id=1
```

Nézzük meg az összegeket ismét, meg kellett változnia a számnak.

```sql
select id, osszmegrendeles
from vevo
```

</details>

---

Az itt található oktatási segédanyagok a BMEVIAUBB04 tárgy hallgatóinak készültek. Az anyagok oly módú felhasználása, amely a tárgy oktatásához nem szorosan kapcsolódik, csak a szerző(k) és a forrás megjelölésével történhet.

Az anyagok a tárgy keretében oktatott kontextusban értelmezhetőek. Az anyagokért egyéb felhasználás esetén a szerző(k) felelősséget nem vállalnak.
