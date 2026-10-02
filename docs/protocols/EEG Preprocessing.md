# EEG Preprocessing

(**Note:** These whole preprocessing steps are for reference purposes only. Please verify the complete preprocessing steps with Prof. Vignesh before implementing.) 

## 1\. Purpose and Pre-Processing Philosophy

Preprocessing has two objectives: **data transformation** (converting raw data into a computable, analyzable format) and **artifact handling** (removing or attenuating non-neural contamination, such as eye movements, muscle activity, and electrical noise).

There is no single universal pipeline. Steps should be selected based on 

1. Data quality/noise level  
2. The intended analysis (ERP vs. oscillatory vs. connectivity vs. BCI classification)   
3. Requirements of downstream algorithms (e.g., ICA needs high-pass filtering first)  
4. Comparability with prior literature.

Document every parameter used at every step \- this is required for reproducibility per the OHBM **COBIDAS-MEEG** reporting guidelines (Pernet et al., 2020, *Nature Neuroscience*) and the Keil et al. (2014) EEG/MEG publication guidelines committee report.

**Note the ongoing debate**: Delorme (2023, *Scientific Reports*, "EEG is better left alone") argues excessive preprocessing can be unnecessary or harmful for very clean data, while de Cheveigné (2023) and Larsen & Versace (2024) counter that preprocessing remains essential for noisier/complex datasets (e.g., mobile EEG, motor imagery paradigms with muscle/movement contamination). 

## 2\. Pre-Preprocessing: Data Quality at Acquisition (Prevention)

Preprocessing quality is capped by acquisition quality. Before touching the data:

- [ ] Confirm channel montage/coordinates were imported correctly;   
- [ ] Electrode position accuracy matters for interpolation, referencing, and source work  
- [ ] Keep electrode impedances low \- below 10 kΩ for passive electrodes; active electrodes can tolerate \>20 kΩ without quality loss (Kappenman & Luck, 2010, *Psychophysiology*).  
- [ ] Log all sources of technical artifact during the session (line noise, cable movement, static discharge, electrode pops), since these inform later cleaning decisions.  
- [ ] For mobile/motor tasks (walking, cycling, motor imagery with limb movement), expect rhythmic motion artifacts synchronized to movement frequency \- plan extra EMG/movement channels or accelerometry if possible.

## 3\. Step-by-Step Preprocessing Pipeline

### Step 1: Initial Data Inspection

-  Verify all channels/coordinates loaded correctly; check for flat or non-functioning channels and mislabeled/missing event markers.  
- Standardise/rename event markers for consistency; discard channels/segments irrelevant to analysis.

### Step 2: Resampling (if needed)

- Downsample high acquisition rates (e.g., 1000 Hz) to reduce file size/computation. For analyses up to the Beta band (\~30 Hz), 250–500 Hz is typically sufficient; EMG, fMRI-gradient, or TMS-pulse correction require higher rates first, then downsample afterwards.  
- **Always low-pass filter before downsampling to prevent aliasing** — many toolboxes do this automatically, but verify.

### Step 3: Filtering

- Apply non-causal, zero-phase filters (preserves phase of retained frequencies):  
  * **High-pass:** 0.1–0.5 Hz to remove slow drifts (body sway, skin potentials).  
  *  **Low-pass:** \~45 Hz cutoff for typical cognitive/oscillatory EEG (removes muscle/high-frequency noise); adjust upward if gamma-band or EMG content is of interest.  
  * **Notch filter:** 50 or 60 Hz (per your region's mains frequency) plus harmonics, for line noise.  
* Choose FIR or IIR based on your needs; higher filter order \= steeper roll-off but reduced temporal precision. Use the half-power (−3 dB) or half-amplitude (−6 dB) cutoff convention consistently and report which one you used.  
* Filter minimally and only as needed \-  filtering can distort data in non-obvious ways (de Cheveigné & Nelken, 2019, *Neuron*; Widmann et al., 2015; Zhang et al., 2024).

### Step 4: Bad Channel Detection and Interpolation

- Identify flat, noisy, or persistently artifactual channels using visual inspection or automated metrics (e.g., within the **PREP pipeline;** Bigdely-Shamlo et al., 2015, *Front. Neuroinform.*).  
- Interpolate using spherical spline interpolation (standard default in most toolboxes). Use sparingly \- over-interpolation can distort true brain activity; there's no fixed cap, but keep it minimal and report the count/percentage interpolated.  
* **Order matters:** interpolate bad channels *before* applying an average reference to preserve balanced spatial sampling; also, running ICA before interpolation/re-referencing avoids rank-deficiency problems.

### Step 5: Re-referencing

- Choose a reference scheme deliberately: online reference proximity can suppress amplitude near that site; use of **Average Reference** improves SNR but requires ≥64 channels with even scalp coverage and no remaining bad channels (Bigdely-Shamlo et al., 2015; Hu et al., 2018, *J Neural Eng*; Tsuchimoto et al., 2021, *J Neurosci Methods*).  
- For mastoid-referenced setups, average the left/right mastoids to avoid hemispheric bias.  
- Match your reference scheme to the literature you intend to compare against.

### Step 6: Independent Component Analysis (ICA) for Stereotyped Artifacts

Used to correct eye blinks, eye movements, and muscle activity (spatially/temporally consistent artifacts) without discarding data points.

#### **Algorithm options:** 

* Infomax ICA (Makeig et al., 1996\) — most widely used; FastICA (Hyvärinen & Oja, 2000);   
* RELICA (Artoni et al., 2014);   
* AMICA (Palmer et al., 2010; Klug et al., 2024). 

Algorithm choice depends on your data's artifact profile (Delorme et al., 2012; Klug et al., 2024).

#### Requirements for good ICA decomposition:

* Use ≥64 channels where possible for adequate spatial resolution (Cohen, 2014; Klug & Gramann, 2020, *Eur J Neurosci*).  
* Sufficient training data: at least (channels² × 20\) data points for \<64 channels, or (channels² × 30\) for ≥64 channels (Makeig & Onton, 2011).  
* Apply a high-pass filter (\~1–2 Hz) beforehand to remove slow drifts that degrade decomposition quality; you can later apply ICA weights to a less-filtered copy of the data to avoid distorting ERP timing (Debener et al., 2010; Klug & Gramann, 2020; Winkler et al., 2015).  
* Run ICA **before** interpolation or average referencing to avoid rank-deficiency (reduced effective number of independent channels), or reduce the number of extracted components accordingly.

#### **Component classification:** 

Visually inspect the topography, time course, and spectrum of each component. Automated classifiers such as **ICLabel** (Pion-Tonachini et al., 2019, *NeuroImage*) assign each independent component a probability across seven classes \- Brain, Muscle, Eye, Heart, Line Noise, Channel Noise, Other \- and are available in EEGLAB (MATLAB) and via mne-icalabel (Python).

*ICLabel expects extended-Infomax ICA on average-referenced data filtered \~1–100 Hz.*

**Avoid over-correction:** only remove components confidently identified as artifact; removing components containing genuine brain signal degrades data quality.

### Step 7: Regression-Based Ocular Correction (Alternative/Supplement to ICA)

The **Gratton & Coles (1983)** algorithm regresses out blink/eye-movement contributions using EOG channels — computationally cheap but requires good EOG recordings (Jiang et al., 2019, *Sensors*, review of EEG artifact removal methods).

### Step 8: Rejection of Non-Stereotyped Artifacts

For sporadic events (electrode pops, gross movement, transient muscle tension) that don't have a consistent spatial signature suitable for ICA correction:

*  Detect via automatic thresholding (amplitude, kurtosis, joint probability) or manual visual marking. No universal threshold exists \- tune to your dataset and apply consistently across subjects, adjusting only when a dataset is clearly noisier.  
* Reject affected trials/channels/segments, or in extreme cases, the entire session.  
* **Best practice:** combine correction (ICA/regression) with rejection rather than relying on either alone, especially for mobile/motor paradigms (Zhang et al., 2024; Gorjan et al., 2022\)

### Step 9: Segmentation (Epoching)

* Define pre- and post-event windows based on your paradigm and expected neural latency.  
* For **time-frequency/wavelet analysis**, add safety margins to both segment edges to avoid edge effects and smearing (Herrmann et al., 2014, *Brain Topography*; Roach & Mathalon, 2008).  
* For **FFT/Welch spectral analysis**, ensure segment length covers at least one cycle of your lowest frequency of interest; use power-of-two lengths for computational efficiency.  
* For **motor imagery/BCI classification**, epoch relative to cue onset with sufficient pre-cue baseline for baseline correction and post-cue window covering the full imagined-movement period .

## 4\. Automated Preprocessing Pipelines (Optional, for Standardisation)

If you want a validated, standardised pipeline rather than assembling steps manually:

| Pipeline | Platform | Notes |
| :---: | ----- | ----- |
| **PREP** | EEGLAB/MATLAB | Standardised preprocessing for large-scale EEG (line noise removal, robust referencing, bad channel detection) (Bigdely-Shamlo et al., 2015\) |
| **HAPPE** | EEGLAB/MATLAB | Designed for developmental/high-artifact data (Gabard-Durnam et al., 2018; updated Monachino et al., 2022\) |
| **ADJUST** | EEGLAB/MATLAB | Automated IC artifact detection (Mognon et al., 2011\) |
| **RELAX** | EEGLAB/MATLAB | Automated cleaning pipeline (Bailey et al., 2023\) |
| **DISCOVER-EEG** | EEGLAB/MATLAB | Fully automated pipeline for biomarker discovery (Gil Ávila et al., 2023, *Scientific Data*) |
| **ICLabel / mne-icalabel** | EEGLAB or MNE-Python | Automated IC classification (Pion-Tonachini et al., 2019\) |

 

If you are proficient with MNE-Python and EEGLAB, mne-icalabel \+ a custom MNE pipeline, or EEGLAB with ICLabel/ADJUST, are both suitable starting points; you can build a hybrid pipeline and script it for batch/version-controlled processing.

## 5\. Special Consideration for Motor Imagery / BCI Pipelines

* A comparative study (Gao et al., 2025, *\[pipeline effects on motor imagery classification\]*) found that **baseline correction and band-pass filtering** consistently provided the most beneficial preprocessing effects for motor-imagery classification accuracy, while more complex steps (ICA, surface Laplacian) had more variable benefit depending on the dataset \- worth validating empirically on your own data rather than assuming a heavier pipeline is always better.  
* Surface Laplacian/Current Source Density (CSD) is commonly used as a spatial filter prior to connectivity or sensorimotor rhythm analyses to reduce volume conduction effects.  
* Movement and EMG contamination are more prominent in active motor tasks; apply the movement-artefact-specific guidance in Gorjan et al. (2022) and consider dedicated EMG channels for muscle-artefact regression.

## 6\. Documentation and Reporting Checklist (for Publication/Reproducibility)

* Acquisition system (manufacturer/model), electrode count, montage, electrode type (active/passive, material).  
* Sampling rate, analogue filter bandwidth, and any online notch filtering.  
* Reference and ground electrode locations (both online and any offline re-referencing scheme).  
* All offline filter parameters (type, order, cutoff, causal vs. zero-phase).  
* Bad channel count/criteria and interpolation method.  
* ICA algorithm, number of components removed, and classification method/criteria (manual or ICLabel-based).  
* Artifact rejection thresholds/criteria and percentage of data/trials rejected.  
* Epoching window and baseline correction window.  
* Software and version numbers for every processing step (supports exact reproducibility).

**References**

1. Bigdely-Shamlo, N., Mullen, T., Kothe, C., Su, K. M., & Robbins, K. A. (2015). The PREP pipeline: Standardized preprocessing for large-scale EEG analysis. *Frontiers in Neuroinformatics*, 9, 16\.  
2. Cohen, M. X. (2014). *Analyzing Neural Time Series Data: Theory and Practice*. MIT Press.  
3. de Cheveigné, A., & Nelken, I. (2019). Filters: When, why, and how (not) to use them. *Neuron*, 102(2), 280–293.  
4. de Cheveigné, A. (2023). Comment on "EEG is better left alone." *bioRxiv*.  
5. Debener, S., Thorne, J., Schneider, T. R., & Viola, F. C. (2010). Using ICA for the analysis of multi-channel EEG data. In *Simultaneous EEG and fMRI*. Oxford University Press.  
6. Delorme, A. (2023). EEG is better left alone. *Scientific Reports*, 13(1), 2372\.  
7. Delorme, A., & Makeig, S. (2004). EEGLAB: An open source toolbox for analysis of single-trial EEG dynamics. *Journal of Neuroscience Methods*, 134(1), 9–21.  
8. Delorme, A., Palmer, J., Onton, J., Oostenveld, R., & Makeig, S. (2012). Independent EEG sources are dipolar. *PLoS ONE*, 7(2), e30135.  
9. Gabard-Durnam, L. J., Mendez Leal, A. S., Wilkinson, C. L., & Levin, A. R. (2018). The Harvard Automated Processing Pipeline for EEG (HAPPE). *Frontiers in Neuroscience*, 12, 97\.  
10. Gao, X. et al. (2025). Effects of different preprocessing pipelines on motor imagery classification. *PubMed* \[PMID:40031268\].  
11. Gil Ávila, C., Bott, F. S., Tiemann, L., et al. (2023). DISCOVER-EEG: An open, fully automated EEG pipeline for biomarker discovery in clinical neuroscience. *Scientific Data*, 10(1), 613\.  
12. Gorjan, D., Gramann, K., De Pauw, K., & Marusic, U. (2022). Removal of movement-induced EEG artifacts: Current state of the art and guidelines. *Journal of Neural Engineering*, 19(1), 011004\.  
13. Gratton, G., Coles, M. G. H., & Donchin, E. (1983). A new method for off-line removal of ocular artifact. *Electroencephalography and Clinical Neurophysiology*, 55(4), 468–484.  
14. Gwin, J. T., Gramann, K., Makeig, S., & Ferris, D. P. (2010). Removal of movement artifact from high-density EEG recorded during walking and running. *Journal of Neurophysiology*, 103(6), 3526–3534.  
15. Hu, S., Lai, Y., Valdes-Sosa, P. A., Bringas-Vega, M. L., & Yao, D. (2018). How do reference montage and electrodes setup affect the measured scalp EEG potentials? *Journal of Neural Engineering*, 15(2), 026013\.  
16. Hyvärinen, A., & Oja, E. (2000). Independent component analysis: Algorithms and applications. *Neural Networks*, 13(4), 411–430.  
17. Jiang, X., Bian, G.-B., & Tian, Z. (2019). Removal of artifacts from EEG signals: A review. *Sensors*, 19(5), 987\.  
18. Kappenman, E. S., & Luck, S. J. (2010). The effects of electrode impedance on data quality and statistical significance in ERP recordings. *Psychophysiology*, 47(5), 888–904.  
19. Keil, A., Debener, S., Gratton, G., et al. (2014). Committee report: Publication guidelines and recommendations for studies using EEG and MEG. *Psychophysiology*, 51(1), 1–21.  
20. Klug, M., & Gramann, K. (2020). Identifying key factors for improving ICA-based decomposition of EEG data in mobile and stationary experiments. *European Journal of Neuroscience*.  
21.  Klug, M., Berg, T., & Gramann, K. (2024). Optimizing EEG ICA decomposition with data cleaning in stationary and mobile experiments. *Scientific Reports*, 14(1), 14119\.  
22. Larsen, B. A., & Versace, F. (2024). EEG might be better left alone, but ERPs must be attended to. *International Journal of Psychophysiology*, 205, 112441\.  
23.  Low, Y. F., & Martinez-Cancino, R. (2026). EEG Preprocessing and Artifact Handling. In T. Warbrick (Ed.), *The EEG Handbook: From Principles to Practice* (Ch. 17). Springer Nature Switzerland.  
24. Kadlec, D., et al. (2026). Getting Clean Data: Artifacts and How to Prevent Them. In T. Warbrick (Ed.), *The EEG Handbook* (Ch. 15). Springer Nature Switzerland.  
25. Makeig, S., & Onton, J. (2011). ICA of EEG. In *Oxford Handbook of Event-Related Potential Components*.  
26.  Mognon, A., Jovicich, J., Bruzzone, L., & Buiatti, M. (2011). ADJUST: An automatic EEG artifact detector based on the joint use of spatial and temporal features. *Psychophysiology*, 48(2), 229–240.  
27. Palmer, J., Kreutz-Delgado, K., & Makeig, S. (2010). AMICA: An adaptive mixture of independent component analyzers with shared components.  
28. Pernet, C., Garrido, M. I., Gramfort, A., et al. (2020). Issues and recommendations from the OHBM COBIDAS MEEG committee for reproducible EEG and MEG research. *Nature Neuroscience*, 23, 1473–1483.  
29.  Pion-Tonachini, L., Kreutz-Delgado, K., & Makeig, S. (2019). ICLabel: An automated electroencephalographic independent component classifier, dataset, and website. *NeuroImage*, 198, 181–197.  
30. Tsuchimoto, S., Shibusawa, S., Iwama, S., et al. (2021). Use of common average reference and large-Laplacian spatial filters enhances EEG SNR in intrinsic sensorimotor activity. *Journal of Neuroscience Methods*, 353, 109089\.  
31.  Widmann, A., Schröger, E., & Maess, B. (2015). Digital filter design for electrophysiological data—a practical approach. *Journal of Neuroscience Methods*, 250, 34–46.  
32. Winkler, I., Debener, S., Müller, K.-R., & Tangermann, M. (2015). On the influence of high-pass filtering on ICA-based artifact reduction in EEG-ERP.

