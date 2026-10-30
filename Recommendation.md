# Recommendation: suitable uses, limits and practical implications

**Bottom line.** The model estimates washing-machine power from the whole-house (aggregate) signal at 1-minute resolution. It is good enough for an approximate customer-insight pilot (when and how often the machine runs), but not for billing or per-cycle energy accounting.

## 4.1 Practical opportunity: load-shifting advice

**Opportunity.** Tell customers on a time-of-use tariff when their washing machine runs and suggest running it in the low-tariff period. In the six EDA households a detected cycle used a median of 0.25 to 0.70 kWh, and households ran 3.5 to 10.8 cycles per week (House 1: about 4.7 cycles x 0.32 kWh, roughly 1.5 kWh per week). Shifting changes *when* energy is used (cost, peak demand), not *how much*, so the benefit is modest and depends on the tariff gap. It should be run as a customer-insight pilot, not promised as an energy saving.

**What the model supports.** On the House 1 test period the Random Forest had RMSE 97.0 W (always-off: 134.6 W), active-state F1 0.33 and an energy error of about +17%. Always-off still had the lower MAE (10.7 W against 12.5 W), and 7.7% of the minutes with the machine off were predicted as active (precision 0.20). The estimates can show an approximate usage pattern, not exact cycle energy or cost.

**How to measure whether it worked.**
- Randomised pilot with a control group that gets no advice; compare a baseline period with an advice period in the same season (difference-in-differences).
- Main metric: share of washing-machine energy used in the low-tariff window. Secondary: cost, peak-period demand, customer engagement. Check that total washing-machine energy does not rise.
- Check the model on a subset of homes with plug-level meters (cycle start-time error, detection precision and recall). I evaluated minute-level power only, not cycle-level detection.

## 4.2 Limitations

- **Overlapping signatures.** In the test period, 309 minutes with the machine off were predicted at 500 W or more, where the median aggregate was about 2.5 kW: another load of similar size looks like the washing-machine heater. Heating bursts are found (70 of 75 reach 500 W) but under-reconstructed (median prediction 990 W against about 2.3 kW).
- **Missing data.** About 13% of House 1's minutes have no usable washing-machine reading (longest outage 41.6 days). The official file forward-fills outages, so I excluded suspected frozen stretches (0.3% of minutes). The `Issues` flag is set in 64% of heater minutes, and in about 5% of active minutes the target is larger than the aggregate, so the sensors disagree.
- **Household differences.** One household and one chronological test period were evaluated. Across the six EDA households, median cycle duration ranged from 34 to 102 minutes and median cycle energy from 249 to 703 Wh, so performance should not be assumed to carry over.
- **UK to India.** Not tested. The model leans on the roughly 2 kW heating bursts of a UK machine. A machine that heats little or no water would show only low-power phases (about 30 to 270 W in our examples), hard to see against a base load of about 200 W. Other large loads (for example water heaters), power quality, usage schedules and tariffs may differ too. It needs re-validation, probably retraining, on Indian data.

## 4.3 15- or 30-minute smart-meter data

The heating bursts in our examples last about 3 to 12 minutes, so within one 15-minute interval or less. A 5-minute burst at 2.3 kW adds only about 0.8 kW to a 15-minute average, similar to other household loads, and a typical cycle (34 to 102 minutes) is only about 2 to 7 readings (1 to 3 at 30 minutes). The cycle rule (10 W threshold, 15-minute merge gap, 1,000 W peak) and the 1-minute `delta` and lag features would not work in their current form, so start and stop times and cycle stages would be much harder to find, and the washing machine harder to separate from other loads. Coarser data might still support daily consumption or time-of-day patterns, but precise appliance-level disaggregation would be much harder.
