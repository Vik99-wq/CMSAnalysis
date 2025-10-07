# Histograms Subsystem

The **Histograms** module in the `CMSAnalysis` framework is responsible for building, filling, and managing histograms that summarize physics observables from simulated (GenSim) and reconstructed (Reco) events.  
It serves as the primary visualization and validation layer of the analysis pipeline, transforming selected event data into meaningful statistical distributions.

---

## 📘 Overview

In CMS-style analyses, histograms form the bridge between **event-level data** and **physics interpretations**.  
They allow researchers to visualize distributions of observables (mass, transverse momentum, pseudorapidity, etc.), compare Monte Carlo (MC) simulations with real data, and extract statistical insights.

Within this project:
- The **`HistogramManager`** class provides an interface for defining and filling histograms.
- **Event dumps** (`GenSimEventDump.txt`, `RecoEventDump.txt`) provide validation data.
- Output histograms can be saved into ROOT files for plotting and statistical fitting.

---

## 🧩 Directory Structure

```
Histograms/
├── BuildFile.xml               # CMSSW build configuration for this package
├── GenSimEventDump.txt         # Example generator-level (truth) data
├── RecoEventDump.txt           # Example reconstructed data for comparison
├── interface/
│   └── HistogramManager.h      # Header for histogram control class
└── src/
    └── HistogramManager.cc     # Implementation of histogram creation and filling
```

---

## 🎯 Goals

The histogram subsystem is designed to:

1. **Aggregate event observables** into statistical distributions.
2. **Provide modular access** to histogram creation and filling routines.
3. **Ensure consistency** between generator-level and reconstructed data.
4. **Integrate with CMS analysis tools** (e.g., ROOT and Combine).
5. **Simplify validation** of event selection and filtering steps.

---

## ⚙️ How It Works

### 1. Initialization

Each analysis module creates its own instance of the `HistogramManager`.  
This object maintains a registry of histograms and handles binning, naming, and output management.

```cpp
#include "CMSAnalysis/Histograms/interface/HistogramManager.h"

HistogramManager histManager("output_histograms.root");
```

At initialization, the manager loads (or creates) histograms with standard CMS naming conventions:
- `process_variable`
- `process_variable_Up` / `process_variable_Down` (for systematic variations)
- `data_obs` for observed data

---

### 2. Defining Histograms

Histograms can be declared dynamically, specifying the name, binning, and axis labels.

```cpp
histManager.Create1D("electron_pt", 50, 0, 200, "Electron pT [GeV]");
histManager.Create1D("dimuon_mass", 80, 60, 120, "Dimuon invariant mass [GeV]");
histManager.Create2D("eta_vs_phi", 60, -3, 3, 64, -3.2, 3.2, "η vs ϕ");
```

The `HistogramManager` internally uses ROOT’s `TH1F` and `TH2F` classes to store these distributions.

---

### 3. Filling Histograms

As the event loop runs (after selection by modules in `/Filters`), physics quantities are filled into the corresponding histograms:

```cpp
for (const auto& electron : electrons) {
    histManager.Fill("electron_pt", electron.pt());
    histManager.Fill("eta_vs_phi", electron.eta(), electron.phi());
}

double dimuonMass = (muon1.p4() + muon2.p4()).M();
histManager.Fill("dimuon_mass", dimuonMass);
```

Each `Fill()` call automatically updates the histogram counts and sum of weights, maintaining compatibility with ROOT statistical operations.

---

### 4. Output and Storage

After the analysis completes, all histograms are written to a ROOT file, usually stored in `Output/`:

```cpp
histManager.Write();
```

The resulting file (e.g., `histograms.root`) can be opened in ROOT or Python for plotting:

```bash
root -l Output/histograms.root
```

---

## 📊 GenSim vs. Reco Validation

The provided example dumps — `GenSimEventDump.txt` and `RecoEventDump.txt` — allow developers to verify the correctness of the histogram logic by comparing truth-level and reconstructed distributions.

For instance:
- Check that reconstructed invariant masses match generator-level distributions.
- Ensure object multiplicities (e.g., jet counts) align across both datasets.

This comparison ensures accurate reconstruction algorithms and event selection behavior.

---

## 🔬 Example Workflow

A typical end-to-end usage scenario within the CMSAnalysis framework:

1. **Event Selection**
   - The `Filters/HiggsSelector` module selects Higgs candidate events.

2. **Histogram Filling**
   - The selected events are passed to `HistogramManager` for filling histograms such as:
     - `HiggsMass`
     - `LeadMuonPt`
     - `JetMultiplicity`

3. **ROOT Output**
   - Histograms are written to a ROOT file under `Output/`.

4. **Plotting**
   - Histograms are visualized using ROOT or a Python plotting tool:
     ```bash
     root -l -q 'plotHistograms.C("Output/histograms.root")'
     ```

5. **Statistical Analysis**
   - The resulting histograms serve as inputs for limit calculations in the `Statistics/` module.

---

## 🧠 Design Philosophy

The histogram subsystem is built around **modularity**, **reusability**, and **integration**:

- **Modularity:**  
  Each physics channel or analysis stage can define its own histograms without modifying core code.
  
- **Reusability:**  
  The same histogram definitions can be reused across analyses by sharing `HistogramManager` configurations.

- **Integration:**  
  Designed to work seamlessly with CMS software stack (`CMSSW`), ensuring compatibility with official data formats and workflows.

---

## 🧰 Dependencies

- [ROOT](https://root.cern/) ≥ 6.0  
- CMSSW build environment (via `BuildFile.xml`)  
- C++17 or later

Optional for plotting and statistical post-processing:
- `Python 3.x`
- `matplotlib`, `uproot`

---

## 📂 Output Example

Example histograms typically produced:

| Histogram Name   | Description                        | Units  |
|------------------|------------------------------------|--------|
| `electron_pt`    | Transverse momentum of electrons    | GeV    |
| `dimuon_mass`    | Invariant mass of muon pairs        | GeV    |
| `nJets`          | Jet multiplicity per event          | count  |
| `eta_vs_phi`     | Angular distribution of particles   | none   |

---

## 🧩 Future Extensions

- Automatic systematic variation handling (`*_Up`, `*_Down`)  
- YAML-based histogram configuration files  
- Python wrapper for interactive plotting  
- Integration with Combine datacard generation

---

## 🧑‍💻 Author & Maintenance

Developed and maintained by CMS student collaborators as part of the **CMS Analysis Framework** project.  
Contributions are welcome through pull requests and issues.

---

## 📜 License

This module is distributed under the **MIT License** (or project license if different).  
See the root `LICENSE` file for full terms.

---
