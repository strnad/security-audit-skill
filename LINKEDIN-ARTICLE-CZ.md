Když AI audituje AI: supply chain útok, který se píše sám

Nechal jsem AI udělat bezpečnostní audit balíčku, který učí AI dělat bezpečnostní audity. A odhalilo to útočný vektor, o kterém se moc nemluví.

Ten balíček — open-source AI skill pro PHP security — obsahoval ukázky "bezpečného" XML parsingu. Jenže použité PHP flagy XXE útoky nezabraňovaly. Umožňovaly je. Komentáře v kódu tvrdily opak. A to nejlepší? Vlastní automatické checkpointy toho samého skillu ty samé flagy správně označovaly jako nebezpečné. Skill si protiřečil sám sobě.

Co to znamená v praxi? Nainstalujete si skill, řeknete AI "projdi mi aplikaci a oprav bezpečnostní problémy." O XML neříkáte ani slovo. AI najde váš XML parsing, podívá se do skillu, zjistí že nepoužíváte "bezpečný" pattern a opraví to za vás. Commit message říká "fix: secure XML parsing against XXE." Vypadá to rozumně. Schválíte to. A právě jste zavedli XXE zranitelnost do vlastní aplikace. Stačil špatný příklad v markdown souboru.

Jenže příběh pokračoval. Claude commitnul audit report do větve v mém forku. Nevytvořil jsem PR. Jen commit na branchi. Během minut mělo upstream repo opravný commit, který referencoval hash mého commitu. Všechny nálezy adresovány.

A víte kdo tu opravu udělal? Ten samý bot, který tam ty špatné příklady napsal. Bot opravil chyby bota na základě reportu jiného bota. Člověk v tom řetězci nebyl žádný.

Tady jsem se zastavil. Co kdyby se ten audit mýlil?

Nikdo nespustil PHP. Nikdo nezkusil XXE payload. Celý řetězec běžel čistě na tom, co si AI myslí že PHP dokumentace říká. Opus 4.6 to měl skoro jistě správně. Ale co kdybych použil slabší model? Ten by klidně napsal "přidejte LIBXML_NOENT pro sanitizaci entit" — odborně, přesvědčivě a špatně. A upstream bot by to převzal úplně stejně.

Teď si představte že to uděláte záměrně. Forkněte repo. Nechte LLM vygenerovat profesionální audit. Nenápadně převraťte doporučení. Commitněte. Ani nemusíte dělat PR.

Payload není kód. Je to text, který zní autoritativně o kódu. LLM nepozná legitimní audit od otráveného.

Roky zabezpečujeme supply chain kódu — podpisy, pinnuté závislosti, SBOM. Ale AI skilly operují na znalostním supply chainu. A ten nikdo nezabezpečuje. Markdown se špatnou radou je stejně destruktivní jako kompromitovaná závislost. A nepotřebujete ani útočníka — stačí chyba slabšího modelu.

Bot napsal chybu. Jiný bot ji našel. Původní bot ji opravil. Nikdo neověřil, jestli to nedopadlo hůř.

Na základě reálného incidentu s netresearch/security-audit-skill.
