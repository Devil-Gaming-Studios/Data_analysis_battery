# Data_analysis_battery
VoltRelay Energy
Battery-Swap Network Performance Analysis
Gradient Learnings Data Analytics Hackathon
Analysis Report — Data Analyst Submission
Analysis period: January 2024 – June 2025
Coverage: Bengaluru · Delhi NCR · Hyderabad · Pune · Mumbai · Jaipur
 
1. Problem Understanding
VoltRelay Energy operates a battery-swapping network for electric two- and three-wheelers across six Indian cities, serving gig and logistics riders who cannot afford to stop and charge for hours. Over the eighteen months covered by this dataset (January 2024 to June 2025), VoltRelay expanded its station network in two waves, raised its base pricing, piloted peak/off-peak pricing in two cities, onboarded a new battery supplier, and renegotiated its largest fleet contract.
Completed swaps and revenue both grew substantially over this period, but three warning signs emerged alongside that growth: service failures rose, new-rider retention fell, and per-swap profitability showed signs of erosion once quality is accounted for. Leadership needs a consolidated, evidence-based view of what is actually driving these outcomes before committing its next operating budget.
Objective
Determine how the network has performed over the available period; what factors are associated with service failures, rider churn, and eroding margins; where performance varies most across stations, geography, equipment, and partner segment; and what VoltRelay should prioritize as a result — using evidence from the data rather than assumptions.
Scope of the underlying data
Table	Rows	Grain
swap_events	3,877,013	1 row per swap attempt (completed or not)
station_hourly_status	1,487,712	1 row per station per hour
riders	20,000	1 row per registered rider
batteries	6,500	1 row per battery pack
support_tickets	44,000	1 row per customer support ticket
stations	152	1 row per swap station
city_daily_context	3,282	1 row per city per day
fleet_partners	12	1 row per B2B fleet customer
2. Analytical Approach
2.1 Data cleaning decisions
Each documented data-quality issue was handled deliberately rather than silently:
•	City spelling — Multiple spellings, abbreviations, and stray whitespace (e.g. "bengaluru " with a trailing space) were standardized to the six canonical city names before any geographic grouping.
•	Firmware timestamp bug — Events at stations running firmware v3.2.0 between 10 Mar and 14 Apr 2025 had timestamps corrected by +5h30m to fix a documented timezone logging bug, affecting 139,490 rows (3.6% of all events).
•	Near-duplicate retries — Consecutive same-rider, same-station, same-outcome events less than 120 seconds apart within offline-sync traffic were treated as retry duplicates and removed (580 rows, 0.015% of events).
•	Outlier handling — Negative or implausibly large km_since_last_swap readings (outside a 0–300km range) were set to missing rather than used as-is; SOC/SOH readings marginally above 100% (sensor drift) were capped at 100%.
•	Test stations — The two STN-TST-* internal test stations were flagged and excluded from all network-performance analysis.
•	Station telemetry gaps — Blank telemetry fields (where telemetry_status is "partial" or "missing") were left as missing rather than filled with zero, to avoid understating stock levels at exactly the poorest-connectivity stations.
2.2 Metric definitions
Contribution margin per swap is calculated as amount charged, net of energy cost (energy drawn × local grid tariff) and an allocated share of each station’s monthly rent and maintenance cost, split across that station’s completed swaps that month. This proxy intentionally excludes battery capital wear, which is examined separately and qualitatively in Section 3.
New-rider retention is defined over two windows, chosen to have full follow-up data before the dataset’s 30 June 2025 cutoff: a rider is "activated" if they complete at least one swap within 30 days of signup, and "retained" if an activated rider completes at least one further swap in days 31–90 after signup. The new-rider cohort is limited to riders who signed up between January 2024 and April 2025 to guarantee at least two months of observable follow-up.
2.3 Techniques used
•	Time-series trend analysis (monthly KPIs) for network performance.
•	Segmentation and cross-tabulation across station, city, hour, season, vehicle class, equipment generation, and connectivity tier.
•	Before/after and difference-in-differences comparisons for the base price change, the peak/off-peak pricing pilot, and the largest fleet contract amendment.
•	Cohort-based retention analysis with a standardized logistic regression to rank the relative importance of candidate churn drivers while controlling for confounds.
•	Supplier- and lot-level battery degradation analysis, controlling for battery age to avoid conflating "young" with "healthy."
3. Key Insight 1 — Network Performance Over Time
Completed swaps and revenue both grew strongly and consistently across the full period — completed swaps roughly tripled, from about 103,000 in January 2024 to over 300,000 per month by mid-2025, and revenue grew in step. On the surface, the simple contribution-margin-per-swap proxy also improved, rising from roughly ₹18–23 to ₹35+ per swap — driven mainly by a base list-price increase around July 2024.
 
Figure 1. Completed swaps, revenue, failure rate, and contribution margin per swap by month.
But the metrics do not all move together. Failure rate shows a clear recurring seasonal spike every April–June (rising from a ~4.3–4.5% baseline to 8–12%) in both 2024 and 2025 — a repeating, forecastable pattern rather than a one-off incident. And critically, average delivered battery health (state of health, or SOH) — the actual quality of the pack a rider receives — has declined steadily and substantially, from about 99% in January 2024 to about 77% by June 2025 — a 21-percentage-point drop over eighteen months.
This decline in fleet-wide battery health is not priced into any revenue or margin line today. It shows up instead as increased failure risk and reduced delivered range — a real and growing cost that the simple accounting margin does not yet reflect. Section 6 traces this back to a specific, fixable cause.
Bottom line: growth and headline margin look healthy, but failure rate and battery health — the leading indicators of future cost and churn — are moving in the wrong direction underneath a rising top line.
4. Key Insight 2 — Service Failures & Customer Experience
Service quality is not spread evenly across the network — it is heavily concentrated. Station-level failure rates range from roughly 3.5% at the best-performing stations to over 9% at the worst, around a network average of about 5.7%.
 
Figure 2. Failure rate by hour and season, average queue wait by hour, and the distribution of station-level failure rates.
Three patterns stand out:
•	Vehicle class — Three-wheelers fail at roughly double the rate of two-wheelers (about 10.4% vs 5.4%) — one of the largest, cleanest effects in the entire dataset.
•	Season — Failure rate is highest in summer (about 7.3%) and lowest in winter/post-monsoon (about 4.3%), consistent with a heat-related effect on battery charging and thermal performance.
•	Hour of day — Evening hours (roughly 19:00–22:00) run about 1–1.5 points higher than midday, but average queue wait time is essentially flat (~240–245 seconds) across every hour of the day.
That last point is a useful, somewhat counter-intuitive finding: busier periods show up as more failed or abandoned attempts, not as longer waits for riders who are served. The bottleneck looks like battery availability and equipment stress, not queueing capacity — which points the investigation toward station and equipment factors (Section 5) rather than staffing or throughput.
5. Key Insight 3 — Station & Geographic Patterns
Charger generation is a strong, clean driver of failure rate: Gen1 stations fail at about 7.2%, versus about 4.8% for Gen2/Gen3 — roughly 2.4 percentage points worse from equipment age alone.
 
Figure 3. Failure rate by city, charger generation, connectivity tier, and location type.
Geography compounds this rather than simply reflecting it. Failure rate by city ranks Jaipur and Delhi NCR worst (about 7.4–7.9%), Hyderabad next (about 6.7%), with Bengaluru, Mumbai, and Pune clearly best (about 4.5–5.0%). This gap persists even within the same charger generation — at Gen3 stations specifically, Delhi NCR and Jaipur still run about 0.7–0.8 points worse than Bengaluru and Mumbai, so equipment age alone does not fully explain the pattern.
The two effects are entangled by rollout history: Jaipur and Delhi NCR have a much higher share of older Gen1 stations from the original Launch wave, while Bengaluru, Mumbai, and Pune received a larger share of Gen2/Gen3 equipment in later expansion waves. Connectivity tier has only a mild, non-monotonic effect network-wide, though within the worst cities (notably Jaipur) poor-connectivity stations run meaningfully worse than good-connectivity ones — connectivity looks like a local amplifier rather than a primary network-wide driver.
Bottom line: the worst-performing part of the network is the combination of older Gen1 hardware concentrated in hotter, harder-climate cities — equipment age and climate compound each other rather than acting independently.
6. Key Insight 4 — Battery & Equipment Performance
One battery supplier, Kyron, stands out as a systematically underperforming cohort — and this does not look like a normal aging pattern.
 
Figure 4. Current SOH, degradation rate, SOH-vs-age by supplier, and retirement rate by supplier.
•	SOH — Average current state of health: Kyron about 64%, versus about 81% for both Cellora and Amptek.
•	Degradation rate — Kyron packs are actually younger on average (about 0.67 years in service) than Cellora (about 1.2 years), yet degrade roughly 2.5–3x faster — about 54 percentage points of SOH lost per year for Kyron, versus about 15–21 points/year for the other two suppliers, even after restricting to batteries with at least six months of tenure.
•	Retirement rate — About 88% of Kyron batteries have already been retired (end-of-life), versus roughly 0% for Cellora and Amptek.
•	Age correlation — Within Kyron, age and current SOH are strongly negatively correlated (about –0.61) — degradation is visibly accelerating with time in service — while Cellora and Amptek show almost no such relationship over the observed range.
This directly explains part of the fleet-wide SOH decline identified in Section 3: a meaningful share of it traces back to one specific, identifiable supplier cohort, not a diffuse "batteries just age" story. That makes it an actionable, fixable finding rather than an unavoidable cost of growth.
7. Key Insight 5 — Pricing & Partner Economics
 
Figure 5. Partner revenue and margin ranking, before/after contract amendment, and peak-hour demand shift.
•	Partner value is not just about volume — ZipDrop has the highest swap volume of any fleet partner, but a lower total revenue and a markedly lower margin percentage of revenue than FeastFly, which earns more from fewer swaps. "Largest partner" depends on which metric is used.
•	Contract amendment impact — ZipDrop’s contract amendment (contracted discount raised from 12% to 28%) visibly cut per-swap economics: average amount charged fell by roughly 15%, and average margin per swap fell by roughly 17%, comparing before versus after the amendment date. Total margin kept growing afterward only because network-wide volume growth outpaced the discount, not because the amendment was margin-accretive on a like-for-like basis.
•	Peak/off-peak pilot — The peak/off-peak pricing pilot (Bengaluru and Pune only, from around October 2024) shows real evidence of working: the share of swaps in peak hours declined slightly in pilot cities after launch while staying flat in non-pilot cities over the same period, and a difference-in-differences comparison shows pilot cities gained several additional rupees of margin per swap beyond what the network-wide base price increase alone would explain.
8. Key Insight 6 — Root Cause Analysis of Retention
Activation is not the problem: over 99% of new riders complete at least one swap within their first 30 days. Churn happens later — roughly one in eight activated riders (about 12–13%) never returns in the following two months. A standardized logistic regression, controlling for vehicle class, plan type, signup channel, home city, KYC status, first-swap wait time, and more, was used to rank the independent weight of each candidate driver.
 
Figure 6. Retention by first-attempt outcome, home city, vehicle class, and the top logistic-regression drivers.
Primary drivers
•	1. A failed first swap — Riders whose very first attempt does not complete are meaningfully less likely to return — one of the largest individual effects in the model. First impressions matter disproportionately.
•	2. Home-city service quality — Riders based in Jaipur, Delhi NCR, and Hyderabad — the same cities identified in Section 5 as having the oldest equipment and highest failure rates — retain measurably worse than Bengaluru/Mumbai/Pune riders. This directly connects the operational findings to the churn outcome leadership cares about.
•	3. Partner-onboarding acquisition channel — Riders onboarded directly through a fleet-partner employer retain worse than self-selected riders (app store, referral), consistent with employer-mandated sign-ups being less "sticky" than riders who chose the platform themselves.
Secondary / contributing factors
•	Vehicle class & plan type — Vehicle class (3W riders churn somewhat more) and plan type (partner-billed riders churn slightly more than pay-as-you-go) are real but smaller effects, and partly overlap with the primary drivers above.
•	KYC status — KYC verification status shows only a minor association with retention.
What does not appear to drive churn
•	Queue wait time — Queue wait time on the first swap has essentially no relationship with retention, consistent with Section 4’s finding that wait times barely vary across the network.
•	Early support tickets — Filing an early support ticket is positively, not negatively, associated with retention. This is very likely a confound — riders who use the platform more have more opportunities to encounter and report an issue — and is flagged explicitly here rather than presented as "complaints improve loyalty."
Bottom line: retention risk concentrates sharply among riders whose first experience fails and among riders based in the same equipment-and-climate-challenged cities flagged throughout this report — strong evidence that the service-quality problems identified above are a primary, not merely correlated, driver of churn.
9. Business Findings
What is working
•	Growth is real and broad-based: completed swaps and revenue have both roughly tripled over eighteen months, across all six cities.
•	Pricing actions are working as intended: the July 2024 base-price increase and the Bengaluru/Pune peak/off-peak pilot both show clean, measurable margin improvement, with the pilot delivering an incremental effect beyond the base price move.
What is not working
•	Service quality is deteriorating in a way the accounting margin does not yet reflect: average delivered battery health has fallen from about 99% to about 77% over the period, a real and growing cost not priced into any revenue line today.
•	Failures are concentrated, not evenly spread: three-wheelers fail at roughly double the rate of two-wheelers; Gen1 chargers fail at about 1.5x the rate of Gen2/Gen3; and Jaipur, Delhi NCR, and Hyderabad underperform Bengaluru, Mumbai, and Pune even within the same equipment generation.
•	One battery supplier (Kyron) is a specific, fixable outlier — not a general aging problem — responsible for a disproportionate share of the fleet-wide health decline.
•	Retention losses concentrate exactly where service quality is worst: a failed first swap and residence in the underperforming cities are the two largest independent predictors of a new rider not returning, directly connecting operational quality to the business outcome leadership is most concerned about.
•	The highest-volume fleet partner is not clearly the highest-value one once margin percentage and a recent contract amendment are accounted for.
10. Actionable Recommendations
In priority order, based on where the evidence is strongest and most actionable:
1. Fix the Jaipur / Delhi NCR / Hyderabad Gen1 station cohort first
This is where equipment age, climate, failure rate, and rider churn all point to the same root cause, making it the highest-leverage place to invest in charger upgrades and battery reallocation. Prioritize Gen1-to-Gen2/3 upgrades and better battery stocking specifically in these three cities before expanding elsewhere.
2. Address the Kyron battery cohort directly
Pause further Kyron procurement and accelerate retirement and replacement of the existing Kyron cohort. This is a specific, quantifiable, fixable contributor to the network-wide battery-health decline — not a cost that more capital spent on the current supplier mix will resolve on its own.
3. Roll out peak/off-peak pricing network-wide
The pilot’s evidenced, incremental margin lift and modest peak-hour demand shift make this a comparatively low-risk lever to pull next, once the equipment issues above are underway.
4. Hold off on a long-term fleet-partner exclusive
Until partner-level profitability — including battery wear and station cost, not just energy cost — is fully quantified. The highest-volume partner (ZipDrop) is not clearly the highest-value one on the evidence gathered here; committing to a long-term exclusive before that analysis is complete carries real downside risk.
5. Build seasonal (April–June) operational readiness
The failure-rate spike each summer is recurring and predictable rather than a surprise. Thermal- and battery-health interventions timed to this seasonal pattern — rather than reactive responses each year — should be built into standard operating planning.
Caveats and what would strengthen this analysis further
•	The contribution-margin proxy used here excludes battery capital wear; a fuller cost model incorporating battery replacement economics (especially for Kyron) would likely show more margin erosion than the simple metric suggests.
•	Retention definitions (30-day activation, 90-day return window) are a reasonable but not the only valid choice; a full cohort/survival-curve analysis would refine these estimates further.
•	csat_score was not used quantitatively given it is missing for the majority of tickets and is not missing at random; any future use should explicitly correct for resolution-time bias rather than averaging it naively.
