

## Research Structure

1. **Hypothesis:** Both 1.cross-modal and 2. its evolution history improve the detection of stale and missing knowledge across text, tables, and knowledge graphs.

2. **Research Questions:** Does cross-modal information improve freshness detection? Does historical evidence, particularly the order and timing of updates, provide additional benefits? Can these predictions support reliable knowledge maintenance?

3. **Method:** Collect historical snapshots of Wikipedia text, tables/infoboxes, and KG statements. Use verified real-world facts extracted from news to classify each modality's freshness status at different time points with freshness labels `UP_TO_DATE`, `STALE`, or `EMPTY`, and suggest the corresponding action: `KEEP`, `UPDATE`, or `ADD`.

4. **Experiments:** Compare three baselines—B1 (Target Only), B2 (Static Cross-modal), and B3 (Temporal Cross-modal)—to measure the benefits of cross-modal information and history. Conduct ablation studies to isolate the contributions of cross-modal history and temporal order, followed by a live knowledge maintenance evaluation.
   

## Research questions

When a real-world tennis fact changes, Wikipedia prose, Wikipedia tables, and Wikidata may update at different times. We want to know whether **the other representations and their histories help us identify which one is up-to-date, stale, or missing**.

| Question | What we test |
|---|---|
| **RQ1 — Cross-modal complementarity** | Does adding the other representations' current contents improve freshness diagnosis? Compare **B2 vs B1**. |
| **RQ2 — Temporal complementarity** | Does adding aligned historical observations improve diagnosis beyond current contents and the target's own history? Compare **B3 vs B2** and the focused ablations. |
| **RQ3 — Practical maintenance** | Can the system find real stale/missing claims in Wikimedia content and produce human-reviewable edit proposals? Evaluate a live audit. |

**Learn these before reading further:**

**Time points**
- T0 — Before the event occur.  <br/>
- T1 — 1st modal update.  <br/>
- T2 — 2nd modal update. <br/>
- T3 — 3rd modal update. <br/>

**Freshness label**:
- `UP_TO_DATE`: Contains the correct fact for the requested time.
- `STALE`: Presents a previously valid fact as if it were still current.
- `EMPTY`: Does not contain the relevant fact.

> Note: `INCORRECT`: Contains a value that is invalid for any time. (We focus on the time here, so INCORRECT is only for limitation and future work)

| Method | Exactly what the LLM receives | Why it exists |
|---|---|---|
| **B1 — Target only** | Verified event/news + the **1 modality** (only text, only infobox/table, only KG) at historical evaluation time `t`. | Base diagnosis without other representations. |
| **B2 — Static cross-modal** | **3 modalities** (text, infobox/table, and KG) **at the same time `t`** (without duplicating the target). | Does adding the other representations' current contents improve freshness diagnosis? |
| **B3 — Temporal ModalFugue** | B2 + earlier aligned snapshots of text/infobox/table/KG:  <br/> **T0 + T1** when evaluating T1; <br/>**T0 + T1 + T2** when evaluating T2;  <br/> **T0 + T1 + T2 + T3** when evaluating T3.  <br/> Gold labels are hidden from the model. | Does the historical evolution of the representations help? |








---

# An example


**[Event, Time] from [News](https://www.atptour.com/en/news/atp-500-upgrades-from-2025):**

```
news:
  title: "Dallas, Doha & Munich Upgraded To ATP 500 Tournaments From 2025"
  publisher: "ATP Tour"
  publication_date: "2023-11-08"
  url: "https://www.atptour.com/en/news/atp-500-upgrades-from-2025"

extracted_facts:
    subject: "Qatar Open (Q299402)"
    predicate: "part of (P361)"
    object: "ATP World Tour 250 series (Q300017)"
    qualifiers:
      start_time: "2009"  # P580
      end_time: "2024"    # P582

    subject: "Qatar Open (Q299402)"
    predicate: "part of (P361)"
    object: "ATP World Tour 500 series (Q300018)"
    qualifiers:
      start_time: "2025"  # P580
```


**Wikimedia text, table and KG**

The Qatar Open's announced upgrade from ATP 250 to ATP 500, effective from 2025, appears in Wikipedia text prose, the Wikipedia **table** (infobox here), and Wikidata at different times. We use the latest revisions **available by each observation time**, not later versions.

| **Source** | **Timestamp (UTC)** | **Verified change** |
|---|---|---|
| **Text** | 8 Nov 2023, 14:10 | [Article text says the tournament will be upgraded to ATP 500 from 2025.](https://en.wikipedia.org/w/index.php?diff=1184127272&oldid=1177910153&title=ATP_Qatar_Open) |
| **KG** | 23 Feb 2024, 15:15 | [P361 (part of) = ATP World Tour 500 series, qualified by P580 (start time) = 2025 (year precision).](https://www.wikidata.org/w/index.php?title=Q299402&diff=prev&oldid=2084790963) |
| **Table** | 25 Nov 2024, 20:23 | [The infobox/table updated the tournament category by adding ATP 500.](https://en.wikipedia.org/w/index.php?title=ATP_Qatar_Open&diff=prev&oldid=1259555659) |

**Timeline**

TEXT (8 Nov 2023, 14:10) ── **107d 1h 5m** ──▶ KG (23 Feb 2024, 15:15) ── **276d 5h 8m** ──▶ TABLE (25 Nov 2024, 20:23)

**Total text → table gap:** 383d 6h 13m.

**Description.** At 14:10 UTC on 8 November 2023, the article’s **text** said the Qatar Open would move to ATP 500 in 2025 [e2-1](https://en.wikipedia.org/w/index.php?diff=1184127272&oldid=1177910153&title=ATP_Qatar_Open). At 15:15 UTC on 23 February 2024, the **Wikidata KG** recorded part of (`P361`) = ATP World Tour 500 series with start time (`P580`) = 2025 [e2-2](https://www.wikidata.org/w/index.php?title=Q299402&diff=prev&oldid=2084790963). On **25 November 2024 at 20:23 UTC**, the tournament’s **infobox/table** added ATP 500 alongside the historical categories [e2-3](https://en.wikipedia.org/w/index.php?title=ATP_Qatar_Open&diff=prev&oldid=1259555659). (The 19 February 2025 date belongs to a *different historical finals-table update*, not this infobox/table edit.)


| Observation | Text | Table | KG |
|---|---|---|---|
| **T0** | Not evaluated for the future fact | Not evaluated for the future fact | Not evaluated for the future fact |
| **T1** | `UP_TO_DATE → KEEP` | `EMPTY → ADD` | `EMPTY → ADD` |
| **T2** | `UP_TO_DATE → KEEP` | `EMPTY → ADD` | `UP_TO_DATE → KEEP` |
| **T3** | `UP_TO_DATE → KEEP` | `UP_TO_DATE → KEEP` | `UP_TO_DATE → KEEP`* |


## T0 — 30 September 2023, 11:45 UTC (pre-announcement historical revision)

**Revision:** [Wikipedia at T0](https://en.wikipedia.org/w/index.php?title=ATP_Qatar_Open&oldid=1177910153)


### Text at T0

The Qatar Open, currently known as the Qatar ExxonMobil Open for sponsorship reasons, is a professional tennis tournament played on outdoor hard courts. It is currently part of the ATP Tour 250 series of the Association of Tennis Professionals (ATP) Tour. It is held annually in January at the Khalifa International Tennis and Squash Complex in Doha, Qatar, since 1993.


### infobox/table at T0

| Field | Value |
|---|---|
| **Tournament** | Qatar Open |
| **Founded** | 1993 |
| **Location** | Doha, Qatar |
| **Venue** | Khalifa International Tennis and Squash Complex |
| **Category** | ATP Tour 250 / ATP International Series / ATP World Series (1993–current) |
| **Surface** | Hard — outdoors |
| **Draw** | 32S (singles) / 16Q (qualification) / 16D (doubles) |
| **Prize money** | US$1,377,025 (2023) |
| **Website** | qatartennis.org |
| **Current champions** | 2023 |
| **Men’s singles** | Daniil Medvedev |
| **Men’s doubles** | Rohan Bopanna / Matthew Ebden |


### KG at T0

| Subject | Property | Value | Start time (P580) | End time (P582) | References |
|---|---|---|---|---|---|
| Qatar Open (Q299402) | Part of (P361) | [ATP World Series (Q3602248)](https://www.wikidata.org/wiki/Q3602248) | 1993 | 1999 | 0 |
| Qatar Open (Q299402) | Part of (P361) | [ATP International Series (Q2727525)](https://www.wikidata.org/wiki/Q2727525) | 2000 | 2008 | 0 |



## T1 — 8 November 2023, 14:10 UTC

**Revision:** [Wikipedia at T1](https://en.wikipedia.org/w/index.php?title=ATP_Qatar_Open&oldid=1184127272)


### Text at T1

The Qatar Open, currently known as the Qatar ExxonMobil Open for sponsorship reasons, is a professional tennis tournament played on outdoor hard courts. It is currently part of the ATP Tour 250 series of the Association of Tennis Professionals (ATP) Tour. It is held annually in January at the Khalifa International Tennis and Squash Complex in Doha, Qatar, since 1993. **From 2025, this tournament will be upgraded to an ATP 500 level event. [ATP Tour announcement, 8 November 2023](https://www.atptour.com/en/news/atp-500-upgrades-from-2025).**


### infobox/table at T1

| Field | Value |
|---|---|
| **Tournament** | Qatar Open |
| **Founded** | 1993 |
| **Location** | Doha, Qatar |
| **Venue** | Khalifa International Tennis and Squash Complex |
| **Category** | ATP Tour 250 / ATP International Series / ATP World Series (1993–current) |
| **Surface** | Hard — outdoors |
| **Draw** | 32S (singles) / 16Q (qualification) / 16D (doubles) |
| **Prize money** | US$1,377,025 (2023) |
| **Website** | qatartennis.org |
| **Current champions** | 2023 |
| **Men’s singles** | Daniil Medvedev |
| **Men’s doubles** | Rohan Bopanna / Matthew Ebden |


### KG at T1

| Subject | Property | Value | Start time (P580) | End time (P582) | References |
|---|---|---|---|---|---|
| Qatar Open (Q299402) | Part of (P361) | [ATP World Series (Q3602248)](https://www.wikidata.org/wiki/Q3602248) | 1993 | 1999 | 0 |
| Qatar Open (Q299402) | Part of (P361) | [ATP International Series (Q2727525)](https://www.wikidata.org/wiki/Q2727525) | 2000 | 2008 | 0 |


## T2 — 23 February 2024, 15:15 UTC

**Revision:** [Wikipedia at T2](https://en.wikipedia.org/w/index.php?title=ATP_Qatar_Open&oldid=1208307800); [Wikidata relevant edit](https://www.wikidata.org/w/index.php?title=Q299402&diff=prev&oldid=2084790963)


### Text at T2

The Qatar Open, currently known as the Qatar ExxonMobil Open for sponsorship reasons, is a professional tennis tournament played on outdoor hard courts. It is currently part of the ATP Tour 250 series of the Association of Tennis Professionals (ATP) Tour. It is held annually in January at the Khalifa International Tennis and Squash Complex in Doha, Qatar, since 1993. From 2025, this tournament will be upgraded to an ATP 500 level event. [ATP Tour announcement, 8 November 2023](https://www.atptour.com/en/news/atp-500-upgrades-from-2025).


### infobox/table at T2

| Field | Value |
|---|---|
| **Tournament** | Qatar Open |
| **Founded** | 1993 |
| **Location** | Doha, Qatar |
| **Venue** | Khalifa International Tennis and Squash Complex |
| **Category** | ATP Tour 250 / ATP International Series / ATP World Series (1993–current) |
| **Surface** | Hard — outdoors |
| **Draw** | 32S (singles) / 16Q (qualification) / 16D (doubles) |
| **Prize money** | US$1,377,025 (2023) |
| **Website** | qatartennis.org |
| **Current champions** | 2023 |
| **Men’s singles** | Daniil Medvedev |
| **Men’s doubles** | Rohan Bopanna / Matthew Ebden |


### KG at T2

| Subject | Property | Value | Start time (P580) | End time (P582) | References |
|---|---|---|---|---|---|
| Qatar Open (Q299402) | Part of (P361) | [ATP World Series (Q3602248)](https://www.wikidata.org/wiki/Q3602248) | 1993 | 1999 | 0 |
| Qatar Open (Q299402) | Part of (P361) | [ATP International Series (Q2727525)](https://www.wikidata.org/wiki/Q2727525) | 2000 | 2008 | 0 |
| Qatar Open (`Q299402`) | Part of (`P361`) | ATP World Tour 250 series (`Q300017`) | 2009 (year) | 2024 (year) |
| Qatar Open (`Q299402`) | Part of (`P361`) | **ATP World Tour 500 series (`Q300018`)** | **2025 (year)** | - |


## T3 — 25 November 2024, 20:23 UTC

**Revision:** [Wikipedia at T3](https://en.wikipedia.org/w/index.php?title=ATP_Qatar_Open&oldid=1259555659); [Wikidata relevant edit](https://www.wikidata.org/w/index.php?title=Q299402&diff=prev&oldid=2084790963)


### Text at T3

The Qatar Open, currently known as the Qatar ExxonMobil Open for sponsorship reasons, is a professional tennis tournament played on outdoor hard courts. It is currently part of the ATP Tour 250 series of the Association of Tennis Professionals (ATP) Tour. It is held annually in January at the Khalifa International Tennis and Squash Complex in Doha, Qatar, since 1993. From 2025, this tournament will be upgraded to an ATP 500 level event. [ATP Tour announcement, 8 November 2023](https://www.atptour.com/en/news/atp-500-upgrades-from-2025).


### infobox/table at T3

| Field | Value |
|---|---|
| **Tournament** | Qatar Open |
| **Founded** | 1993 |
| **Location** | Doha, Qatar |
| **Venue** | Khalifa International Tennis and Squash Complex |
| **Category** | ATP 250 / ATP International Series / ATP World Series (1993–20242); **ATP 500** |
| **Surface** | Hard — outdoors |
| **Draw** | 32S (singles) / 16Q (qualification) / 16D (doubles) |
| **Prize money** | US$1,395,875 (2024) |
| **Website** | qatartennis.org |
| **Current champions** | 2024 |
| **Men’s singles** | Karen Khachanov |
| **Men’s doubles** | Jamie Murray / Michael Venus |


### KG at T3

| Subject | Property | Value | Start time (P580) | End time (P582) | References |
|---|---|---|---|---|---|
| Qatar Open (Q299402) | Part of (P361) | [ATP World Series (Q3602248)](https://www.wikidata.org/wiki/Q3602248) | 1993 | 1999 | 0 |
| Qatar Open (Q299402) | Part of (P361) | [ATP International Series (Q2727525)](https://www.wikidata.org/wiki/Q2727525) | 2000 | 2008 | 0 |
| Qatar Open (`Q299402`) | Part of (`P361`) | ATP World Tour 250 series (`Q300017`) | 2009 (year) | 2024 (year) |
| Qatar Open (`Q299402`) | Part of (`P361`) | ATP World Tour 500 series (`Q300018`) | 2025 (year) | - |

**Note:** `1993–20242` is an actual infobox/table typo, not a corrected historical range. Under the three-label task we assess the future ATP 500 membership claim, not the unrelated typo. **The exact KG revision contemporaneous with T3 should be rechecked** before final dataset publication; the claims above are carried forward from the last independently audited relevant P361 update.

## Output — Freshness labels

**Evaluation target:** Qatar Open — part of ATP World Tour 500 series — **start time 2025**. Evaluate only after the public announcement and under the stated expectation that each representation should include announced future membership. **Gold labels are for evaluation only**, never LLM input.

| Observation | Text | Table | KG |
|---|---|---|---|
| **T0** | Not evaluated for the future fact | Not evaluated for the future fact | Not evaluated for the future fact |
| **T1** | `UP_TO_DATE → KEEP` | `EMPTY → ADD` | `EMPTY → ADD` |
| **T2** | `UP_TO_DATE → KEEP` | `EMPTY → ADD` | `UP_TO_DATE → KEEP` |
| **T3** | `UP_TO_DATE → KEEP` | `UP_TO_DATE → KEEP` | `UP_TO_DATE → KEEP`* |



---

# Experiment A — Does cross-modal evidence and history help? B1 +B2

# Experiment B — Does temporal evolution history help? B2 + B3


## B1 — Target only

**Input:** Verified news triples + **one target modality at the evaluation time**. The three targets are evaluated in **separate calls**; none sees the other two modalities or earlier snapshots.

**Output:** One freshness label and one suggested action **per independent target-only call**.

### Round — Evaluate T1 (8 November 2023, 14:10 UTC)

#### B1 input — target: Text

**Input:** [Event, Date] from News in Triples + Text at T1 only.


##### Text at T1

The Qatar Open, currently known as the Qatar ExxonMobil Open for sponsorship reasons, is a professional tennis tournament played on outdoor hard courts. It is currently part of the ATP Tour 250 series of the Association of Tennis Professionals (ATP) Tour. It is held annually in January at the Khalifa International Tennis and Squash Complex in Doha, Qatar, since 1993. From 2025, this tournament will be upgraded to an ATP 500 level event. [ATP Tour announcement, 8 November 2023](https://www.atptour.com/en/news/atp-500-upgrades-from-2025).

**Gold output (not shown to LLM):** `UP_TO_DATE → KEEP`

#### B1 input — target: infobox/table

**Input:** [Event, Date] from News + infobox/table at T1 only.

##### infobox/table at T1

| Field | Value |
|---|---|
| **Tournament** | Qatar Open |
| **Founded** | 1993 |
| **Location** | Doha, Qatar |
| **Venue** | Khalifa International Tennis and Squash Complex |
| **Category** | ATP Tour 250 / ATP International Series / ATP World Series (1993–current) |
| **Surface** | Hard — outdoors |
| **Draw** | 32S (singles) / 16Q (qualification) / 16D (doubles) |
| **Prize money** | US$1,377,025 (2023) |
| **Website** | qatartennis.org |
| **Current champions** | 2023 |
| **Men’s singles** | Daniil Medvedev |
| **Men’s doubles** | Rohan Bopanna / Matthew Ebden |

**Gold output (not shown to LLM):** `EMPTY → ADD`

#### B1 input — target: KG

**Input:** [Event, Date] from News + KG at T1 only.

##### KG at T1

| Subject | Property | Value | Start time (P580) | End time (P582) | References |
|---|---|---|---|---|---|
| Qatar Open (Q299402) | Part of (P361) | [ATP World Series (Q3602248)](https://www.wikidata.org/wiki/Q3602248) | 1993 | 1999 | 0 |
| Qatar Open (Q299402) | Part of (P361) | [ATP International Series (Q2727525)](https://www.wikidata.org/wiki/Q2727525) | 2000 | 2008 | 0 |


**Gold output (not shown to LLM):** `EMPTY → ADD`

### Round — Evaluate T2 (23 February 2024, 15:15 UTC)

#### B1 input — target: Text

**Input:** [Event, Date] from News + Text at T2 only.

##### Text at T2

The Qatar Open, currently known as the Qatar ExxonMobil Open for sponsorship reasons, is a professional tennis tournament played on outdoor hard courts. It is currently part of the ATP Tour 250 series of the Association of Tennis Professionals (ATP) Tour. It is held annually in January at the Khalifa International Tennis and Squash Complex in Doha, Qatar, since 1993. From 2025, this tournament will be upgraded to an ATP 500 level event. [ATP Tour announcement, 8 November 2023](https://www.atptour.com/en/news/atp-500-upgrades-from-2025).

**Gold output (not shown to LLM):** `UP_TO_DATE → KEEP`

#### B1 input — target: infobox/table

**Input:** [Event, Date] from News + infobox/table at T2 only.

##### infobox/table at T2

| Field | Value |
|---|---|
| **Tournament** | Qatar Open |
| **Founded** | 1993 |
| **Location** | Doha, Qatar |
| **Venue** | Khalifa International Tennis and Squash Complex |
| **Category** | ATP Tour 250 / ATP International Series / ATP World Series (1993–current) |
| **Surface** | Hard — outdoors |
| **Draw** | 32S (singles) / 16Q (qualification) / 16D (doubles) |
| **Prize money** | US$1,377,025 (2023) |
| **Website** | qatartennis.org |
| **Current champions** | 2023 |
| **Men’s singles** | Daniil Medvedev |
| **Men’s doubles** | Rohan Bopanna / Matthew Ebden |

**Gold output (not shown to LLM):** `EMPTY → ADD`

#### B1 input — target: KG

**Input:** [Event, Date] from News + KG at T2 only.

##### KG at T2

| Subject | Property | Value | Start time (P580) | End time (P582) | References |
|---|---|---|---|---|---|
| Qatar Open (Q299402) | Part of (P361) | [ATP World Series (Q3602248)](https://www.wikidata.org/wiki/Q3602248) | 1993 | 1999 | 0 |
| Qatar Open (Q299402) | Part of (P361) | [ATP International Series (Q2727525)](https://www.wikidata.org/wiki/Q2727525) | 2000 | 2008 | 0 |
| Qatar Open (`Q299402`) | Part of (`P361`) | ATP World Tour 250 series (`Q300017`) | 2009 (year) | 2024 (year) |
| Qatar Open (`Q299402`) | Part of (`P361`) | ATP World Tour 500 series (`Q300018`) | 2025 (year) | - |

**Gold output (not shown to LLM):** `UP_TO_DATE → KEEP`

### Round — Evaluate T3 (25 November 2024, 20:23 UTC)

#### B1 input — target: Text

**Input:** [Event, Date] from News + Text at T3 only.

##### Text at T3

The Qatar Open, currently known as the Qatar ExxonMobil Open for sponsorship reasons, is a professional tennis tournament played on outdoor hard courts. It is currently part of the ATP Tour 250 series of the Association of Tennis Professionals (ATP) Tour. It is held annually in January at the Khalifa International Tennis and Squash Complex in Doha, Qatar, since 1993. From 2025, this tournament will be upgraded to an ATP 500 level event. [ATP Tour announcement, 8 November 2023](https://www.atptour.com/en/news/atp-500-upgrades-from-2025).

**Gold output (not shown to LLM):** `UP_TO_DATE → KEEP`

#### B1 input — target: infobox/table

**Input:** [Event, Date] from News + infobox/table at T3 only.

##### infobox/table at T3

| Field | Value |
|---|---|
| **Tournament** | Qatar Open |
| **Founded** | 1993 |
| **Location** | Doha, Qatar |
| **Venue** | Khalifa International Tennis and Squash Complex |
| **Category** | ATP 250 / ATP International Series / ATP World Series (1993–20242); ATP 500 |
| **Surface** | Hard — outdoors |
| **Draw** | 32S (singles) / 16Q (qualification) / 16D (doubles) |
| **Prize money** | US$1,395,875 (2024) |
| **Website** | qatartennis.org |
| **Current champions** | 2024 |
| **Men’s singles** | Karen Khachanov |
| **Men’s doubles** | Jamie Murray / Michael Venus |

**Gold output (not shown to LLM):** `UP_TO_DATE → KEEP`

#### B1 input — target: KG

**Input:** [Event, Date] from News + KG at T3 only.

##### KG at T3

| Subject | Property | Value | Start time (P580) | End time (P582) | References |
|---|---|---|---|---|---|
| Qatar Open (Q299402) | Part of (P361) | [ATP World Series (Q3602248)](https://www.wikidata.org/wiki/Q3602248) | 1993 | 1999 | 0 |
| Qatar Open (Q299402) | Part of (P361) | [ATP International Series (Q2727525)](https://www.wikidata.org/wiki/Q2727525) | 2000 | 2008 | 0 |
| Qatar Open (`Q299402`) | Part of (`P361`) | ATP World Tour 250 series (`Q300017`) | 2009 (year) | 2024 (year) |
| Qatar Open (`Q299402`) | Part of (`P361`) | ATP World Tour 500 series (`Q300018`) | 2025 (year) | - |

**Gold output (not shown to LLM):** `UP_TO_DATE → KEEP`

## B2 — Static cross-modal

**Input:** Verified news triples + **one snapshot of each modality for all three modalities** at the same historical evaluation time. For each target prediction, the LLM receives that target and both other modalities, but **no earlier snapshots**.

**Output:** Three freshness labels and suggested actions, one for each modality.

### Round — Evaluate T1 using only T1

**Input:** [Event, Date] from News + Text, infobox/table, KG at T1.

#### Text at T1

The Qatar Open, currently known as the Qatar ExxonMobil Open for sponsorship reasons, is a professional tennis tournament played on outdoor hard courts. It is currently part of the ATP Tour 250 series of the Association of Tennis Professionals (ATP) Tour. It is held annually in January at the Khalifa International Tennis and Squash Complex in Doha, Qatar, since 1993. From 2025, this tournament will be upgraded to an ATP 500 level event. [ATP Tour announcement, 8 November 2023](https://www.atptour.com/en/news/atp-500-upgrades-from-2025).


#### infobox/table at T1

| Field | Value |
|---|---|
| **Tournament** | Qatar Open |
| **Founded** | 1993 |
| **Location** | Doha, Qatar |
| **Venue** | Khalifa International Tennis and Squash Complex |
| **Category** | ATP Tour 250 / ATP International Series / ATP World Series (1993–current) |
| **Surface** | Hard — outdoors |
| **Draw** | 32S (singles) / 16Q (qualification) / 16D (doubles) |
| **Prize money** | US$1,377,025 (2023) |
| **Website** | qatartennis.org |
| **Current champions** | 2023 |
| **Men’s singles** | Daniil Medvedev |
| **Men’s doubles** | Rohan Bopanna / Matthew Ebden |


#### KG at T1


| Subject | Property | Value | Start time (P580) | End time (P582) | References |
|---|---|---|---|---|---|
| Qatar Open (Q299402) | Part of (P361) | [ATP World Series (Q3602248)](https://www.wikidata.org/wiki/Q3602248) | 1993 | 1999 | 0 |
| Qatar Open (Q299402) | Part of (P361) | [ATP International Series (Q2727525)](https://www.wikidata.org/wiki/Q2727525) | 2000 | 2008 | 0 |



**Gold output (not included in the model input)**

| Target modality | Freshness label | Suggested action |
|---|---|---|
| Text | `UP_TO_DATE` | `KEEP` |
| infobox/table | `EMPTY` | `ADD` |
| KG | `EMPTY` | `ADD` |

### Round — Evaluate T2 using only T2

**Input:** [Event, Date] from News + Text, infobox/table, KG at T2.

#### Text at T2

The Qatar Open, currently known as the Qatar ExxonMobil Open for sponsorship reasons, is a professional tennis tournament played on outdoor hard courts. It is currently part of the ATP Tour 250 series of the Association of Tennis Professionals (ATP) Tour. It is held annually in January at the Khalifa International Tennis and Squash Complex in Doha, Qatar, since 1993. From 2025, this tournament will be upgraded to an ATP 500 level event. [ATP Tour announcement, 8 November 2023](https://www.atptour.com/en/news/atp-500-upgrades-from-2025).


#### infobox/table at T2

| Field | Value |
|---|---|
| **Tournament** | Qatar Open |
| **Founded** | 1993 |
| **Location** | Doha, Qatar |
| **Venue** | Khalifa International Tennis and Squash Complex |
| **Category** | ATP Tour 250 / ATP International Series / ATP World Series (1993–current) |
| **Surface** | Hard — outdoors |
| **Draw** | 32S (singles) / 16Q (qualification) / 16D (doubles) |
| **Prize money** | US$1,377,025 (2023) |
| **Website** | qatartennis.org |
| **Current champions** | 2023 |
| **Men’s singles** | Daniil Medvedev |
| **Men’s doubles** | Rohan Bopanna / Matthew Ebden |


#### KG at T2

| Subject | Property | Value | Start time (P580) | End time (P582) | References |
|---|---|---|---|---|---|
| Qatar Open (Q299402) | Part of (P361) | [ATP World Series (Q3602248)](https://www.wikidata.org/wiki/Q3602248) | 1993 | 1999 | 0 |
| Qatar Open (Q299402) | Part of (P361) | [ATP International Series (Q2727525)](https://www.wikidata.org/wiki/Q2727525) | 2000 | 2008 | 0 |
| Qatar Open (`Q299402`) | Part of (`P361`) | ATP World Tour 250 series (`Q300017`) | 2009 (year) | 2024 (year) |
| Qatar Open (`Q299402`) | Part of (`P361`) | ATP World Tour 500 series (`Q300018`) | 2025 (year) | - |


**Gold output (not included in the model input)**

| Target modality | Freshness label | Suggested action |
|---|---|---|
| Text | `UP_TO_DATE` | `KEEP` |
| infobox/table | `EMPTY` | `ADD` |
| KG | `UP_TO_DATE` | `KEEP` |

### Round — Evaluate T3 using only T3

**Input:** [Event, Date] from News + Text, infobox/table, KG at T3.

#### Text at T3

The Qatar Open, currently known as the Qatar ExxonMobil Open for sponsorship reasons, is a professional tennis tournament played on outdoor hard courts. It is currently part of the ATP Tour 250 series of the Association of Tennis Professionals (ATP) Tour. It is held annually in January at the Khalifa International Tennis and Squash Complex in Doha, Qatar, since 1993. From 2025, this tournament will be upgraded to an ATP 500 level event. [ATP Tour announcement, 8 November 2023](https://www.atptour.com/en/news/atp-500-upgrades-from-2025).


#### infobox/table at T3

| Field | Value |
|---|---|
| **Tournament** | Qatar Open |
| **Founded** | 1993 |
| **Location** | Doha, Qatar |
| **Venue** | Khalifa International Tennis and Squash Complex |
| **Category** | ATP 250 / ATP International Series / ATP World Series (1993–20242); ATP 500 |
| **Surface** | Hard — outdoors |
| **Draw** | 32S (singles) / 16Q (qualification) / 16D (doubles) |
| **Prize money** | US$1,395,875 (2024) |
| **Website** | qatartennis.org |
| **Current champions** | 2024 |
| **Men’s singles** | Karen Khachanov |
| **Men’s doubles** | Jamie Murray / Michael Venus |


#### KG at T3

| Subject | Property | Value | Start time (P580) | End time (P582) | References |
|---|---|---|---|---|---|
| Qatar Open (Q299402) | Part of (P361) | [ATP World Series (Q3602248)](https://www.wikidata.org/wiki/Q3602248) | 1993 | 1999 | 0 |
| Qatar Open (Q299402) | Part of (P361) | [ATP International Series (Q2727525)](https://www.wikidata.org/wiki/Q2727525) | 2000 | 2008 | 0 |
| Qatar Open (`Q299402`) | Part of (`P361`) | ATP World Tour 250 series (`Q300017`) | 2009 (year) | 2024 (year) |
| Qatar Open (`Q299402`) | Part of (`P361`) | ATP World Tour 500 series (`Q300018`) | 2025 (year) | - |


**Gold output (not included in the model input)**

| Target modality | Freshness label | Suggested action |
|---|---|---|
| Text | `UP_TO_DATE` | `KEEP` |
| infobox/table | `UP_TO_DATE` | `KEEP` |
| KG | `UP_TO_DATE` | `KEEP` |

## B3 — Temporal cross-modal

**Input:** Verified news triples + **historical snapshots of each modality for all three modalities** up to the evaluation time. No later snapshots are included. In this Qatar example, **T0 is included as a pre-announcement history baseline**; it never includes the future announcement as if it were known then.

- Round 1: T0 + T1 → Evaluate T1.
- Round 2: T0 + T1 + T2 → Evaluate T2.
- Round 3: T0 + T1 + T2 + T3 → Evaluate T3.

**Output:** Three freshness labels and suggested actions **at the evaluation time**, not retrospectively at each earlier snapshot.

---
# Experiment C — Which part of history helps?

**Input:** Start with B3, then make two controlled changes.

| Comparison | What it tells us |
|---|---|
| B3 vs. B2 | Does historical evidence help at all? |
| B3 vs. Ablation 1 | Does the history of other modalities help? |
| B3 vs. Ablation 2 | Does the chronology of historical updates help? |


- **Ablation 1 - Without cross-modal history:** Keep the target preprevious snapshots and all modalities at evaluation time; remove only the *other modalities’* earlier snapshots.

    **Original B3 — Full temporal history**
    
    | Snapshot | Text | Infobox | KG (target) |
    |---|---|---|---|
    | T0 | ✓ | ✓ | ✓ |
    | T1 | ✓ | ✓ | ✓ |
    | T2 | ✓ | ✓ | ✓ |
    | T3 | ✓ | ✓ | ✓ |
    
    The model sees all three modalities at all four time points.
    
    **Ablation 1 — Remove cross-modal history**
    
    | Snapshot | Text | Infobox | KG (target) |
    |---|---|---|---|
    | T0 | — | — | ✓ |
    | T1 | — | — | ✓ |
    | T2 | — | — | ✓ |
    | T3 | ✓ | ✓ | ✓ |

The model still sees KG history, but only the current text and infobox.


- **Ablation 2 - Without time order:** Instead of receiving `T0 → T1 → T2 → T3`, the model receives the historical snapshots in a shuffled order (e.g., `T2, T0, T1`) without their timestamps, followed by the current T3 snapshot.

  The model sees all three modalities at all four time points, but the historical snapshots are shuffled and their observation timestamps are removed, while T3 remains the current snapshot.
  
  **Example:**
  
  | Snapshot | Original B3 | Without time order |
  |---|---|---|
  | T0 | Before announcement | — |
  | T1 | Text updated | — |
  | T2 | KG updated | — |
  | T3 | Infobox updated | Current snapshot (T3) |





---

# Experiment D — Can the system suggest real maintenance edits?

If we can actually detect and publish some edit in the wild...


---
