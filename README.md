# CINN_BIDS

Organising raw MRI DICOM data into **BIDS** format at CINN, ready for downstream analysis (fMRIPrep, MRIQC, SPM, FSL, …).

This guide uses [BIDScoin](https://bidscoin.readthedocs.io/en/latest/), which converts DICOMs to NIfTI (via dcm2niix) and names/arranges everything according to the [BIDS specification](https://bids-specification.readthedocs.io/). You do **not** need to clone this repo — just follow the steps below.

---

## Overview

```
Raw DICOMs ──► sourcedata/ (sorted by subject/session) ──► bidsmapper ──► bidseditor ──► bidscoiner ──► BIDS dataset
                                                          (scan data)    (check/fix)    (convert)
```

| Step | Where to run | Needs a GUI? |
|---|---|---|
| Copy & organise raw data | NoMachine terminal | No |
| Quick check with dcm2niix (optional) | NoMachine terminal | No |
| `bidsmapper` / `bidseditor` | **NoMachine terminal** | **Yes** |
| `bidscoiner` | NoMachine terminal | No |
| Analysis (fMRIPrep, SPM, FSL …) | RACC2 | No |

> **Tip:** All of the BIDS conversion can be done from the NoMachine terminal — you don't need to log in to RACC2 until you start your analysis. The BIDScoin editor needs a GUI, which is only supported on NoMachine.

---

## 1. Load the modules

In a NoMachine terminal:

```bash
module load neurodesk-modules
module load bidscoin
```

`module avail` shows all the other modules you can load.

Check that BIDScoin and its plugins are working:

```bash
bidscoin -t
```

---

## 2. Copy your raw data to your working directory

Never work on the original copy of your scans. Copy them to your own working space first:

```bash
cp -r <source_directory> <destination_directory>
```

---

## 3. Organise the DICOMs into a `sourcedata` folder

BIDScoin expects **one folder per subject**, optionally with **one folder per session** inside it, and the DICOMs underneath:

```
myproject/
├── sourcedata/                  # raw DICOMs (input)
│   ├── sub-001/
│   │   ├── ses-01/
│   │   │   ├── 001-localizer/     # one folder per scan series (ideal)
│   │   │   ├── 002-T1w_MPRAGE/
│   │   │   ├── 003-rest_fMRI/
│   │   │   └── ...
│   │   └── ses-02/
│   └── sub-002/
│       └── ses-01/
└── bids/                        # BIDS output (created by BIDScoin)
```

Notes:

- The session level is optional — leave it out if you only have one session per participant.
- If your DICOMs are all in one flat folder per subject (no series sub-folders), that's fine — BIDScoin sorts them automatically.
- If your folders use a different prefix (e.g. `P018` instead of `sub-018`), tell BIDScoin with `-n P` (subject prefix) and `-m <prefix>` (session prefix) in the commands below.
- Use participant **codes**, not names, in folder names.

---

## 4. (Optional) Quick check with dcm2niix

Before running BIDScoin, you can convert one session by hand to check the scans look right (BIDScoin runs dcm2niix itself in step 6, so this is only a sanity check):

```bash
mkdir -p ~/dcm2niix_test
dcm2niix -o ~/dcm2niix_test myproject/sourcedata/sub-001/ses-01
```

Open the NIfTI files in FSLeyes to check them:

```bash
module load fsl
fsleyes ~/dcm2niix_test/*.nii.gz
```

---

## 5. Map your scans: `bidsmapper` + `bidseditor`

The bidsmapper scans your source data and works out which BIDS name each scan series should get. It then opens the **bidseditor** so you can check and fix the mapping.

```bash
cd myproject
bidsmapper sourcedata bids
```

In the bidseditor:

1. Go through each scan series and check its **datatype** (`anat`, `func`, `fmap`, `dwi`, …) and **suffix** (`T1w`, `bold`, `epi`, …).
2. For functional runs, set the **task** label (e.g. `task-rest`, `task-reading`).
3. Set series you don't want (localizers, scouts, derived images) to **exclude**.
4. Click **Save** when done.

The mapping is saved as `bids/code/bidscoin/bidsmap.yaml`. You only need to do this once per study (as long as the scan protocol doesn't change).

To re-open the editor later:

```bash
bidseditor bids
```

---

## 6. Convert: `bidscoiner`

Convert all subjects using the saved bidsmap:

```bash
bidscoiner sourcedata bids
```

Useful options:

| Option | What it does |
|---|---|
| `-p 001 002` | Only convert these subjects (the `sub-` prefix can be left out) |
| `-f` | Re-convert subjects that already exist in `bids/` |

**Adding new participants later:** copy their DICOMs into `sourcedata/` and re-run `bidscoiner sourcedata bids`. Subjects already in `bids/` are skipped. If the scan protocol has changed, re-run `bidsmapper` first.

---

## 7. Check the result

Your `bids/` folder should now look something like this:

```
bids/
├── dataset_description.json
├── participants.tsv
├── participants.json
├── code/bidscoin/          # bidsmap and logs
└── sub-001/
    └── ses-01/
        ├── anat/
        │   ├── sub-001_ses-01_T1w.nii.gz
        │   └── sub-001_ses-01_T1w.json
        └── func/
            ├── sub-001_ses-01_task-rest_bold.nii.gz
            └── sub-001_ses-01_task-rest_bold.json
```

Then:

- **Edit `dataset_description.json`** — add your study name, authors, etc.
- **Check the logs** in `bids/code/bidscoin/` for warnings or errors.
- **Validate** the dataset with the [BIDS Validator](https://bids-standard.github.io/bids-validator/). It runs in your browser and does not upload your data.

---

## 8. Next steps

Once your data are in BIDS, you can run your analysis on RACC2. Log in from the NoMachine terminal:

```bash
ssh -X username@racc.rdg.ac.uk
```

Other ways to log in: [RACC2 – Login and Interactive Computing](https://research.reading.ac.uk/act/knowledgebase/racc2-login-and-interactive-computing/)

Other CINN guides:

- [CINN_SPM](https://gitlab.act.reading.ac.uk/op936728/CINN_SPM) — SPM batch template (`module load spm26`)
- [CINN_fsl_sub](https://gitlab.act.reading.ac.uk/op936728/CINN_fsl_sub) — running FSL jobs on RACC2 with `fsl_sub` (the RACC2 section is useful for any analysis)

---

## Quick reference

```bash
# NoMachine terminal
module load neurodesk-modules
module load bidscoin

cp -r <source_directory> myproject/sourcedata/
cd myproject
bidsmapper sourcedata bids     # map scans + check in the GUI
bidscoiner sourcedata bids     # convert to BIDS
```

---

## Further reading

- [BIDScoin workflow](https://bidscoin.readthedocs.io/en/latest/workflow.html)
- [BIDScoin: preparing your data](https://bidscoin.readthedocs.io/en/latest/preparation.html)
- [BIDS specification](https://bids-specification.readthedocs.io/)

Questions or problems? Contact Simon Yuan (CINN).
