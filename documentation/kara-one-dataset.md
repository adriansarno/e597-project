## Kara One Dataset: Format and Organization

### Overview

The Kara One dataset is a public, multimodal imagined speech dataset collected at the University of Toronto. It includes synchronized recordings of EEG, audio, and auxiliary signals during tasks involving perceived speech, imagined speech, and overt speech.

For this project, only the EEG data and imagined-word trials are used.

Each participant’s data is distributed as a compressed archive (.tar.bz2) containing continuous EEG recordings and associated metadata files that define trial boundaries and stimulus labels.

---

### Directory Structure (per participant)

After extraction (e.g., P02.tar.bz2), the directory typically contains:

```graphql
P02/
 ├── *.cnt              # Continuous EEG recording (Neuroscan format)
 ├── epoch_inds.mat     # Trial boundary indices
 ├── ID.txt or ID_p.txt # Prompt / stimulus order
 ├── audio/             # (optional) audio recordings
 ├── video/             # (optional) face video
 └── README / metadata
```



Only the .cnt, .mat, and ID*.txt files are required for EEG decoding.

---

### EEG Data (.cnt file)

- Format: Neuroscan CNT (binary)
- Accessed via: `mne.io.read_raw_cnt`

Content:
- Continuous EEG signal for the entire session
- Multiple channels (high-density montage)
- Fixed sampling rate (typically ~1 kHz)


In this project, a subset of 8 channels is selected to emulate an OpenBCI Cyton montage:

Fp1, Fp2, C3, C4, P7, P8, O1, O2


---

### Trial Boundaries (epoch_inds.mat)

This MATLAB file defines where each trial begins and ends within the continuous EEG recording.

- Loaded with: `scipy.io.loadmat`
- Contains: one or more arrays of integer indices

Typical structure:

(n_trials × ≥2 columns)


Where:
- Column 0 = trial start sample index
- Last column = trial end sample index

Because the exact variable name can differ across releases, a robust parser is required to:
- identify arrays with appropriate shape
- extract (start, end) sample indices

---

### Prompt / Label File (ID.txt or ID_p.txt)

This text file defines the stimulus sequence corresponding to each trial.

- One line per trial
- Entries may include:
  - imagined words (pat, pot, knew, gnaw)
  - phonemes or non-word prompts
- Order matches the trial order in epoch_inds.mat

In this project, only trials whose labels are in the closed vocabulary:

{pat, pot, knew, gnaw}


are retained. Non-word or phoneme trials are discarded.

---

### Alignment of EEG and Labels

The dataset is aligned as follows:

- Continuous EEG provides the raw signal
- epoch_inds.mat defines trial boundaries in sample indices
- ID*.txt provides the stimulus label for each trial

Trial i uses:
- EEG samples [start_i : end_i)
- Label ID[i]

For feasibility and robustness, a fixed-length window (e.g., 5 seconds) is extracted from the center of each trial to approximate the imagined-speech period. This avoids reliance on more complex state annotations.

---

### Final Analysis Dataset (after preprocessing)

After filtering and feature extraction, each trial becomes:

X[i] ∈ ℝ^(8 × T) → features ∈ ℝ^24
y[i] ∈ {pat, pot, knew, gnaw}  
  
Where:
- 8 EEG channels
- T = samples in the fixed window
- Features = log-bandpower in standard EEG frequency bands

---


