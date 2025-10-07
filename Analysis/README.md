# Analysis Subsystem

The **Analysis** package provides the orchestration layer of the CMSAnalysis framework.  
It connects data ingestion, filtering, histogramming, and statistical evaluation into a single reproducible workflow.

This subsystem defines the primary logic for **running full analyses**, **configuring modules**, and **managing execution** over event datasets.  
It serves as both a **user interface** (for running analyses and generating outputs) and a **developer interface** (for defining new workflows, modules, or physics analyses).

---

## Overview

CMSAnalysis is organized around a modular pipeline:

```
DataCollection  →  Filters  →  Analysis  →  Histograms  →  Statistics
```

The **Analysis** subdirectory contains the code responsible for:

1. Constructing the analysis chain (by instantiating modules and their dependencies).  
2. Managing configuration and runtime parameters.  
3. Running event loops and invoking module processing hooks.  
4. Writing final outputs such as histograms, text summaries, and logs.

Each analysis is represented by a combination of:
- **Configuration classes** (`AnalyzerOptions`, `AnalyzerParams`)
- **Execution drivers** (`Analyzer`)
- **Helper macros and scripts** under `bin/` for running, plotting, and combining results.

---

## Directory Structure

```
Analysis/
├── BuildFile.xml            # CMSSW build configuration
├── README.md                # This file
├── bin/                     # Executable drivers and ROOT macros
│   ├── AutoSuperImpose.C
│   ├── CombineCanvases.C
│   ├── HiggsEstimation.C
│   └── <additional analysis scripts>
├── interface/               # Public headers (core analysis classes)
│   ├── Analyzer.hh
│   ├── AnalyzerOptions.hh
│   └── AnalyzerParams.hh
└── src/                     # Implementations of analysis logic
    ├── Analyzer.cc
    ├── AnalyzerOptions.cc
    └── AnalyzerParams.cc
```

---

## Core Classes and Architecture

### Analyzer

**Purpose:** Central driver of the analysis.  

This class coordinates:
- Module initialization (`initialize()` methods)
- Event processing loop
- Execution of filtering, reconstruction, and histogram modules
- Finalization and output writing

A typical `Analyzer` object constructs a list of modules (e.g., `FilterModule`, `HistogramModule`), sets their dependencies, and calls `processEvent()` for each event.

**Key responsibilities:**
- Control the analysis lifecycle: `initialize()`, `processEvent()`, and `finalize()`
- Manage inter-module data flow (via `EventInput` references)
- Handle timing, bookkeeping, and logging
- Write text summaries and ROOT histograms after completion

**Developer usage:**
```cpp
#include "CMSAnalysis/Analysis/interface/Analyzer.hh"

int main() {
    Analyzer analyzer;
    analyzer.initialize();
    analyzer.run();     // Executes full event loop
    analyzer.finalize();
}
```

---

### AnalyzerOptions

**Purpose:** Defines global and per-module configuration parameters.

Holds user-configurable options such as:
- Input files or datasets
- Trigger paths and selection flags
- Output directory paths
- Boolean toggles for optional features (e.g., smearing, debugging, verbosity)

Options are typically passed to `Analyzer` upon construction, allowing the same code to run different analyses without recompilation.

Example:
```cpp
AnalyzerOptions options;
options.setInputFile("input.root");
options.setOutputDir("Output/HiggsStudy/");
options.enableSmearing(true);
Analyzer analyzer(options);
```

---

### AnalyzerParams

**Purpose:** Encapsulates numeric and physics-specific parameters.

Used to centralize constants that influence selection, reconstruction, or weighting, such as:
- Mass cuts
- pT thresholds
- Cross-sections
- Luminosity scaling factors
- Systematic variations

By separating parameters from logic, this class supports version-controlled parameter sets and quick retuning of analyses.

Example:
```cpp
AnalyzerParams params;
params.setCut("dimuon_mass_min", 60.0);
params.setCut("dimuon_mass_max", 120.0);
```

---

## Developer Guide

### Extending the Framework

To implement a new analysis:
1. **Create a configuration header** under `interface/` with your analysis-specific options.  
2. **Write a corresponding `.cc` file** under `src/` implementing:
   - Initialization of modules (`Filters`, `Histograms`, etc.)
   - Event loop customization
   - Output formatting (histograms, plots, summaries)
3. **Add a driver script** under `bin/` that instantiates your analyzer and runs it.

Example pattern:
```cpp
#include "CMSAnalysis/Analysis/interface/Analyzer.hh"
#include "CMSAnalysis/Filters/interface/HiggsSelector.hh"
#include "CMSAnalysis/Histograms/interface/DeltaRHist.hh"

int main() {
    AnalyzerOptions opts;
    Analyzer analyzer(opts);

    // Example: Add a Higgs event selector and a histogram
    auto higgsFilter = std::make_shared<HiggsSelector>();
    auto deltaRHist  = std::make_shared<DeltaRHist>(/* ... */);

    analyzer.addModule(higgsFilter);
    analyzer.addModule(deltaRHist);

    analyzer.initialize();
    analyzer.run();
    analyzer.finalize();

    return 0;
}
```

---

## User Guide

### Running a Standard Analysis

Users can run predefined analyses using scripts under `bin/`:

```bash
# Example: combine multiple canvases into a single comparison plot
root -l -q 'CombineCanvases.C("Higgs","ZJets")'

# Example: estimate signal contribution from control regions
root -l -q 'HiggsEstimation.C'
```

Most macros take arguments for dataset names, histogram ROOT files, and output directories.

---

### Typical Workflow

1. **Prepare input files**
   - Simulated and/or data ROOT files should be accessible via the `DataCollection` subsystem.

2. **Configure the analyzer**
   - Modify `AnalyzerOptions` or parameters in your macro to point to the right input files and settings.

3. **Run the analysis**
   ```bash
   cmsRun Analysis/bin/RunHiggsAnalysis_cfg.py
   ```
   or, if using standalone ROOT macros:
   ```bash
   root -l -q 'Analysis/bin/RunHiggsAnalysis.C'
   ```

4. **Inspect outputs**
   - ROOT histograms written to `Output/`
   - Text summaries in terminal or log files
   - Optional plots generated via `CombineCanvases.C` or `AutoSuperImpose.C`

---

## Interactions with Other Packages

| Package | Role in the Pipeline |
|----------|----------------------|
| **DataCollection** | Supplies `EventInput` objects that wrap event data. |
| **Filters** | Provide event selection modules to reduce background or define categories. |
| **Histograms** | Define, fill, and output physics histograms from event observables. |
| **Statistics** | Use histograms for limit setting, fits, and uncertainty estimation. |
| **Utility** | Provides shared classes like `HistogramPrototype`, `Filter`, and `ScaleFactor`. |

The `Analysis` layer binds these components into a coherent execution framework.

---

## Example: Custom Dilepton Analysis

A minimal workflow might include:
1. **Modules**
   - `DileptonSelector` to filter events with two leptons.
   - `InvariantMassHist` to record the invariant mass of the pair.

2. **Driver Script**
   ```cpp
   Analyzer analyzer;
   analyzer.addModule(std::make_shared<DileptonSelector>());
   analyzer.addModule(std::make_shared<InvariantMassHist>());
   analyzer.run();
   ```

3. **Output**
   ```
   Output/
   ├── histograms.root
   ├── logs/
   └── summary.txt
   ```

---

## Design Philosophy

- **Reusability:** Modules are self-contained; any analysis can reuse existing filters or histograms.
- **Transparency:** Configuration and parameters are stored explicitly in source or text files.
- **CMSSW Integration:** Designed to compile and run seamlessly within CMSSW using SCRAM.
- **Minimal coupling:** `Analyzer` coordinates modules through interfaces, not hardcoded dependencies.

---

## Dependencies

- **ROOT** ≥ 6.0  
- **CMSSW** runtime environment  
- C++17 compiler  
- Optional: Python 3.x for plotting wrappers

---

## License

This component follows the project-wide license defined in the root of the repository.

---

## Authors and Maintenance

Developed as part of the **CMSAnalysis Framework**.  
Maintainers and contributors: CMS student collaborators and instructors affiliated with the project.

---
