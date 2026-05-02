# 2SFCA Accessibility Score: What Changed and Why

## What the Code Does

The notebook calculates a pharmacy accessibility score for every Small Area Layer (SAL) in Gauteng and KwaZulu-Natal. The score helps us answer *how well-served is this SAL by nearby pharmacies, given how many other people are also competing for those same pharmacies?*

From my understanding after looking at the code and the methodology:

Step 1 looks at each pharmacy and asks how much demand pressure is this pharmacy under? It finds all the people living within a certain travel distance (catchment) and adds them up. A pharmacy surrounded by 50,000 people has more demand pressure than one surrounded by 500. The result is a supply-to-demand ratio for each pharmacy.

Step 2 looks at each SAL and asks what pharmacies can people here reach, and how stretched are those pharmacies? It adds up the supply ratios of all reachable pharmacies. A high score means good access and nearby pharmacies aren't overwhelmed. A zero score means no pharmacy is reachable within the travel distance at all.

The model runs four times with walking and driving networks for each province.

---

## Some Issues

1. SALs were being dropped from the count.

When the code mapped each SAL to a point on the road network, some SALs landed on the same road intersection. The old code `sal.set_index("node")["sal2023_est"].to_dict()` only kept one SAL's population per intersection and silently threw the others away. In a dense area where three SALs of 5,000 people each share a road node, the code counted 5,000 instead of 15,000, so this made pharmacies in dense areas look less busy than they actually are, which inflated their accessibility scores.

**Fix:** Changed the code to add up all SAL populations that land on the same road point instead of keeping only one with `.groupby("node")["sal2023_est"].sum().to_dict()`.

2. Extreme outlier scores were distorting everything.

When a pharmacy sits in a very sparsely populated area, its demand denominator is tiny so like maybe 3 people within the catchment. The code would calculate a supply ratio as if that pharmacy had enormous unused capacity and that inflated ratio then spread to every nearby SAL, which gave them wildly high accessibility scores. The worst case produced a score of 527 when the average was 0.1, which is a 5,000x distortion that's just the nature of the 2SFCA formula. So this compressed the entire rest of the score distribution toward zero and made it nearly impossible to distinguish meaningfully between SALs.

**Fix:** Added a minimum population floor of 50, so if a pharmacy's catchment population is below 50, the ratio calculation uses 50 instead. This prevents near-empty catchments from producing runaway scores while preserving the real signal for pharmacies that genuinely serve small populations.

HOWEVER, this comes with tradeoffs. It means that pharmacies in very remote areas will have their accessibility scores capped, which may understate the fact that they are the only option for those few people and p much obscures real variation at the low-demand end. But it was necessary to prevent the entire model from being skewed by outliers AND it gives us the ability to provide meaningful categorizations. This is part of the distribution most relevant to the rural KZN equity analysis, we could consider creating a different floor for KZN. Also, a floor of 50 applied to a rural KZN pharmacy near a small traditional settlement means something very different than the same floor applied to a Gauteng pharmacy on the urban fringe, so the threshold doesn't adjust for context. Creating a floor is also a documented practice in studies and not unusual to control crazy outliers.

Also the catchment population in question is decay-weighted and not a raw headcount. So a pharmacy might have 500 real people within 10 km but only 12 in decay-weighted terms because they're all clustered near the edge of the catchment. Those 500 people are not "sparse" in any meaningful sense as they're just far away. This means the floor treats distance-diluted demand the same as genuinely sparse demand, which is conceptually kinda muddled.

Also, the affected pharmacies are edge cases, so one with fewer than 50 decay-weighted people in its catchment is by definition in a very sparse area. These SALs are almost certainly already classified as Formal Desert or Pharmacy Desert, so capping their Rj at 50 likely (have to actually check) does not change their tier but just prevents them from contaminating the scores of every nearby SAL.

3. The distance weighting was too flat.

And onto the decay function lol, the model applies a decay function so pharmacies farther away should count for less than pharmacies nearby. The old setting barely distinguished between near and far, so a pharmacy 100 meters away got 100% weight, a pharmacy 10 kilometers away still got 37% weight, so the score was essentially just measuring *whether any pharmacy exists within range*, not *how conveniently located it is*.

**Fix:** Steepened the decay so that distance matters more. Under the new setting a pharmacy 1 km away gets 74% weight, 5 km gets 22%, and 10 km gets just 5%. This better reflects real behavior where people strongly prefer closer pharmacies and are unlikely to travel 10 km when one is 1 km away. Again, this is under the assumption that South Africans do behave and have lifestyles that reflect that.

4. No way to categorize results for policy use.

The raw scores are continuous numbers that are hard to communicate, so telling a policymaker that a SAL has an Ai score of 0.03 is not helpful.

**Fix:** Now that the floor was added, I added a classification system that assigns every SAL to one of five tiers:

| Tier | What It Means |
|------|--------------|
| **Formal Desert** | No registered pharmacy is reachable at all within the travel distance. |
| **Pharmacy Desert** | A pharmacy is technically reachable, but the area is in the worst-served third of the province. |
| **Average Access** | Middle third, baseline level of service. |
| **Served** | Upper third, good access relative to the province. |
| **Overserved** | Top 5%, very high access, often wealthy commercial areas. |

These tiers are calculated separately for each province because the two provinces use different travel distances by design where KZN is roughly multiple times larger and much less dense, so residents there might travel farther for FORMAL services, no knowledge on gray market pharmacies that sprung up to cover the formal gaps.

## BIG NOTE

This means that the Ai scores are NOT directly comparable between provinces. A SAL with a score of 0.1 in Gauteng is not necessarily better or worse than a SAL with a score of 0.1 in KZN because the underlying travel distances and population densities are different. The scores are only meaningful within the context of their own province's distribution. So we can say "This SAL is in the top 5% of Gauteng" or "This SAL is in the bottom third of KZN" but we CANNOT say "This SAL has better access than that SAL in the other province" based on the raw Ai scores alone. The tiers help with this by categorizing SALs relative to their own province's distribution, but we should be careful not to overinterpret the scores as absolute measures of access without considering the province context and the different assumptions that went into each model run. The scores are best used for identifying relative access within each province and for flagging areas that are particularly underserved or overserved compared to their province peers, rather than for making direct cross-province comparisons.

---

## What Was NOT Changed

The different travel distances between provinces (Gauteng uses 2 km walk / 5 km drive, KZN uses 3 km walk / 10 km drive) were kept. They reflect real differences in how far people travel for medicine in urban versus rural/traditional areas.

The uniform supply assumption where every pharmacy is treated as having the same capacity remains because there is no data on individual pharmacy staffing, hours, or inventory.

ANOTHER BIG NOTE: There is still truncation in the model that I want to get rid of. As background, when you load the KZN walk network, you're getting roads for the entire province that I saved and only that province, so my assumption was the truncation was for speed purposes. SO, the second truncation is not to the province boundary, but it's to the SAL bounding box of all the SAL centroids we're actually analyzing that further pares down the network. With ~14,000 SALs and ~2,000 pharmacies, the Dijkstra is running MANY times, so a KZN road network might have a bunch of nodes and a truncated KZN network might have either significantly less or somewhat less nodes (in the process of checking), but depending on how steep the truncation is it could be a speed boost, but it could also critically cut out roads people genuinely use.
---

## What Was Added as Future Notes

**Transit accessibility:** The model currently only considers walking and driving. Most South Africans rely on minibus taxis, which are invisible to the current analysis. A note was added flagging the potential to incorporate transit data (GTFS) as a third mode in a future iteration.

**NHI Readiness Index:** The accessibility score measures one dimension of pharmacy access. A note was added describing a future composite index that would combine spatial access with demographics, provider capacity, transit, and historical deprivation patterns to better reflect the NHI's equity goals.