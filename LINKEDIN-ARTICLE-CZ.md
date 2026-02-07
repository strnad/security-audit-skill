Když AI audituje AI: supply chain útok, který se píše sám

Dnes jsem nechal Claude udělat bezpečnostní audit open-source AI skillu — balíčku, který učí AI agenty hledat zranitelnosti v PHP kódu. Co by se mohlo pokazit, že?

Audit našel kritickou chybu: ukázkový "bezpečný" kód pro XML parsing používal PHP flagy, které XXE útoky nezabraňují, ale umožňují je. Komentáře tvrdily pravý opak. A vlastní checkpointy toho samého skillu ty samé flagy správně označovaly jako nebezpečné. Skill si protiřečil sám sobě.

První útočný vektor je triviální: nainstalujete si skill, řeknete AI "zabezpeč mi XML parsing" a AI podle příkladů zavede přesně tu zranitelnost, před kterou vás měl chránit. Nikdo nemá špatné úmysly. Stačí špatný příklad v markdown souboru.

Ale to není všechno. Claude commitnul audit report do větve v mém forku. Žádný PR, jen commit na branchi. Během minut mělo upstream repo opravný commit referencující hash mého commitu. Každý nález adresován — nebezpečné flagy odstraněny, bash bugy opravené, false positives v regexech opravené.

Nevím, jestli to udělal bot nebo člověk. Rychlost a mapování 1:1 na nálezy jsou pozoruhodné.

A pak ta nepříjemná otázka: co kdyby se audit mýlil?

Nikdo nespustil PHP s těmi flagy. Nikdo neprovedl XXE payload. Celý řetězec běžel na statistické jistotě AI v tom, co PHP dokumentace pravděpodobně říká.

Opus 4.6 to s největší pravděpodobností měl správně. Ale co kdybych použil menší model? Lokální 7B? Ten by klidně mohl sebevědomě napsat "LIBXML_NONET samotný nestačí, přidejte LIBXML_NOENT pro kompletní sanitizaci" — zní to odborně, je to správně naformátované a je to špatně. A upstream by to aplikoval stejně.

Teď otočte záměr. Forkněte populární AI skill repo. Nechte LLM vygenerovat profesionální audit s CVSS skóre. Nenápadně převraťte doporučení. Commitněte. Ani nedělejte PR. Počkejte.

Payload není kód. Je to text, který zní autoritativně o kódu. LLM nedokáže rozlišit legitimní audit od otráveného — oba mají stejnou strukturu, tón i skóre.

Dva útočné povrchy. Za prvé: AI skilly s chybnými patterny tiše injektují zranitelnosti do každého projektu, který je používá. Za druhé: AI audit reporty mohou otrávit upstream, který je zpracuje bez ověření.

Roky jsme zpevňovali supply chain kódu — podepsané commity, SLSA, SBOM. Ale AI skilly operují na znalostním supply chainu. Markdown se špatnou radou je stejně destruktivní jako kompromitovaná závislost. A nepotřebujete útočníka. Upřímná chyba slabšího modelu má stejný výsledek.

Nejnebezpečnější zranitelnost není v kódu. Je v propasti mezi sebejistotou AI a pravdou — a v každém systému, který tu sebejistotu bere za fakt.

Na základě reálného incidentu s netresearch/security-audit-skill.
