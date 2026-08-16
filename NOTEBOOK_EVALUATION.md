# Stream Data Analytics notebook review

## Scope

This report reviews the three project notebooks:

1. [`Stream Data Analytics.ipynb`](Stream%20Data%20Analytics.ipynb) — Notebook 1, the original implementation.
2. [`improved Streat data analytics.ipynb`](improved%20Streat%20data%20analytics.ipynb) — Version 2, the causal scientific baseline.
3. [`Scientific Stream Data Analytics v3.ipynb`](Scientific%20Stream%20Data%20Analytics%20v3.ipynb) — Version 3, the cardiac-cycle extension.

The apparent targets are `S` (systolic blood pressure), `D` (diastolic blood pressure), `H` (heart rate), and `R` (respiratory rate). All notebooks use the first 20 minutes of the 100 Hz seismocardiogram (SCG) for development and predict the final 7 minutes one second at a time.

The project metric is:

$$
\text{score}=100-[MAE(S)+MAE(D)+MAE(H)+MAE(R)].
$$

The numerical score measures performance on this recording. It is not a clinical grade and does not measure generalization to new people, devices, postures, or recording sessions.

## Results at a glance

| Notebook | MAE S | MAE D | MAE H | MAE R | Sum MAE | Task score | Evaluation coverage |
|---|---:|---:|---:|---:|---:|---:|---:|
| Notebook 1 | Not reported separately at completion | — | — | — | 14.525 | 85.475 | Not explicitly reported |
| Version 2 | 2.364 | 1.200 | 5.164 | 1.532 | 10.260 | 89.740 | 398/420 seconds (94.8%) |
| Version 3 | 2.778 | 1.200 | 1.750 | 1.283 | 7.011 | 92.989 | 398/420 seconds (94.8%) |

Version 2 improves the observed score by 4.266 points over Notebook 1. Version 3 improves it by another 3.249 points over Version 2. Most of the Version 3 gain comes from heart rate: `H` MAE falls from 5.164 to 1.750 after explicit cardiac-cycle timing is introduced. `S` becomes 0.414 worse, `D` is unchanged to three decimals, and `R` improves by 0.249.

These comparisons are descriptive. Version 2 and Version 3 were developed sequentially and evaluated on the same final seven-minute segment, so that segment is no longer a pristine independent test of the whole research process.

## Notebook 1 — original implementation

### What it did well

- It correctly framed the work as multivariate regression from a 100 Hz SCG signal to one prediction per second for `S`, `D`, `H`, and `R`.
- It attempted a broad signal-processing pipeline: low-pass filtering, Isolation Forest artifact detection, interpolation, extrema and derivative features, rolling summaries, variational mode decomposition (VMD), Lasso-based selection, polynomial expansion, Ridge regression, and Random Forest comparison.
- It reserved the later part of the development period for chronological validation instead of randomly choosing the outer validation observations.
- It implemented the seven-minute prediction phase as a second-by-second loop and preserved a reproducible final summed MAE of 14.525.
- As an exploratory notebook, it identified several useful building blocks that informed the later versions.

### What needs improvement

- Preprocessing is learned or computed over the complete 20-minute development signal before the internal validation split. In particular, batch `filtfilt`, Isolation Forest fitting, and VMD allow validation-period signal structure to influence validation features. This makes the reported validation errors optimistic for a genuinely forward stream.
- `LassoLarsCV`, `RidgeCV`, and `GridSearchCV` use ordinary cross-validation defaults. Those folds do not preserve time order. Blocked or forward validation is more appropriate for dependent time-series evaluation.
- Feature selection is run separately for each target, but the final polynomial transformer uses only the features selected for `S` and applies them to all four targets. The selected predictors are `is_peak1`, `diff2_49`, clock minute, and clock second; therefore, the final model largely abandons the richer VMD and morphology work.
- Clock minute and second can encode the location within this particular recording. They may predict gradual label drift without learning a transferable relationship between physiology and SCG.
- Final-stream labels are accessed inside the prediction loop to print live errors. They are not passed into the fitted model, but this design weakens test-set discipline and permits human feedback during evaluation.
- A broad `except Exception` treats missing labels and unexpected program defects alike. Failed seconds are skipped without a final coverage audit, so the score is harder to verify.
- A runtime package installation and removed pandas/scikit-learn APIs (`squeeze`, `Series.append`, and `normalize`) make the notebook fail in current environments.
- It provides no uncertainty interval, no explicit baseline comparison at final evaluation, and no ablation showing whether VMD or artifact removal contributes to performance.

### Assessment

Notebook 1 is a useful exploratory prototype and historical baseline. Its observed task score is **85.475**, but the validation design, clock predictors, inconsistent feature-selection deployment, and incomplete evaluation audit prevent that number from being treated as a reliable estimate of future performance.

## Version 2 — causal scientific baseline

### What it did well

- It audits the data explicitly: 1,200 development seconds, 420 stream seconds, 1,156 completely labelled development seconds, and 398 completely labelled stream seconds.
- Its 199 features are available after each completed second and use causal filters and trailing windows. Artifact-clipping parameters are learned only from development data, and no clock-derived predictors are present.
- The features cover robust amplitude statistics, derivatives, line length, cardiac and morphology frequency bands, spectral entropy and energy, peak statistics, a cardiac-rate estimate, and trailing summaries over 5–60 seconds.
- It compares a compact set of regularized linear and tree models using three expanding-window validation folds. The winner, Ridge with `alpha=100`, is selected using the same sum-of-MAEs objective as the official score.
- It generates all 420 predictions before accessing the final labels, then reports missing-label coverage symmetrically across targets.
- It reports a moving-block bootstrap interval for the score, `[88.412, 90.906]`, which is more informative than a point estimate alone.
- It is reproducible with current libraries, interprets standardized Ridge coefficients, and completes the simulated 420-second stream in approximately 0.88 seconds in the recorded run.

### What needs improvement

- One multi-output model and one shared representation are used for all targets even though blood pressure, heart rate, and respiratory rate have different physiological relationships with SCG. The final `H` MAE of 5.164 is the clearest limitation.
- The feature space is large and highly correlated relative to only 1,156 fully labelled development seconds. Ridge regularization controls this reasonably, but the scientific meaning of individual coefficients remains uncertain.
- Peak-rate and band-energy features only approximate cardiac cycles; the notebook does not explicitly align, normalize, or compare complete beats.
- The bootstrap interval describes uncertainty within this one dependent recording. It does not estimate between-subject or between-session uncertainty.
- The study still has no external subject, recording, sensor placement, or posture for validation.

### Assessment

Version 2 is the strongest general-purpose baseline. Its main achievement is methodological: it removes obvious temporal leakage, removes clock shortcuts, separates model selection from final evaluation, and makes coverage and uncertainty visible. Its observed task score is **89.740**.

## Version 3 — cardiac-cycle model

### What it did well

- It turns the known repeating mechanical heartbeat structure into explicit predictors instead of relying only on generic spectral summaries.
- A trailing 20-second morphology window is used to detect envelope anchors. Valid inter-beat intervals define complete cycles, which are interpolated to 32 phase positions and summarized by robust templates.
- Its 61 cycle features include inter-beat timing, direct cycle-derived heart rate, cycle count and validity, amplitude, consistency, template shape, and first-half/second-half morphology. Combined with Version 2 features, the model has 260 predictors.
- It allows target-specific representations and estimators. Forward validation selects base Ridge models for `S` and `D`, direct cardiac-cycle rate for `H`, and combined Extra Trees for `R`.
- The direct cycle-rate estimator is scientifically interpretable and produces the largest practical gain: `H` MAE improves by 3.414 relative to Version 2.
- It retains the strict prediction-before-evaluation protocol, explicit 398/420 coverage, progress timing, and a moving-block bootstrap score interval of `[92.133, 93.834]`.
- Its final task score of **92.989** is the best observed result among the three notebooks.

### What needs improvement

- Cycle anchors are peaks of a Hilbert-envelope representation, not ECG R-peaks or manually confirmed aortic/mitral valve events. The cycles are plausible recurring mechanical beats, but their physiological landmarks have not been validated.
- The labels “systolic” and “diastolic” for the first and second halves of a phase-normalized template are convenient engineering summaries. Without reference annotations, they should not be interpreted as exact systolic and diastolic intervals.
- Filtering a complete trailing window with `sosfiltfilt` uses no samples beyond the prediction time, so the endpoint prediction remains temporally valid; however, it is a recomputed window method rather than a stateful real-time filter and has boundary effects.
- Target-specific selection searches more model/feature combinations on only three validation blocks. This improves flexibility but also increases model-selection optimism.
- Version 3 was designed after Version 2 had already been evaluated on the same final seven minutes. The 3.249-point gain is therefore promising exploratory evidence, not a confirmatory out-of-sample result.
- The same single-record limitation remains. No conclusion about clinical blood pressure, heart-rate, or respiratory-rate estimation should be made without new subjects and synchronized reference measurements.

### Assessment

Version 3 is the best research candidate and the best-performing notebook on the available recording. Its cycle-derived heart-rate result strongly supports the cardiac-cycle hypothesis, but ECG/reference validation and a new untouched test cohort are required before claiming generalization.

## Comparison with the approach suggested in `readme.txt`

The [`readme.txt`](readme.txt) combines mandatory requirements with optional implementation ideas. Its mandatory core is the 20-minute development period, seven-minute per-second prediction stream, meaningful signal-processing or machine-learning model, and summed-MAE score. The VMD, SWT, Matrix Profile, Grafana, autocorrelation, envelope, and linear-regression items are presented as potential methods rather than a single required algorithm.

All three notebooks satisfy the basic non-trivial modelling requirement. Version 2 and Version 3 satisfy the streaming and evaluation protocol most rigorously. None creates a separate training, validation, and test partition entirely inside the first 20 minutes; Notebook 1 has a chronological training/validation split, while Versions 2 and 3 use expanding-window validation and treat the final seven minutes as the final holdout.

| README recommendation | Notebook 1 | Version 2 | Version 3 |
|---|---|---|---|
| First 20 minutes for development; final 7 minutes for streaming | Mostly compliant | Fully compliant | Fully compliant |
| Separate train, validation, and test sets inside the first 20 minutes | No; training and validation only | No; expanding validation only | No; expanding validation only |
| Signal processing plus regression | Yes | Yes | Yes |
| Low-pass filtering or wavelet denoising | 3 Hz low-pass filter | Causal 0.7–4 Hz and 4–30 Hz bands | Same bands plus cycle processing |
| Automatic artifact handling | Isolation Forest and interpolation, fitted before validation | Development-only robust clipping | Robust clipping plus cycle-window artifact fraction |
| Stable envelope extraction | Local extrema, without demonstrated stability | Peak, spectral, and autocorrelation features; no explicit envelope | Trailing Hilbert envelope and beat anchors; closest implementation |
| Grafana or equivalent full-duration stability assessment | No | Whole-record signal diagnostics, but no envelope dashboard | Cycle diagnostic window, but no full-duration stability dashboard |
| Autocorrelation | Exploratory plot only | Used for heart-rate features | Cycle timing largely replaces it |
| Similarity or template analysis | No | No explicit template | Robust median cycle template and consistency features |
| Matrix Profile motif/discord search | No | No | No |
| VMD, SWT, or other decomposition | Batch VMD is computed but not used by the final stream model | Deliberately omitted | Omitted in favour of band-pass and Hilbert-envelope processing |
| Moving one-minute stacking or summaries | One-second moving average only | Trailing 5, 15, 30, and 60-second summaries | Same summaries plus trailing 20-second cycle windows |
| Feature points derived from peaks and timing | Simple extrema and derivative locations | Peak spacing, FFT, autocorrelation, and band features | Explicit inter-beat intervals and phase-normalized cycle features |
| Start with simple linear regression | Ridge is used, but with clock and weak signal predictors | Regularized multi-output Ridge wins forward validation | Ridge remains the base; direct cycle rate and Extra Trees win for selected targets |
| Specific cardiac-event annotation | No | No | Partial; recurring cycles are detected, but AO, AC, MC, and MO are not annotated |
| Missing-label audit | Failed seconds are skipped without a coverage summary | Explicit 398/420 coverage | Explicit 398/420 coverage |
| Causal one-second deployment | Partial because preprocessing and decomposition are batch operations | Strong | Strong at the prediction endpoint, although cycle windows are recomputed |

### Notebook 1 versus the suggested method

Notebook 1 resembles the README checklist most literally because it contains low-pass filtering, Isolation Forest, extrema, VMD, Lasso, Ridge, and Random Forest. The correspondence is mostly superficial, however. VMD-derived features do not reach the final deployed model, the polynomial transformer uses only the features selected for `S`, and minute and second become predictors for every target. Its preprocessing and ordinary cross-validation also weaken the forward-stream interpretation. It therefore implements many suggested components without integrating them into a reliable version of the proposed system.

### Version 2 versus the suggested method

Version 2 follows fewer optional tips but implements the core task more defensibly. Causal cardiac and morphology bands replace batch VMD; development-only robust clipping replaces a globally fitted anomaly model; expanding validation replaces ordinary cross-validation; and trailing multiscale features replace manual Grafana inspection. Ridge provides the requested simple regression baseline. Its main gap relative to the README is that it estimates peak rate and spectral structure without explicitly segmenting stable heartbeat envelopes or complete cycles.

### Version 3 versus the suggested method

Version 3 is closest to the README's physiological intention. It detects recurring beats through a Hilbert envelope, measures inter-beat intervals, phase-normalizes complete cycles, builds a robust template, quantifies cycle consistency, and preserves 60-second summaries. This explains why `H` MAE decreases from 5.164 in Version 2 to 1.750 in Version 3.

Version 3 is not equivalent to the papers suggested by the README. The cited VMD method decomposes SCG, constructs and smooths a heart-rate envelope, and annotates characteristic envelope points. Notebook 1 implements only the decomposition stage, while Version 3 reaches the cycle-extraction objective through a different Hilbert-envelope method. Similarly, the cited systolic-time-interval method uses a sliding template to locate particular SCG peaks. Version 3 summarizes phase-normalized beats but does not confirm aortic or mitral valve events.

### Missing elements and their priority

1. **Abnormal-period detection:** all versions operate mainly at the sample level. The README asks for detection and removal of corrupted periods; a future version should identify low-quality intervals using artifact fraction, cycle validity, and template consistency before prediction.
2. **Full-duration envelope validation:** Grafana itself is optional, but a quantitative 27-minute view of anchor density, valid inter-beat intervals, cycle consistency, artifacts, and missing predictions is still needed.
3. **Matrix Profile:** none of the notebooks performs motif or discord analysis. Matrix Profile is relevant for repeated-pattern discovery and abnormal-subsequence detection and can be updated incrementally, but a streaming implementation must prevent future samples from entering current features.
4. **Confirmed cardiac landmarks:** Version 3 cycles are plausible mechanical beats, not validated ECG R-peaks or annotated AO, AC, MC, and MO events.
5. **Independent evaluation:** another tuning pass on the same final seven minutes would be less valuable than freezing the current pipelines and evaluating them on a new synchronized recording.

The README-alignment ranking is therefore:

1. **Version 3:** closest to the intended physiological envelope/cycle method and best observed score, `92.989`.
2. **Version 2:** cleanest causal and reproducible implementation of the core streaming task, `89.740`.
3. **Notebook 1:** broadest literal checklist coverage but weakest methodological integration, `85.475`.

## Recommended role for each notebook

| Notebook | Recommended role |
|---|---|
| Notebook 1 | Preserve as the historical exploratory prototype. Do not use its validation design as the production template. |
| Version 2 | Use as the reproducible causal baseline and benchmark for future experiments. |
| Version 3 | Use as the leading cycle-aware research model, subject to confirmatory validation. |

The best next experiment is not another tuning pass on the same seven minutes. Freeze Version 2 and Version 3, collect or reserve a new synchronized recording, validate detected cycles against ECG or expert annotations, and compare the frozen models with a paired blocked analysis. Subject-level splits should be used as soon as multiple participants are available.

## Scientific basis

- SCG is a non-invasive measurement of cardiac-induced chest vibrations, and the field literature emphasizes both its clinical promise and its sensitivity to signal complexity, noise, morphology, and acquisition conditions: Taebi et al., [“Recent Advances in Seismocardiography”](https://doi.org/10.3390/vibration2010005), 2019.
- SCG contains respiratory information through heartbeat intensity modulation, within-beat timing, and between-beat timing: Pandia et al., [“Extracting respiratory information from seismocardiogram signals acquired on the chest using a miniature accelerometer”](https://doi.org/10.1088/0967-3334/33/10/1643), 2012.
- SCG landmarks are related to mechanical cardiac events, but accurate automatic annotation is difficult because morphology varies and the signal is noise-prone; sliding ensemble templates are one validated strategy: Shafiq et al., [“Automatic Identification of Systolic Time Intervals in Seismocardiogram”](https://doi.org/10.1038/srep37524), 2016.
- ECG-independent heartbeat timing and inter-beat intervals can be estimated from SCG using a Hilbert-transform approach, supporting Version 3’s cycle-rate hypothesis: Jafari Tadi et al., [“A real-time approach for heart rate monitoring using a Hilbert transform in seismocardiograms”](https://doi.org/10.1088/0967-3334/37/11/1885), 2016.
- Time-series model selection should respect temporal dependence; blocked validation is recommended over ordinary randomly structured cross-validation: Bergmeir and Benítez, [“On the use of cross-validation for time series predictor evaluation”](https://doi.org/10.1016/j.ins.2011.12.028), 2012.
