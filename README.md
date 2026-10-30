# REFIT washing-machine disaggregation

Take-home assignment for the Junior Data Scientist role at Flock Energy: clean REFIT smart-home data, study how washing machines are used, and estimate a washing machine's power from the whole-house (aggregate) power at 1-minute resolution.

| Document | Content |
|---|---|
| [DATA.md](DATA.md) | Raw-data coverage and quality findings, cleaning steps, decisions about gaps and outliers, remaining concerns |
| [RESULTS.md](RESULTS.md) | Washing-machine EDA, plots, model results, validation approach, failure analysis, interpretation |
| [Recommendation.md](Recommendation.md) | One-page utility-facing summary (suitable uses and limits) and the answers to the Section 4 questions |

## What I built

Four notebooks, to be run in order. All numbers in the documents come from these notebooks.

| Notebook | What it does |
|---|---|
| `01_understanding_data.ipynb` | Reads the **raw** House 1 files and finds the problems: unsorted rows, duplicates, overlapping parts, gaps, missing values, glitch values, frozen readings. It never changes a file. |
| `02_cleaning.ipynb` | Turns the two raw House 1 files into one clean 1-minute table (assignment Section 1), with before/after plots. |
| `03_washing_machine_eda.ipynb` | Washing-machine EDA on the **official cleaned** data of six households (Section 2): cycle rule, usage by hour and weekday, cycle characteristics, three required plots. |
| `04_washing_machine_disaggregation.ipynb` | Estimates washing-machine power from the aggregate for House 1 (Section 3): always-off baseline, Ridge, Random Forest, XGBoost and LightGBM, chronological train/validation/test split, failure analysis. |

Section 4 (practical implications) is answered in `Recommendation.md`.

## Repository layout

```
README.md  DATA.md  RESULTS.md  Recommendation.md  requirements.txt
notebooks/   the four notebooks above
results/     the plots used in the documents (exported from the executed notebooks)
docs/        assignment PDF and the readme of the cleaned REFIT data
data/        NOT in the repository (see below)
```

## Setup and run

1. **Python and dependencies.** Python 3.11 or newer is required (the pinned numpy needs it); the notebooks were tested on Python 3.13.9. From the repository root, a virtual environment is recommended:

   ```bash
   python -m venv .venv
   .venv\Scripts\activate          # Windows
   source .venv/bin/activate       # macOS / Linux
   pip install -r requirements.txt
   ```

2. **Data.** The datasets are large and are not committed to the repository (`data/` is in `.gitignore`). Place the supplied folders in the following structure:

   ```text
   data/
   ├── REFIT_RAW_081116/
   │   ├── RAW_House1_Part1.csv
   │   ├── RAW_House1_Part2.csv
   │   └── ...
   └── CLEAN_REFIT_081116/
       ├── CLEAN_House1.csv
       └── ... CLEAN_House21.csv
   ```

   Files actually read: notebooks 1 and 2 use the two raw House 1 files; notebook 3 uses the cleaned files of Houses 1, 2, 3, 5, 6, 7, 9, 10, 11, 15, 16, 17, 18, 19, 20 and 21; notebook 4 uses `CLEAN_House1.csv` and, for one check, the raw House 1 files.

3. **Run the notebooks.** From the repository root, launch JupyterLab:

   ```bash
   jupyter lab
   ```

   Open the `notebooks/` folder and run the notebooks in order, 1 to 4, using **Run All**. Each notebook also runs on its own: none reads another notebook's output, and the order only follows the analysis. The notebooks must run with `notebooks/` as the working directory (their paths start with `../data`); JupyterLab does this automatically, and in other tools such as VS Code you may need to set it. If you are asked for a kernel, select the Python environment where the dependencies were installed (the files were saved with a kernel named `conda-base-py`, which will not exist on another machine).

4. **Runtime and memory.** Notebook 3 takes about 5 minutes the first time (it reads 16 large files and caches the prepared tables). Notebook 4 takes about 10 to 12 minutes; the Random Forest fits are the slow part. I did not time notebooks 1 and 2. Runtime depends on hardware. The two raw House 1 tables alone take about 0.8 GB in memory, so allow several GB of RAM.

5. **Generated files.** The notebooks create `data/processed/` themselves and write nothing elsewhere. Notebook 2 saves the clean House 1 table there. Notebook 3 caches one table per house in `data/processed/wm_eda/`; the cache file names contain the settings, and you should delete that folder if you change the preparation code. Notebooks 3 and 4 use the official cleaned REFIT data directly rather than notebook 2's output, because the assignment asks for it in Sections 2 and 3.

The raw and official cleaned dataset files are read-only inputs; the notebooks never modify them.

## Assumptions and preprocessing decisions

The full reasoning and the numbers are in [DATA.md](DATA.md) and [RESULTS.md](RESULTS.md). The thresholds below are my own analytical choices, not standards.

| Choice | Value | Why (short) |
|---|---|---|
| Washing-machine channel | readme appliance list (House 1: `Appliance5`) | Washer-dryer channels are different machines and are never used; Houses 4, 8, 12 and 13 are excluded (two machines, unverifiable mapping, no channel, machine replaced) |
| Resolution | 1-minute **mean** power; energy in Wh = sum of the minute values / 60 | The assignment asks for 1-minute resolution |
| Overlap of the raw parts | Part 1 up to its last timestamp, Part 2 after it | Both record the same house for about 3 months; stacking would double count |
| Duplicates | exact copies dropped; same timestamp with different values: keep the last | Simple, stated rule |
| Implausible appliance readings | above 4,000 W set to missing (appliances only) | Mostly powers of two (sensor glitch); the aggregate is not capped because a house can really use more |
| Missing values | gaps of 5 minutes or less filled with the last value; longer gaps stay missing | A short fill cannot invent a cycle; a long outage is never filled |
| Suspected frozen readings | 30+ minutes with every channel exactly unchanged: flagged in notebook 2, excluded in notebooks 3 and 4 | The official files are forward-filled, so a stalled logger looks like real data |
| Cycle rule (heuristic) | active above 10 W; periods up to 15 minutes apart are merged; must reach 1,000 W | Chosen from the data (two histograms); a detected cycle is not a confirmed programme |
| Household selection (notebook 3) | at least 400 valid days, 200 cycles, at most 5% `Issues`-flagged minutes; then the six with the least missing data | Reproducible rule, not hand-picked |
| `Issues` flag | kept, not removed | Most heating minutes are flagged; removing them removes most of the washes (RESULTS.md) |
| Clock | `Time` used as given | See the limitation in DATA.md; the hour-of-day plot uses winter months only |

## Validation design (notebook 4)

- **Chronological split**, 70% / 15% / 15% of the usable minutes, cut at timestamps. No minute is shared between parts, and nothing is shuffled.
- **Train** fits the models, **validation** chooses the settings, the model and whether an extra feature is kept, and **test** is scored once at the end. The test targets are not looked at before that.
- **No future information.** Every feature uses the current and earlier aggregate readings only. In a separate check (a script outside the notebooks) I rebuilt every feature from the history up to each minute for 404 sampled timestamps and found 0 differences; the notebook itself asserts that the lag and `delta` columns are aligned.
- **Missing values are never used** as targets or inputs, and are never treated as 0 W.
- **Baseline and metrics.** An always-off baseline (0 W for every minute), MAE and RMSE in watts, F1 for the active state (above 10 W, the same threshold for actual and predicted power) and the relative energy error over the test period. Negative predictions are clipped to 0 W.

## Trade-offs

- **Simple and explainable over complex.** Small feature set, small settings grids and standard models. The tuning is limited on purpose, and the best settings sit at the edge of the grids, so a wider search might change the results.
- **MAE against RMSE.** The models are trained to minimise squared error, so they predict small non-zero power when the machine is off. Always-off therefore keeps the lower MAE, while the Random Forest has the lower RMSE and the higher F1. I report all three and do not rank the models on one metric.
- **One household for the model.** This keeps the validation clean and easy to explain, but it shows nothing about other houses. The EDA covers six houses.
- **Heuristic cycles.** There are no programme labels, so cycles are defined by a rule and described as detected events.
- **Past-only features** are realistic for a live service, but a model that also sees the following minutes would probably recognise cycles better.

## Research used

The only reference used is the dataset citation required by the REFIT readme: D. Murray, J. Liao, L. Stankovic, V. Stankovic, R. Hauxwell-Baldwin, C. Wilson, M. Coleman, T. Kane and S. Firth (2015), *A data management platform for personalised real-time energy feedback*, 8th International Conference on Energy Efficiency in Domestic Appliances and Lighting. No other article was used for the method; the approach (1-minute resampling, event rule, chronological split, tree models) is my own and kept simple.

## AI tools used

I write the initial code and ideas myself, then use Claude Code (Anthropic, a Claude Sonnet model) to improve them: to refine and refactor the notebook code, to run checks (for example the independent metric and leakage checks above), and to help draft and tidy these documents.

## Possible improvements

- Test across households (train on some houses, test on unseen ones) and add confidence intervals.
- Evaluate at cycle level (detected cycles against true cycles), which the utility use case needs.
- Models that also see a window around each minute (for example Seq2Point) and a wider, validation-only search for XGBoost and LightGBM.
- Study why the `Issues` flag is so frequent during cycles and fix the clock question in the raw files.
- Test the approach on non-UK data and on 15 and 30-minute data (see Recommendation.md).
