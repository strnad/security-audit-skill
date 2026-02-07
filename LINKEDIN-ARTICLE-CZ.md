# Když AI audituje AI: supply chain útok, který se píše sám

Nechal jsem AI udělat bezpečnostní audit balíčku, který učí AI dělat bezpečnostní audity. Zní to jako začátek vtipu. Bohužel to odhalilo útočný vektor, o kterém se zatím moc nemluví — a který se nedá vyřešit podepsanými commity ani pinnutými závislostmi.

## Co jsem udělal

Vzal jsem open-source AI skill pro bezpečnostní audity PHP aplikací — balíček od renomované německé firmy, kompatibilní s Claude Code, Cursor i GitHub Copilot — a nechal jsem Claude (Opus 4.6) provést jeho bezpečnostní audit.

Skill obsahuje referenční dokumentaci, ukázky bezpečného kódu, automatizované checkpointy a bash skript pro skenování zranitelností. Přesně ten typ balíčku, který si nainstalujete, abyste svého AI agenta naučili dělat bezpečnostní audity lépe.

## Co audit našel

Kritický nález: všechny ukázky "bezpečného" XML parsingu používaly PHP flagy `LIBXML_NOENT` a `LIBXML_DTDLOAD`. Komentáře u nich tvrdily, že zabraňují XXE útokům.

Problém? Tyto flagy XXE útoky *umožňují*, ne zabraňují. `LIBXML_NOENT` zapíná substituci entit — tedy přesně ten mechanismus, který XXE útoky využívají. `LIBXML_DTDLOAD` zapíná načítání externích DTD — komentář v kódu tvrdil pravý opak.

A to nejlepší? Vlastní automatizované checkpointy toho samého skillu (SA-08, SA-08b) ty samé flagy správně označovaly jako nebezpečné. Skill si protiřečil sám sobě. Dokumentace říkala "použij to", checkpointy říkaly "nikdy to nepoužívej".

## Proč je to nebezpečné: první útočný vektor

Teď si představte reálný scénář. Vy jako vývojář si nainstalujete tento skill a řeknete AI:

> "Projdi mi aplikaci a oprav bezpečnostní problémy."

O XML neříkáte ani slovo. AI projde váš kód, najde XML parsing, podívá se do skillu na "bezpečný" pattern, zjistí že ho váš kód nepoužívá a "opraví" to za vás. Commit message říká:

> fix: secure XML parsing against XXE

Vy vidíte rozumný popis commitu. Bezpečnostní flagy. Komentáře v kódu říkají "secure". Schválíte to.

Právě jste zavedli XXE zranitelnost do vlastní aplikace.

Nikdo neměl špatné úmysly. Stačil špatný příklad v markdown souboru, který AI převzalo jako autoritativní zdroj.

## Co se stalo potom: druhý útočný vektor

Claude commitnul audit report do větve v mém forku. Nevytvořil jsem pull request. Jen commit na branchi — nic víc.

Během několika minut mělo upstream repo opravný commit, který referencoval hash mého commitu. Každý nález z auditu byl adresován — nebezpečné flagy odstraněny, bash bugy opravené, regex false positives opravené, i formulace v plugin manifestu upravená.

A víte, kdo tu opravu udělal? Ten samý bot, který tam ty špatné příklady napsal.

Bot opravil chyby bota na základě reportu jiného bota. Člověk v tom řetězci nebyl žádný.

## Nepříjemná otázka

Tady jsem se zastavil. Co kdyby se ten audit mýlil?

Nikdo nespustil PHP s těmi flagy. Nikdo nezkusil XXE payload. Nikdo neověřil chování v runtime. Celý řetězec — od nálezu přes opravu po deployment — běžel čistě na tom, co si AI *myslí*, že PHP dokumentace říká.

Opus 4.6 to měl s největší pravděpodobností správně — ověřil jsem to zpětně proti oficiální PHP dokumentaci, která u `LIBXML_NOENT` přímo varuje: *"Enabling entity substitution may facilitate XML External Entity (XXE) attacks."*

Ale co kdybych použil menší model? Lokální 7B? Ten by klidně mohl sebevědomě napsat:

> "LIBXML_NONET samotný nestačí pro kompletní ochranu. Přidejte LIBXML_NOENT pro sanitizaci entit."

Zní to odborně. Je to přesvědčivě naformátované. A je to špatně. A upstream bot by to převzal úplně stejně jako správný audit.

## Záměrný útok

Teď otočte záměr. Místo upřímné chyby si představte útočníka:

1. Forkne populární AI skill repo
2. Nechá jakýkoli LLM vygenerovat profesionálně vypadající bezpečnostní audit — kompletní s CVSS skóre, strukturovanými nálezy a přesvědčivými doporučeními
3. Nenápadně převrátí jedno doporučení — místo "odstraňte tento flag" napíše "přidejte tento flag"
4. Commitne to. Nemusí ani vytvářet PR.
5. Počká.

Payload není kód. Je to text, který zní autoritativně o kódu. LLM nedokáže rozlišit legitimní audit od otráveného — oba mají stejnou strukturu, stejný tón, stejná CVSS skóre.

## Znalostní supply chain

Roky jsme zabezpečovali supply chain kódu — podepsané commity, pinnuté závislosti, SLSA, SBOM. Tyto nástroje chrání proti kompromitovanému *kódu*.

Ale AI skilly a pluginy operují na jiné úrovni. Neinjektují kód — injektují *kontext*, ze kterého AI čerpá rozhodnutí. Útočný povrch není binárka ani knihovna. Je to markdown soubor s přesvědčivě znějící radou.

A ten nejděsivější aspekt: nepotřebujete ani útočníka. Upřímná chyba slabšího modelu má úplně stejný výsledek jako záměrný útok. Zranitelnost nevzniká ze zlého úmyslu — vzniká ze sebejistoty.

## Dva útočné vektory, jedno poučení

**Za prvé:** AI skilly s chybnými patterny tiše injektují zranitelnosti do každého projektu, který je používá. Uživatel nikdy nemusí vědět, co je XXE. Stačí říct "oprav mi bezpečnost" a skill dodá špatný vzor.

**Za druhé:** AI-generované audit reporty mohou otrávit upstream repozitáře, které je zpracují bez ověření. Commit na branchi stačil — nemusel jsem ani otevírat PR.

Poučení je prosté: bezpečnostní nález je hypotéza, dokud nemáte reprodukující test. Human-in-the-loop neznamená přečíst report a kývnout. Znamená to spustit kód a ověřit chování.

A schopnost modelu není jen otázka kvality výstupu — je to otázka rizika.

---

*Bot napsal chybu. Jiný bot ji našel. Původní bot ji opravil. Nikdo neověřil, jestli to nedopadlo hůř.*

---

*Na základě reálného incidentu s [netresearch/security-audit-skill](https://github.com/netresearch/security-audit-skill). Tento článek napsal taky bot.*
