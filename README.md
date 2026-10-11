<a id="top"></a>

# Telegram ID age research

[Русская версия](README_ru.md)

I wanted a date, preferably an exact day. Telegram age checkers already printed dates, sometimes with hours and minutes, so it seemed reasonable to find out where those numbers came from. A table, a hidden field, a server endpoint. Something I could check and put into a small offline script.

The script stayed small. The investigation did not.

I followed public datasets back to their collectors, inspected Telegram's schema, tried account metadata and old profile photos, unpacked Android clients, compared interpolation methods, and queried date-checking bots. Several apparent breakthroughs fell apart when I checked what the timestamps actually described. I also made mistakes of my own: mixed up channel IDs with user IDs, evaluated a model after cleaning its test data, and connected two unrelated bot settings because their names appeared in the same Java file.

This repository contains the resulting estimator as [main.py](main.py) and the chronological account below. It does **not** recover an exact registration date from every Telegram ID. The numbers embedded in the script are a fitted research snapshot, including sources with imperfect provenance. They are useful for an approximate date, not proof of when someone opened an account.

## Checking the sources and three new dates

The three supplied registration anchors are training data for the common interpolation curve, not a separate lookup. Every numeric ID follows the same `regdate()` path and returns an estimate with a range. No runtime exception returns a contributor's date verbatim.

The fit gives supplied registration anchors weight 10 and other rows weight 1. It removes four misleading source inputs, including authorization timestamps. The three training anchors are 8874594977 / 2026-06-17, 3941874 / 2013-12-31 and 5869015726 / 2023-01-21. Their provenance remains in anchors.json.

The common curve predicts 2026-08-29, 2013-12-31 and 2023-01-21 respectively. All three supplied dates fall inside their model intervals (91-day half-width before clipping to today). These are training checks, not independent validation.

Repeated held-out checks on the existing collection produced p50 42.58 days, p80 89.14 days and mean error 53.82 days across 223 distinct IDs (816 dependent repeated predictions). Compared with the original baseline, p80 improved slightly but p50 and mean error worsened. This does not establish a general accuracy improvement or a guaranteed interval. Full results are in validation.json.

`regdate(id, registration_month="02.2016", not_after="2016-02-20")` can use a known registration month and an existence upper bound. A photo is not a registration date. Contradictions are flagged rather than silently accepted. In Hotaru, `ctx.tools.regdate(id)` and `r.regdate` remain offline; `await r.regdate_live()` attempts additional Telegram evidence.

The additional public-source audit downloaded 46 pinned files and parsed 17 tables or sets of constants. No independently confirmed registration records were found. WizardLoop duplicates Jobians; lastochkin-group adds one unverified row, not imported into the model. Approximate estimator constants were not counted as new observations.

## Run the script

You need Python 3. There are no packages to install and no network requests.

```sh
git clone https://github.com/sepiol026-wq/telegram-id-age-research.git
cd telegram-id-age-research
python3 main.py 103151531 5000000000
```

Current output (age_days changes with the UTC date):

```text
103151531 {'date': '2015-05-30', 'earliest': '2015-02-28', 'latest': '2015-08-29', 'error_days': 91, 'age_days': 4152, 'estimated': True, 'source': 'model', 'conflict': False, 'registration_month': '', 'not_after': ''}
5000000000 {'date': '2021-11-30', 'earliest': '2021-08-31', 'latest': '2022-03-01', 'error_days': 91, 'age_days': 1776, 'estimated': True, 'source': 'model', 'conflict': False, 'registration_month': '', 'not_after': ''}
```

The range is empirical, not a guaranteed interval. Optional refinements: `python main.py 111040773 --month 02.2016 --not-after 2016-02-20`. Conflicting evidence is flagged, not silently accepted.

<details>
<summary>Read the investigation in order</summary>

1. [The month Telegram already exposes](#the-month-telegram-already-exposes)
2. [The first interpolation table](#the-first-interpolation-table)
3. [A timestamp belonging to the wrong event](#a-timestamp-belonging-to-the-wrong-event)
4. [Borrowing the wrong ID sequence](#borrowing-the-wrong-id-sequence)
5. [Finding the signup notices](#finding-the-signup-notices)
6. [Teaching the estimator to show uncertainty](#teaching-the-estimator-to-show-uncertainty)
7. [The profile-photo detour](#the-profile-photo-detour)
8. [Opening the Android clients](#opening-the-android-clients)
9. [More anchors and a better-looking mistake](#more-anchors-and-a-better-looking-mistake)
10. [The month field did not scale](#the-month-field-did-not-scale)
11. [Trying to separate the data centers](#trying-to-separate-the-data-centers)
12. [Rechecking the source of the dates](#rechecking-the-source-of-the-dates)
13. [The signup scan that returned nothing](#the-signup-scan-that-returned-nothing)
14. [Commercial answers and a broken comparison](#commercial-answers-and-a-broken-comparison)
15. [The TgDB conclusion I had to retract](#the-tgdb-conclusion-i-had-to-retract)
16. [What is inside main.py](#what-is-inside-mainpy)
17. [What would change the result](#what-would-change-the-result)
18. [Sources and publication notes](#sources-and-publication-notes)

</details>

## The month Telegram already exposes

The first useful result did not require an estimator. Telegram has a `registration_month` field in `peerSettings`. The [action-bar documentation](https://core.telegram.org/api/action-bar) describes registration information shown when someone contacts you for the first time. The constructor also has fields for phone country and recent name or photo changes.

The distinction matters because these values do not all describe the same event:

```text
registration_month  -> a month such as 08.2017
name_change_date    -> a timestamp for a name change
photo_change_date   -> a timestamp for a photo change
```

I retrieved month values through `messages.getPeerSettings`. Other users returned a settings object without the field. A missing optional field was not a parser failure. Telegram simply had not supplied it in that response.

A test between two accounts made the direction visible: after one account messaged the other, the recipient could see the sender's registration month. The reverse lookup did not return the same information. Later group-member scans were much less productive than looking through existing private conversations.

That gave me a useful per-account signal, but not the intended tool. An offline function with only an integer as input cannot ask Telegram for `peerSettings`. Even the online version needs a resolvable peer, usually including an access hash. A public username or an entity already known to the session can help with resolution. A bare number does not grant access to every user object.

I initially called this a contact gate. That wording was too neat. Adding someone to contacts did not reliably reveal the month, and the documented first-contact UI is more specific than membership in a contact list. The supported statement is that availability depends on the relationship and server response. I did not recover a complete rule for predicting it.

## The first interpolation table

The offline idea was ordinary interpolation. Collect pairs of user IDs and dates, sort by ID, find the two surrounding points, and estimate a timestamp between them.

```text
t = t0 + (t1 - t0) * (id - id0) / (id1 - id0)
```

For an actual chronological counter with reasonably stable issuance, that might work well. Telegram's broad age trend certainly made it tempting: an old small ID and a recent large ID generally belong to different periods.

The local dates did not always increase with the IDs. A larger ID could have an earlier reported date. Some of that could be allocation behavior. Some could be bad labels. Some could be a date-checking site publishing the output of another date-checking site. Before identifying those cases, fitting a curve through every point would just turn conflicting labels into sharp jumps.

I used isotonic regression, specifically PAVA, to produce a nondecreasing fitted sequence. When adjacent fitted values go backwards, PAVA merges their blocks and replaces them with a weighted average. It continues until the fitted dates no longer decrease.

That is a modeling choice. It does not establish that Telegram issues IDs monotonically. It also does not preserve each individual known date. If one account sits out of order relative to its neighbors, PAVA can move its fitted date considerably. This later happened even with a month supplied by Telegram itself.

The early artifact used 84 fitted points. The dataset then grew through versions with roughly 238 and 249 inputs. The exact counts changed because sources overlapped and some records were replaced. The final frozen generator uses 491 distinct positive IDs; the runtime keeps a smaller set of fitted breakpoints.

At this stage I already had a program that printed a timestamp. I had not earned the seconds it printed.

## A timestamp belonging to the wrong event

`account.getAuthorizations` returns `date_created` and `date_active` for authorizations. These are precise timestamps, and the name `date_created` is easy to overread when the whole investigation is about creation dates.

An authorization is a login session. Its creation time is not automatically the creation time of the account.

The early notes treated the oldest available authorization as a registration timestamp for apparently fresh accounts. That needs independent evidence that the authorization was established during signup. A recently logged-in old account can have only a recent session remaining. Revoked sessions do not stay available as a complete historical ledger.

I also misread an authorization TTL as a maximum session age. An inactivity expiration policy does not imply that every session disappears a fixed number of days after its original creation. Keeping a session active and leaving a session unused are different cases.

One particularly misleading result involved five IDs paired with authorization timestamps spanning 243 seconds. Their ID range spanned roughly 267 million. Later summaries called these five controlled BotFather creations. The earlier records show that they came from imported account authorizations instead. Those are different experiments. The five entries remain part of the historical calibration snapshot, which is one reason I do not advertise it as a clean ground-truth dataset.

Separate BotFather observations in the notebook did show newly created bots receiving IDs out of numeric order. That contradicts a simple increasing counter for those observations. It does not reveal the whole allocator, prove uniform randomness, or give a universal mathematical accuracy floor.

I had written much stronger conclusions at the time. Those conclusions do not survive the distinction between a signup event and a login event.

## Borrowing the wrong ID sequence

The next lead was a study of channel IDs by [TGScope](https://tgscope.io/rnd/telegram-channel-id-clock). The notebook records its analysis of 247,850 dated channels and an interpretation based on changing allocation ranges.

It suggested a different estimator: identify a range's active period and return that period instead of pretending that position inside the range is a precise clock.

The mistake was transferring channel boundaries to users. Both are integers, but Telegram distinguishes [peer ID spaces](https://core.telegram.org/api/bots/ids). Measurements of newly created channels and contemporary user IDs did not sit on the same scale. A channel around the billion-ID mark could be years older than a user near the same numeric value.

I revised the supposed transition boundary several times. First I borrowed the channel boundary. Then I moved it to a user range without enough evidence. Later inversion counts suggested a broad difference around the sparse gap between older IDs near 2.3 billion and newer data above 5 billion. None of those revisions amounts to recovering Telegram's production allocation algorithm.

The channel experiment still taught something useful: do not mix object types in the training set. Channel dates, user registration months, and authorization timestamps are not interchangeable observations just because they fit into `(integer, timestamp)` pairs.

I also investigated `channel.date`. Its meaning and the object returned need checking in the context of the API call and membership state. Cached entities complicated those probes. I did not keep channel dates as user-registration anchors, and this repository does not offer a channel-age mode.

## Finding the signup notices

The [Jobians/telegram-id-age](https://github.com/Jobians/telegram-id-age) collector was more informative than another table of numbers. Its `src/updateDataset.js` looks for a service message with `messageActionContactSignUp` in a conversation and takes the message date.

That is a traceable event source. The [Telegram constructor](https://core.telegram.org/constructor/messageActionContactSignUp) represents a contact signing up. It is a much better candidate for a day-level label than a date guessed from a neighboring ID.

There are still two separate checks. First, the collector needs to associate the service message with the right user. Second, a public dataset needs row-level provenance if every entry is to be treated as independently verified. Reading a collector tells me how it can gather data; it does not prove that every committed row was produced by that collector without edits.

The research notes recorded a small local collection of signup events. Later, another account's much larger scan returned none. Those results are not inherently contradictory: different contact histories can expose different messages. The problem was my tendency to label every successful collection complete and every unsuccessful one impossible. The later scan had a pagination flaw, described below.

The 212-entry public dataset became one of the main sources. My working copy later had 214 rows after additions. Those counts describe different files, not growth in the upstream repository.

## Teaching the estimator to show uncertainty

A timestamp alone was too easy to misread. I added an interval based on prediction errors around neighboring IDs.

The first version used leave-one-out evaluation: remove one point, fit on the rest, predict the removed point, record the error. Later I used repeated holdout splits. The generator shuffles the inputs, trains on 70 percent, and measures absolute errors on the remaining 30 percent. It repeats this for 12 seeds.

I caught a basic implementation bug here. PAVA needs the input ordered by ID. One version fitted a shuffled training subset without sorting it first, producing absurd errors. Sorting inside the fitting function fixed that bug. It did not fix the quality of the dates.

The uncertainty table uses a local collection of held-out errors. The code takes an index window extending 12 positions on either side of an anchor, including the anchor's position. That is up to 25 entries, despite the old description saying "24 nearest anchors." Near the edges there are fewer. Index neighbors are also not necessarily the closest points by numeric distance.

For each window, the generator chooses a value around the 80th percentile and stores whole days. The runtime interpolates that table along with the date table.

This explains how the interval was built. It does not promise 80 percent coverage for future Telegram accounts. Related datasets can leak the same underlying information across a random split. Different eras have different errors. Labels may themselves be estimates. I had written "80 percent of accounts fall inside" in the notebook; the experiment did not establish that population claim.

## The profile-photo detour

Profile photos looked promising because Telegram exposes photo dates with second resolution. If someone uploads their first avatar during registration and keeps it, the oldest visible photo can be close to signup.

A tiny early sample looked encouraging. The notes reported four of five comparisons within a day and one substantially later. I turned that into claims about a working day-accurate method for fresh accounts. That was premature. I had no way to establish that an arbitrary account's oldest remaining photo was its original photo.

People delete old avatars. Some leave an account without a photo for years. An account with one photo does not prove that it has only ever had one photo.

The usable relationship is an upper bound:

```text
account registration <= a photo upload belonging to that account
```

That is not an equality. Even a timestamp down to the second does not supply the missing relationship.

I tried adding photo dates to the training set anyway. An early small experiment worsened the curve. A later bank held 1,867 photo observations. Treating those dates as registration points pushed the recorded holdout median to about 355 days, compared with roughly 34 days for the existing mixed-source model. Old IDs with recently uploaded photos forced large parts of the monotone fit toward recent dates.

I removed those observations from the point-fit path. The local online refinement experiment could still use a photo as an upper bound for that same account. It could not turn the bound into an exact birthday. Nor could I safely propagate a photo bound from one ID to other IDs while simultaneously observing inversions in the supposed chronological order.

The published script does not fetch photos. This detour is included because it was one of the most attractive wrong turns in the investigation.

## Opening the Android clients

A Telegram fork displayed an account creation estimate with a time of day. I wanted to see whether the client had an API I had missed.

In the examined exteraless code, the profile screen called `ProfileDateHelper.getUserTime`. The helper loaded an `id_date.json` asset, found surrounding IDs, and performed linear interpolation. The UI used an approximate marker. The recorded asset contained 144 anchors.

The minutes on screen came from formatting the interpolated timestamp. There was no separate measurement that justified those minutes.

An AyuGram APK contained another implementation under `com.exteragram.messenger.regdate`. Full APK decompilation ran out of memory, so I inspected the relevant DEX files separately. `RegDateController` led to an asset named `extera/registration_dates.bin`.

The asset was encoded with invisible Unicode characters. `InvisibleEncryptor` used an eleven-symbol alphabet and separators to reconstruct bytes. The awkward part was a narrow no-break space, `U+202F`, which the decompiled output made easy to mistake for an ordinary space. Inspecting code points rather than trusting their rendering made the decode work.

The result was 150 ID/date pairs. The new file had limited overlap with the earlier fork tables, so it materially changed the fit. That was a genuine data discovery, but the data were still anchor values shipped by another estimator. A binary file inside an APK does not confer special authority on its contents.

The local fallback in `RegDateController` combined interpolation with the oldest available profile-photo date. The client also had a separate remote request path and a response type with flags such as `EXACT`, `INTERPOLATED`, `LT`, and `ET`.

Those flags describe categories understood by the client. They do not tell me how the server establishes truth, how much of its database is exact, or whether a subscription would improve this project. A developer-gated `addregdate` method proved that manually supplied records were supported. It did not prove that all remote records came from manual submissions or that the server used only the bundled 150 points.

## More anchors and a better-looking mistake

Combining the fork data with the earlier collection improved some recorded errors. The notebook passed through a 353-point version, then around 490 points after the decoded asset, and finally 491 after another accepted candidate.

The growth was less impressive than the number of repositories I searched. Several projects repeated the same table. WizardLoop's copy matched the Jobians data in the comparison. Other sources used conspicuously rounded IDs or dates inconsistent with nearby labels. Duplication is not independent evidence.

I tried removing outliers. The first result looked excellent: the reported upper error percentile almost halved. Then I checked the evaluation order. I had cleaned the whole dataset before dividing it into training and test portions. The future test labels had already influenced which points survived.

Repeating outlier removal inside each training split took most of the improvement away. The remaining changes depended on the threshold and were small enough that I did not keep the extra machinery.

Other experiments treated month observations as intervals, reduced weights for suspicious dates, shifted month labels toward their midpoint, or increased the weight of server-supplied months. None established a consistent improvement worth replacing the simple fit. The notes also record spline and density-based trials without a convincing gain. I am not publishing an unsupported leaderboard for those exploratory runs.

Increasing a month's weight did not even force the curve to respect it exactly. PAVA balances neighboring observations. If I know a specific account's month from Telegram, the answer for that account should use that fact directly rather than ask a smoothed curve to rediscover it.

## The month field did not scale

I returned to `registration_month` because it was still a direct server response rather than another interpolation table.

The early private-dialog probes returned some useful values. Larger scans of group participants mostly did not. The notebook's broad scan records 2,954 candidates and three month hits, all associated with private-dialog cases. That sample does not measure the fraction of all Telegram users whose month is visible; it describes those sessions and those peers.

An online refinement script could nevertheless improve an individual result. One recorded case had a server month outside the entire interval predicted by the offline curve. Intersecting the two mechanically would have produced an empty answer. The correct response was to flag the disagreement and prefer the directly supplied month for that account.

This was also a concrete counterexample to treating the model's interval as a hard bound. Adding an authoritative observation to a training set does not guarantee that a regularized fit will reproduce it.

Adjacent fields did not solve the day problem. `name_change_date` timestamps a name change. Inferring signup from it would require knowing that the user never changed their name and that the field's initialization semantics match signup. I had not established either. The same issue applies to `photo_change_date`.

## Trying to separate the data centers

Perhaps the apparent disorder came from mixing several allocation streams. If each data center issued IDs differently, splitting the dataset might recover a cleaner relationship.

I collected a bank of 690 observations containing photo-related data-center information. The preliminary comparisons did not produce a cleaner chronological sequence within a DC. Unfortunately, the test still used photo dates as a proxy for registration. Their lag was large enough to hide a possible allocation effect.

There was another mismatch: a photo's storage DC is not automatically the account's original registration DC. The field I could retrieve was not necessarily the field the hypothesis needed.

I left the hypothesis unconfirmed. A useful test would pair a reliably observed signup event with relevant account DC metadata at that time. Splitting a noisy photo dataset into smaller noisy datasets was not that test.

## Rechecking the source of the dates

The investigation kept returning to the same question: which entries were observations, and which were outputs of somebody else's model?

Several monotone assets had no date inversions at all. A signup-oriented dataset did have inversions, including in older periods. I initially treated perfect monotonicity as proof that an asset was synthetic. It is a reason to inspect provenance, but not proof by itself. A curated or filtered set could also be monotone.

The opposite inference was just as weak. A dataset with inversions is not automatically authentic. Wrong dates can create inversions too.

Historical leave-one-out runs separated older and newer ranges. The notebook reports median errors around 13 days on an older subset and around 44 days on a newer subset. These were not the same experiment as the final mixed-source repeated holdout, and the exact subset definitions changed during the work. They should not be presented as a single validated accuracy guarantee.

The older subset also contained reversals, especially around 2015. Describing every pre-2021 ID as strictly chronological was wrong. More accurate anchors may help, but one accurate old anchor cannot determine the issuance rate between all other IDs.

I inspected the history of the Jobians dataset, looked for forks and additional records, and checked another small creation-date repository. The extra older points did not deliver the hoped-for jump to day accuracy. A more elaborate interpolator cannot recover information that the input labels have not established.

There was also a unit error in an early issuance-rate calculation: I divided an ID difference by seconds per day rather than by elapsed days. Correcting the arithmetic changed the number dramatically. It did not make the rate a constant. Later attempts to infer a uniform allocation window from pairwise ID differences depended on assumptions that the observations did not validate. I have not retained the notebook's claimed "physical limit" as a result.

## The signup scan that returned nothing

A later collector visited 2,000 dialogs and reported no new `messageActionContactSignUp` anchors. That sounded like a closed avenue until I inspected how it jumped toward old history.

It estimated an old message ID from:

```text
top_message - count + 1
```

Message IDs are not a dense per-dialog array index. Deletions, gaps, and numbering across conversations break that calculation. The shortcut did not prove that the oldest messages had been visited. An earlier attempt using a very large `offset_id` had also misunderstood which end of the history the API would return.

Another bug looked only at the `chats` array from `messages.getDialogs`, overlooking users in the separate `users` array. That produced an empty candidate pool before the history scan even started.

The defensible result is that this traversal found no anchors. It does not prove that the account had none, and it says nothing about every other account. A completeness claim would require following actual pagination results until exhaustion, accounting for errors and inaccessible history.

I did not rerun private account collection just to publish this README. The repository ships the offline artifact and the findings, not session files or a collector that starts making Telegram requests.

## Commercial answers and a broken comparison

Date-checking bots were the next obvious comparison. Some offered monthly answers for free and advertised paid API methods. I queried a sample whose dates were already present in the research dataset.

The retained DateReg result file contains 20 rows. During the publication audit I found that one row had ID `0`, a dataset sentinel rather than a real account. It caused the previously reported maximum error of 625 days. That maximum should never have been used as evidence about account-date accuracy.

There were 19 positive-ID rows. None carried the sampled `exact` or `100%` label. Four answers contained the reference day within the returned calendar month.

The original comparison had converted each monthly answer to the first day of that month. That measures distance to an arbitrarily chosen representative day, not necessarily an error the service made. For example, a January answer for a January 22 reference is correct at month resolution, even though January 1 is 21 days away.

I recalculated both quantities from the saved file:

| Evaluation of the 19 positive-ID rows | Median absolute error | Maximum |
| --- | ---: | ---: |
| Treat answer as the first day of its month | 47 days | 183 days |
| Distance from reference day to the returned month interval | 19 days | 153 days |

In the second row, an answer containing the reference date has zero error at month resolution. It still does not identify the day. The sample is small, its labels have the provenance limitations already discussed, and the bot probe did not establish a rigorous message-ID correlation protocol. These are descriptive results from the retained responses, not a general service benchmark.

My earlier "twice as accurate" comparison mixed this sample with a different older-ID subset of my estimator. I withdrew it. To compare the two, both need the same independently verified accounts and an agreed scoring rule.

I did not fund or test the paid API tier. The free-chat results do not prove what that tier returns. Nor did I verify that an `Appeared` field means the first observation by a crawler. That was an interpretation, not established server behavior.

## The TgDB conclusion I had to retract

This was the most avoidable mistake in the whole write-up.

I found `tgdb_search_bot` in a decompiled `ChatUtils.java`, then announced that exteraGram's `regdate` requests went to TgDB. A live response from TgDB required a purchase, which made the story seem complete: supporter badge, bot request, paid database.

Reading the actual call chain disproved that story.

```text
RegDateController.fetchRegistrationDate
  -> ChatUtils.sendBotRequest
     -> ExteraConfig.getApiBotInfo
        -> setting: extera_api_bot
        -> default: exteraAuthBot
```

The TgDB name belongs to a different helper:

```text
ChatUtils.getBotInfo
  -> setting: search_bot
  -> default: tgdb_search_bot
```

The API-bot setting can be remotely overridden. The inspected default and call chain establish what this code does; they do not reveal every deployed remote configuration. They certainly do not establish TgDB as the registration-date backend.

The separate HTTP API did not close the gap. Profile endpoints returned badge-related data. A tested `regdate` route responded differently from missing paths and returned authorization errors. A `401` tells me I did not receive an authorized result. It does not reveal an exact-date database behind the endpoint. Neither does a `500` response prove the expected parameter contract or the quality of any stored data.

TgDB's purchase gate was real in the recorded interaction. Its advertised coverage of users and chats did not answer the question I cared about. A service can have extensive membership history without knowing when an account was registered. Even a reliable first-seen timestamp would normally be an upper bound on signup, not an equality.

I would not buy access on the strength of this investigation. Before recommending payment I would want sample results for independently known dates, a definition of each returned field, and a reason to expect those results to improve the offline estimator. I have none of those for the paid TgDB path.

## What is inside main.py

This section records the original 55f968d snapshot. Its counts and mixed holdout scores are historical; the source-audit section above and validation.json describe the current release.

The runtime is deliberately plain. It contains a date table, an uncertainty table, a binary-search interpolation function, and command-line output. The published file is the existing estimator renamed to `main.py`, without retraining or an unannounced algorithm change.

The publication check reconstructed the local generator output without overwriting it and compared the result byte for byte with the script. They matched.

| Snapshot property | Value |
| --- | ---: |
| Distinct positive input IDs in the generator | 491 |
| Date breakpoints embedded as `A` | 154 |
| Uncertainty entries embedded as `U` | 491 |
| Repeated holdout seeds | 12 |
| Training share per split | 70% |
| Mixed-source holdout median absolute error | 33.93 days |
| Mixed-source holdout 80th percentile | 76.71 days |
| Mixed-source holdout 90th percentile | 103.67 days |

Those metrics were recomputed for publication. They describe the local generator's full fitted representation, not a fresh independent population test. The runtime compresses that fit: it keeps a point when the fitted date has moved by more than a day since the last retained point, plus the endpoints. It then linearly interpolates between retained points. The published runtime is therefore not identical to evaluating every original PAVA step, and the table is not a separate measured score for that compression.

The input collection combines public data, decoded client anchors, signup observations, month labels represented as point timestamps, and historical authorization-derived labels. I would now require more provenance before calling such a collection a registration-date dataset. I have retained this snapshot so that the artifact matches the investigation rather than silently replacing it with a different experiment.

There is hardcoded calibration data. The program runs offline because those numbers are embedded. I did not discover a table-free arithmetic formula hiding in Telegram IDs.

The private training workspace is not included. It contains account-specific collection material, and publishing it unchanged would be inappropriate. This means the repository makes inference reproducible but does not provide a fully reproducible raw-data training pipeline. The source descriptions and algorithm are documented here so that this limitation is visible rather than hidden behind the word "research."

## What would change the result

The next useful contribution would be a new observation with provenance. A row should say what event was measured, how the account was identified, which timezone applies, and whether the timestamp is exact, rounded to a day, rounded to a month, or only a bound.

A signup recorded as it happens would be stronger than the oldest surviving login. A service-message timestamp with a verified user association would be stronger than a copied age-checker response. A calendar month should remain an interval rather than silently becoming its fifteenth day.

Evaluation also needs to separate sources, not just randomly split rows. If two repositories copied the same curve, placing one copy in training and the other in testing measures agreement between copies. A useful holdout would contain independent events absent from every source used to fit the model. Later time periods should be tested separately so that an aging calibration does not look current merely because old points still fit.

For a commercial service, I would start with the same independent holdout and compare like with like. A month prediction needs a month-aware score. An exact-date claim needs an exact-date check. The name of the pricing tier is not part of the measurement.

The result available today is the script above, with errors measured in weeks and months on the present mixed data. The exact-day goal remains unmet. Another source or a better experiment could change that assessment; another decimal place in the printed timestamp will not.

## Sources and publication notes

The chronological order follows the working notebook, with corrections placed beside the experiments they change. I have not assigned invented dates to stages whose exact timing was not recorded. Private account identifiers, credentials, access hashes, phone numbers, and session paths have been omitted from the narrative. The public script retains its numeric fitted calibration tables.

Primary protocol references:

- [Telegram action bar](https://core.telegram.org/api/action-bar), including registration metadata shown for incoming contacts.
- [`peerSettings`](https://core.telegram.org/constructor/peerSettings) and [`messages.getPeerSettings`](https://core.telegram.org/method/messages.getPeerSettings).
- [`messageActionContactSignUp`](https://core.telegram.org/constructor/messageActionContactSignUp).
- [`account.getAuthorizations`](https://core.telegram.org/method/account.getAuthorizations) and [`authorization`](https://core.telegram.org/constructor/authorization).
- [`photos.getUserPhotos`](https://core.telegram.org/method/photos.getUserPhotos).
- [Telegram peer IDs](https://core.telegram.org/api/bots/ids).

Data and implementations examined:

- [Jobians/telegram-id-age at `86605dc`](https://github.com/Jobians/telegram-id-age/tree/86605dcb20ece9f76d7c18a19a70026c9cf268cc), particularly the dataset collector and estimator. The checked upstream dataset has 212 rows.
- [exteraless/exteraless](https://github.com/exteraless/exteraless), particularly `ProfileDateHelper` and the account-date asset in the examined revision.
- The supplied AyuGram Android build containing exteraGram's `RegDateController`, `InvisibleEncryptor`, `ChatUtils`, and `ExteraConfig`. The account of those internals refers to that examined build, not a claim about every current client version.
- [TGScope's channel-ID study](https://tgscope.io/rnd/telegram-channel-id-clock), used as a channel-specific lead, not a user-ID specification.
- [C0dwiz/DateReg-Lib](https://github.com/C0dwiz/DateReg-Lib) and [DateReg API documentation](https://docs.datereg.pro/api), used to inspect the available API surface. Paid results were not tested.

The examined APK has SHA-256 `2ce48d69be32a6374c0168b27a0f063612ce82be0c5f9ebb7a31c5952d12f883`. It is not redistributed here.

This is a research snapshot, not an official Telegram service or an account-verification tool. It makes no claim to own the upstream datasets or client code cited above. There is no blanket relicensing of those sources in this repository.

## License

The original code and research text in this repository are licensed under [GNU AGPLv3](LICENSE) (`AGPL-3.0-only`). Referenced third-party datasets and client code retain their own licenses.
