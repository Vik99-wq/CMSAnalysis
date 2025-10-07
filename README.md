# CMSAnalysis

CMSAnalysis is a modular analysis framework for high‑energy physics built around the CMS software stack (CMSSW). It provides a clean, reproducible path from raw or skimmed EDM/ROOT inputs to physics results via a sequence of well‑defined stages:

1) Data ingestion and event access  
2) Event selection and reconstruction (filtering, object building, ML inference)  
3) Histogram creation for validation and shape analysis  
4) Statistical interpretation (datacards, fits, post‑fit plots)

The codebase follows a CMSSW‑style package layout with `BuildFile.xml` files, public headers under `interface/`, and implementations under `src/`.

---

## Quick start

The repository is intended to be used inside a CMSSW area.

```bash
# Example: set up a CMSSW release (choose the release appropriate for your site/inputs)
cmsrel CMSSW_12_4_0
cd CMSSW_12_4_0/src
cmsenv

# Clone and build
git clone <your-fork-or-remote> CMSAnalysis
scram b -j8

# Run your analysis or module (examples vary by site/workflow)
# See the directory-specific sections below for typical entrypoints.
```
If your site does not use CMSSW for the full pipeline, you can still use the histogram/utility libraries with ROOT by linking against the built libraries; however, the standard and most tested path is the CMSSW build shown above.

---

## High‑level architecture

At a high level the framework organizes the event processing pipeline as follows:

```
EDM/ROOT files  →  DataCollection            →  Modules Pipeline                      →  Histograms            →  Statistics/Plots
                     (Event loaders,            (FilterModule, Reconstruction,           (1D/2D templates,        (datacards, limits,
                      interfaces)                MLCalculator, custom Modules)            GenSim vs Reco)          post‑fit plots)
```

Key abstractions used throughout the repository:

- **Module** (in `Modules/`): base class that encapsulates a unit of work. Each module can declare dependencies, processes each event, and optionally finalizes at the end of the job.  
- **EventInput** (in `Modules/`): uniform read‑only interface to event content (objects, triggers, MET, generator info).  
- **FilterModule** (in `Modules/`): specialization for event selection; produces a string or a veto and maintains counters.  
- **HistogramPrototype** (in `Utility/`): abstract base for histogram definitions that knows how to make a TH1 and how to compute values and event weights.  
- **Filter** and **ScaleFactor** (in `Utility/`): small strategy objects plugged into `HistogramPrototype` to implement selection strings and event weighting consistently.

---

## Directory‑by‑directory guide

### Analysis
Top‑level control logic and utilities that tie modules together and provide end‑to‑end workflows. Typical contents include small programs or ROOT macros that:
- configure processes and inputs,
- run selections,
- invoke the histogram subsystem,
- combine canvases or superimpose distributions for comparisons.

`Analysis/bin` contains compiled helpers and macros such as `AutoSuperImpose.C`, `CombineCanvases.C`, and scripts for background estimates. The package has its own `BuildFile.xml` and public headers under `Analysis/interface/`.

### DataCollection
Implements data ingestion and event access. You will find the core abstractions for reading events and presenting them to modules:

- `Analyzer.hh`, `AnalyzerOptions.hh`, `AnalyzerParams.hh`: high‑level drivers for running configured pipelines.  
- `EventInterface.hh`, `RootEventInterface.hh`, `CmsswEventInterface.hh`: polymorphic event readers for ROOT trees or EDM/CMSSW formats.  
- `EventLoader.hh`, `RecoAODEventLoader.hh`: concrete loaders for specific CMSSW data tiers.  
- Trigger utilities (`TriggerArray.hh`, `TriggerNames` access via `EventInput`).  
- Process bookkeeping (`Process.hh`, `ProcessDictionary.hh`).  
- File parameter helpers (`SingleFileParams.hh`, `ListFileParams.hh`, `PickFileParams.hh`).

This package hides the details of file formats and exposes a single `EventInput` to all downstream modules.

### EventFiles
Reference event dumps for validation and regression tests. Example files include `GenSimEventDump.txt` and `RecoEventDump.txt`. These are useful when checking basic distributions or debugging object definitions early in a study.

### Filters
Physics selections used to define analysis regions and control samples. For example:
- `HiggsSelector.cc` to retain Higgs‑like topologies,
- `HEEPtest.txt` and related selections for electron ID studies.

The directory defines the filter modules’ headers in `Filters/interface/` and their implementations in `Filters/src/`. Filters typically integrate with `FilterModule` and provide simple pass/fail strings or counters consumed by histograms.

### Histograms
Implements the histogramming subsystem that converts selected events into distributions suitable for validation, plotting, and statistical fits.

Representative headers in `Histograms/interface/` include:
- Small concrete histogram classes, e.g. `DeltaRHist.hh`, `GammaHist.hh`, `DarkPhotonMassHist.hh`, `NLeptonsHist.hh`.
- Generic helpers such as `HistogramPrototype1DGeneral.hh`.
- Base prototypes that connect GenSim and Reco views, e.g. `GenSimRecoPrototype.hh` and `ResolutionPrototype.hh`.
- A consolidated header `Histograms.hh` that groups simple histogram types.

The implementations live in `Histograms/src/` with one `.cc` per histogram. See the dedicated `Histograms/README.md` for an in‑depth guide to goals, design, and usage.

### MCGeneration
Configuration and helpers for Monte Carlo event generation. Contains CMSSW plugin scaffolding under `plugins/` and job/test configurations under `python/` and `test/`. Use this package to produce private signal or background samples that follow the same conventions as the rest of the framework.

### Modules
Core framework classes and physics modules. Important interfaces include:

- `Module.hh`: base interface providing `addDependent`, per‑event `process`, and end‑of‑job `finalize` hooks, plus per‑module timing and parameter handling.  
- `EventInput.hh`: polymorphic accessor to per‑event objects (leptons, jets), generator info, trigger decisions, MET, and run/event IDs.  
- `FilterModule.hh`: base for analysis filters; it coordinates filter strings and counters and can be chained with other modules.  
- Reconstruction and analysis modules referenced by the histogram code, e.g. `LeptonJetReconstructionModule.hh` and `MLCalculator.hh`.

This package is the backbone that orchestrates the pipeline and exposes the common APIs used by Filters and Histograms.

### Plans
Development notes and planning stubs packaged as a CMSSW library. Not required at runtime; useful for tracking analysis ideas and design experiments alongside the code.

### Statistics
Tools for statistical inference and plotting of fit results. Notable scripts include:
- `LimitCalculation.py`, `RealRunLimitCalculation.py`, `RunEELimitCalculation.py`: datacard assembly and limit running helpers,
- `postFitPlot.py`, `makeLimitPlot.py`, `plot1DScan.py`, `diffNuisances.py`: plotting scripts for model scans and post‑fits,
- auxiliary text snippets such as `datacard_part1.txt` and example cards.

These tools assume that the histogram stage has produced standardized shapes and yields per process and category.

### Utility
Common infrastructure used throughout the framework:

- **Histogram prototypes:** `HistogramPrototype.hh`, `HistogramPrototype1D.hh` provide a uniform way to define histograms. A prototype knows how to instantiate a `TH1`, how to compute its values via `value()`, and how to construct an event weight via a list of `ScaleFactor` objects. The base also supports attaching `Filter` objects to skip fills when selection strings are empty.  
- **Filter and ScaleFactor strategies:** clean interfaces to plug in selection logic and event weights independent of the histogram definition.  
- **Physics objects and collections:** classes such as `Particle.hh`, `Lepton.hh`, `ParticleCollection.hh` and their concrete implementations.  
- Miscellaneous helpers for tables and lightweight output formatting.

The `Utility/src/HistogramPrototype.cc` implementation illustrates the event‑weight and filter‑application flow shared by all histograms.

### Output
Default location for generated artifacts (ROOT files, plots, and logs). Keep this directory clean in version control as outputs are typically large and site‑specific.

---

## Histogram subsystem at a glance

The histogram machinery is heavily reused across analyses and is worth calling out explicitly. The essential pieces are:

- **Definition:** a histogram is defined by a `HistogramPrototype` (or a 1D specialization). Derived classes implement `value()` which returns one or more numbers to be filled.  
- **Weights and selections:** the base class composes a list of `ScaleFactor` providers and `Filter` objects. An event is filled only if all filters return non‑empty tags; the weight is the product of scale factors. See `Utility/src/HistogramPrototype.cc`.  
- **Implementations:** concrete classes under `Histograms/src` perform common tasks (mass, pT, multiplicities, angular distances, resolutions). Many have GenSim and Reco variants via `GenSimRecoPrototype`.  
- **Output:** the histograms are written as ROOT `TH1` objects and can be consumed by both the plotting helpers in `Analysis/` and the limit machinery in `Statistics/`.

For a complete, developer‑oriented guide, including example code, see `Histograms/README.md` in this repository.

---

## Building and running

1) **Environment**  
Set up a CMSSW release compatible with your site and input files and run `cmsenv`.

2) **Build**  
From `CMSSW_X_Y_Z/src`, clone this repository and run:
```bash
scram b -j8
```

3) **Pick an entrypoint**  
- For histogram‑centric studies, build and call a small driver that wires together a loader (`DataCollection`), one or more filters (`Filters`), and a set of histogram prototypes (`Histograms`).  
- For plotting and quick visual checks, use the macros and helpers under `Analysis/bin`.  
- For limits or scans, export the shapes to a ROOT file and use the scripts in `Statistics/` to build datacards and run fits.

Because different analyses wire modules differently, the repository leaves the exact driver configuration to the user’s workflow. The headers in `DataCollection/interface/Analyzer.hh` and `Modules/interface/Module.hh` show the intended control flow for building such a driver.

---

## Conventions

- **CMSSW layout:** each package has `BuildFile.xml`, public headers under `interface/`, and implementation under `src/`.  
- **Style:** small single‑purpose classes are favored over monoliths; most specialized histograms live in their own `.cc` file for readability.  
- **Reproducibility:** selections, weights, trigger lists, and binning should be captured in version‑controlled configuration headers or small text files alongside the code (see examples under `Filters/` and `Histograms/`).

---

## Troubleshooting

- Linker errors for ROOT or CMSSW symbols usually indicate a missing `BuildFile.xml` dependency; compare with other packages’ `BuildFile.xml` files and add the missing `use` entries.  
- If histograms are empty, check that filters are returning non‑empty strings and that scale factors are finite. The `HistogramPrototype::shouldDraw()` and `eventWeight()` implementations are good places to instrument logs.  
- Use the `EventFiles/` dumps to validate that objects are present with reasonable kinematics before debugging higher‑level modules.

---

## Acknowledgments

The repository contains contributions from multiple student authors. Code comments and headers attribute original implementations where available (for example, in `Histograms/Histograms.hh`). The structure and interfaces borrow conventions from typical CMSSW analysis packages to ensure maintainability and portability across sites.
