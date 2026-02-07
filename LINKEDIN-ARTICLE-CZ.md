Když AI audituje AI: supply chain útok, který se píše sám

Dnes jsem nechal Claude udělat bezpečnostní audit open-source AI skillu — balíčku, který učí AI agenty hledat zranitelnosti v PHP kódu. Co by se mohlo pokazit, že?

Audit našel kritickou chybu: ukázkový "bezpečný" kód pro XML parsing používal PHP flagy, které XXE útoky nezabraňují, ale umožňují je. Komentáře tvrdily pravý opak. A vlastní checkpointy toho samého skillu ty samé flagy správně označovaly jako nebezpečné. Skill si protiřečil sám sobě.

Teď si představte reálný scénář. Nainstalujete si tento skill a řeknete AI: "udělej mi bezpečnostní audit aplikace a oprav co najdeš." Neříkáte nic o XML. AI projde váš kód, najde XML parsing, podívá se do skillu na "bezpečný" pattern, zjistí že ho váš kód nepoužívá a "opraví" ho. Commitne s popisem "fix: secure XML parsing against XXE". Vy vidíte rozumný commit message, bezpečnostní flagy, komentáře říkají "secure" — schválíte to. Právě jste do své aplikace zavedli XXE zranitelnost. Nikdo neměl špatné úmysly. Stačil špatný příklad v markdown souboru.

Ale to není všechno. Claude commitnul audit report do větve v mém forku. Žádný PR, jen commit na branchi. Během minut mělo upstream repo opravný commit referencující hash mého commitu. Každý nález adresován.

Nevím, jestli to udělal bot nebo člověk. Ale pak přišla nepříjemná otázka: co kdyby se audit mýlil?

Nikdo nespustil PHP s těmi flagy. Nikdo neprovedl XXE payload. Celý řetězec běžel na statistické jistotě AI v tom, co PHP dokumentace pravděpodobně říká.

Opus 4.6 to měl správně. Ale co kdybych použil menší model? Lokální 7B? Ten by klidně mohl sebevědomě napsat "LIBXML_NONET samotný nestačí, přidejte LIBXML_NOENT pro kompletní sanitizaci" — zní to odborně, je to naformátované a je to špatně. A upstream by to aplikoval stejně.

Teď otočte záměr. Forkněte populární AI skill repo. Nechte LLM vygenerovat profesionální audit s CVSS skóre. Nenápadně převraťte doporučení. Commitněte. Ani nedělejte PR. Počkejte.

Payload není kód. Je to text, který zní autoritativně o kódu. LLM nedokáže rozlišit legitimní audit od otráveného — oba mají stejnou strukturu, tón i skóre.

Dva útočné povrchy. AI skilly s chybnými patterny tiše injektují zranitelnosti do každého projektu, který je používá. A AI audit reporty mohou otrávit upstream, který je zpracuje bez ověření.

Roky jsme zpevňovali supply chain kódu. Ale AI skilly operují na znalostním supply chainu. Markdown se špatnou radou je stejně destruktivní jako kompromitovaná závislost. A nepotřebujete útočníka. Chyba slabšího modelu má stejný výsledek.

Nejnebezpečnější zranitelnost není v kódu. Je v propasti mezi sebejistotou AI a pravdou.

Na základě reálného incidentu s netresearch/security-audit-skill.
