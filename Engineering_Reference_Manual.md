# Engineering Reference Manual

## Executive summary

This document is the Engineering Reference Manual of the open-source Python application developed in the `LCADUBS` project folder. The software is designed to generate EnergyPlus models from GIS building footprints, to orchestrate external simulation and post-processing tools, and to support a discrete multi-objective optimization process in which energy performance and environmental impacts are evaluated together at building-cluster scale.

From an engineering point of view, the tool has four tightly connected responsibilities.

- It acts as a **geometry-to-simulation translator**, because it converts shapefile-based urban geometry and typological metadata into EnergyPlus-readable IDF content.
- It acts as a **workflow manager**, because it launches and supervises external freeware/open-source tools such as EnergyPlus and ReadVarsESO.
- It acts as a **mathematical model evaluator**, because it builds alternative intervention scenarios and computes objective functions on them.
- It acts as a **decision-support system**, because it exports scenario-level and Pareto-level outputs that can be directly used in technical reporting and retrofit prioritization.

The present manual documents the complete logic of the tool with emphasis on architecture, computational workflow, mathematical formulation, software interfaces, input/output contracts, validation logic, and known assumptions.

## 1. Purpose, scope, and intended audience

The purpose of this manual is to provide a precise engineering description of the open-source tool so that the code can be understood, reviewed, maintained, extended, and reused in a controlled way.

The scope of the manual includes:

- the software architecture implemented in Python;
- the way external software components are managed by the Python platform;
- the mathematical formulation currently implemented in the simulation and optimization engine;
- the definition of all major inputs and outputs;
- the role of the optimization template and of the early-stage LCA database;
- the assumptions, approximations, and current limitations that affect interpretation of results.

The main intended audiences are:

- software developers who need to maintain or extend the code;
- building-energy and LCA researchers who need to understand the implemented model;
- technical project partners who need a reproducible description of the digital workflow;
- advanced users who need to prepare input data and interpret outputs correctly.

## 2. Functional positioning of the tool

The Python application occupies an intermediate but strategically important role between raw data, simulation engines, and decision outputs.

At the upstream level, the tool consumes urban-scale geometric data and tabular typological data. At the downstream level, it produces both raw simulation outputs and high-level optimization results. In the middle, it manages all transformations required to go from urban geometry and retrofit assumptions to comparable performance indicators.

In this sense, the tool is not simply an EnergyPlus launcher. It is an application-specific orchestration layer that encapsulates a domain model for building stock analysis. The implemented workflow is especially suitable for situations where many buildings must be processed consistently, typological assumptions must be controlled outside the code, and intervention alternatives must be screened under more than one objective.

## 3. System context and external applications

### 3.1 Python as orchestration platform

The application is implemented in Python because Python provides an efficient balance between file-system automation, spreadsheet processing, geometry handling, simulation orchestration, and report generation. The code is currently concentrated in a single file, `main.py`, but the internal logic already reflects a modular separation of concerns.

### 3.2 External simulation and post-processing software

The following external tools are managed by the Python platform.

#### 3.2.1 EnergyPlus

EnergyPlus is the dynamic simulation engine used to evaluate the thermal and electrical behavior of the generated building models. The Python application does not modify EnergyPlus source code. Instead, it generates IDF models, resolves executable paths, runs EnergyPlus through the command line, and captures errors when execution fails.

#### 3.2.2 ReadVarsESO

ReadVarsESO is used as the standard EnergyPlus post-processor to convert `.eso` files into CSV outputs that can be read by Python. The Python code writes the `.rvi` control file, launches ReadVarsESO, and then parses the generated CSV summaries.

### 3.3 Python libraries

The current implementation relies on the following Python libraries.

- `pandas` for reading and writing Excel and CSV data.
- `openpyxl` for Excel export and chart embedding.
- `pyshp` (`shapefile`) for shapefile access.
- `shapely` for adjacency and overlap checks through buffered geometry operations.
- `ezdxf` for DXF merging during report generation.
- `tkinter` for the optional desktop GUI.

### 3.4 Why external tool management matters

A central design objective of the tool is to keep the scientific and engineering workflow reproducible while remaining compatible with external open-source/freeware tools. The Python layer therefore has to manage:

- path discovery and platform-dependent executable names;
- generation of simulation input files;
- output-directory naming and isolation;
- execution sequencing;
- conversion of low-level simulator outputs into engineering indicators;
- error surfacing in CLI and GUI modes.

## 4. Source-code architecture

Although `main.py` is a single file, it can be read as a set of logical subsystems.

### 4.1 Data structures and configuration layer

The main structured objects are:

- `SimulationConfig`: collects all runtime configuration values, including file paths, indices, report generation flags, template selection, and optimization mode.
- `BuildingTypology`: stores wall, floor, roof, window, frame, and schedule information associated with one typology.
- `ScenarioDefinition`: represents one discrete optimization scenario built from a subset of active interventions.
- `BuildingGeometrySummary`: stores the simplified geometric quantities used by the LCA layer.
- `Point`: represents one 3D coordinate used when writing EnergyPlus geometry.

These classes are important because they provide semantic structure around what would otherwise be loosely coupled dictionaries and spreadsheet rows.

### 4.2 Configuration and validation subsystem

The configuration subsystem converts raw user choices into a validated execution context.

Main responsibilities:

- locate default files and directories;
- discover weather files;
- resolve the default optimization template when present;
- validate files, sheets, executable paths, run indices, and report prerequisites.

Key functions:

- `build_default_config()`
- `default_template_path()`
- `validate_config()`
- `config_from_args()`

### 4.3 Geometry processing subsystem

The geometry subsystem starts from the shapefile polygons and turns them into a clean, simulation-ready representation.

Main responsibilities:

- sort points consistently;
- remove degenerate vertices;
- compute segment lengths and polygon area;
- detect adjacency between neighboring buildings;
- compute façade, roof, window, and opaque wall proxies.

Key functions:

- `sort_points_counterclockwise()`
- `clean_vertices()`
- `adjacent_walls()`
- `polygon_area()`
- `building_geometry_summary()`
- `compute_window_parameters()`

### 4.4 Typology loading subsystem

The typology subsystem reads the information used to assign constructions and geometric defaults to each building type.

Two modes are supported.

- **Legacy mode**: a single Excel file is used as typology source.
- **Template mode**: the optimization workbook is used, and the current sheet can be selected explicitly.

Key functions:

- `load_typology_data()`
- `load_template_sheet()`
- `load_typology()`
- `typology_source_label()`

### 4.5 IDF generation subsystem

This subsystem generates EnergyPlus-compliant text blocks for zones, floors, ceilings, roofs, walls, windows, and HVAC objects. It is the link between the typological interpretation of the building stock and the simulation engine.

Key functions:

- `generate_building_lines()`
- `build_zone_lines()`
- `build_floor_and_roof_lines()`
- `build_envelope_lines()`
- `build_window_lines()`
- `build_hvac_lines()`
### 4.5.1 How buildings are constructed from the polygon
The building-generation logic starts from the footprint polygon read from the shapefile. The polygon is first cleaned, simplified, and ordered in a geometrically consistent way. After this preprocessing step, the application treats the polygon as the base plan of the building and extrudes it vertically according to the number of floors and the typology floor height.

The process is conceptually the following:
1. the polygon vertices are cleaned with `clean_vertices()` so that repeated or nearly coincident points do not generate degenerate EnergyPlus surfaces;
2. the cleaned vertices are ordered counterclockwise to preserve a consistent surface orientation;
3. the floor count is read from the shapefile attribute `num_piani`;
4. the floor-to-floor height is read from the typology workbook through `load_typology()`;
5. `floor_levels()` generates the vertical reference elevation of each thermal zone;
6. the same footprint is reused for all floors, thereby creating a vertically extruded building model.
 
For each floor, the code generates one thermal zone named `Piano0`, `Piano1`, `Piano2`, and so on. The result is a multizone model in which each floor corresponds to one thermal zone. This choice is coherent with district-scale and stock-scale workflows because it provides a manageable level of detail while still distinguishing the main vertical thermal partitions of the building.

### 4.5.2 Generation of floors, ceilings, and roof

Once the zone structure is defined, the application creates horizontal envelope surfaces through `build_floor_and_roof_lines()`. The same plan polygon is used to define:

- the ground floor surface for the first zone;
- the intermediate floor surfaces between adjacent zones;
- the ceiling surfaces below upper floors;
- the final roof surface for the top zone.
 
The constructive assignment follows the typology workbook. In particular:

- the lowest floor uses `ground_floor_construction`;
- intermediate floors use `floor_construction`;
- intermediate ceilings use `ceiling_construction`;
- the top horizontal surface uses `roof_construction`.

This means that the constructive identity of the building is not derived from geometry alone, but from the combination of geometry and typological metadata loaded from the Excel input.

### 4.5.3 Generation of walls from polygon edges

Vertical envelope surfaces are generated by `build_envelope_lines()` and `wall_surface_line()`. Each edge of the cleaned polygon corresponds to one wall segment. For each floor and for each polygon side, the code constructs a four-vertex vertical surface using the lower and upper elevations associated with the current floor.

The wall naming convention follows the form `PianoX:MuroY`, which is useful for traceability in generated IDF files and in later debugging.

The wall construction assigned to all external walls of a given building is read from the typology row, specifically from `wall_construction`, which in turn comes from the workbook column `nome della costruzione del muro esterno`. Therefore, changing the workbook is sufficient to change the wall construction used in the generated EnergyPlus model.

### 4.5.4 Detection of adjacent walls and non-exposed façades

The code does not blindly assume that all polygon edges are externally exposed. The function `adjacent_walls()` compares each wall segment against neighboring polygons using a buffered overlap check. If another building overlaps the buffered wall segment, that wall is flagged as adjacent and is excluded from window placement.

This adjacency logic is important because it affects:

- which walls are considered exposed in the geometric/LCA summaries;
- which walls can receive windows;
- the realism of compact urban-cluster simulations where buildings are attached or very close to each other.

### 4.5.5 Generation of windows

The window-generation logic is not based on explicit window geometry in the input shapefile. Instead, windows are synthesized from typological parameters and exposed wall geometry.

The main inputs are:

- `Perc_WWR` (window-to-wall ratio);
- `altezza davanzale` (sill height);
- `Altezza finestra` (window height);
- the minimum usable wall length;
- adjacency flags that disable windows on blocked walls.

The procedure is as follows:

1. the code computes the total façade window area implied by the WWR and the building perimeter;
2. it identifies the usable exposed wall length, excluding walls that are too short or adjacent to another building;
3. it adapts window height and sill height if the requested area does not fit the available façade geometry;
4. it converts the target window area into a linear ratio along each usable wall;
5. it distributes one or more windows on each wall through `window_layout()`;
6. it writes one `FenestrationSurface:Detailed` object for each generated opening.

The generated windows are therefore typology-driven and geometry-constrained. This is a reasonable compromise for urban building-stock applications, where explicit opening-by-opening surveys are usually unavailable.

### 4.5.6 Window and frame construction assignment

Each generated window is assigned both a glazing construction and a frame construction. These values come directly from the workbook columns:

- `nome della costruzione della finestra`;
- `nome della costruzione del Telaio`.

In code, they are stored in `window_construction` and `frame_construction` inside `BuildingTypology`, and then passed to `build_window_lines()`. This means that the transparent component of the envelope is entirely controlled from the Excel input and can be varied typology by typology or scenario by scenario in optimization mode.

### 4.5.7 Assignment of construction typologies from the workbook

The building model is not built from generic default constructions. Instead, all the principal envelope constructions are assigned from the typological workbook. The assignment path is:
1. shapefile attribute `edificio-t` identifies the building typology;
2. `load_typology()` retrieves the corresponding row from the workbook; 
3. the row is converted into a `BuildingTypology` object;
4. `generate_building_lines()` and the downstream geometry functions use the fields of that object when writing EnergyPlus surfaces.

The most relevant workbook-to-model mappings are:
- external wall column -> `wall_construction`;
- ground floor column -> `ground_floor_construction`;
- roof column -> `roof_construction`;
- intermediate floor column -> `floor_construction`;
- ceiling column -> `ceiling_construction`;
- window column -> `window_construction`;
- frame column -> `frame_construction`;
- schedule column -> `schedule_name`.

As a result, the Python tool does not hard-code the constructive typologies of the stock. It reads them externally and uses them consistently in model generation, optimization, and reporting.

### 4.5.8 HVAC insertion for each thermal zone

The current tool inserts a predefined HVAC configuration for every zone through `build_hvac_lines()`. The implemented plant-side demand structure is based on a four-pipe fan-coil arrangement connected to heating and cooling water loops.
For each zone/floor, the code generates:

- zone equipment connections;
- zone equipment lists;
- inlet and exhaust node lists;
- an outdoor-air mixer;
- a constant-volume fan;
- a water cooling coil;
- a water heating coil;
- chilled-water and hot-water demand branches.

This means that every floor zone is assigned a consistent terminal-unit-based HVAC object set, rather than being left as a free-floating thermal zone without conditioning system.

### 4.5.9 Schedule assignment for the HVAC system

The operation of the zone HVAC equipment is controlled by the schedule read from the workbook column `Schedule impianto`. This value is stored in `schedule_name` and passed to `build_hvac_lines()` and `fan_coil_lines()`. Therefore, the typology workbook does not only determine envelope constructions, but also the activation schedule used by the zone terminal units.

### 4.5.10 Engineering implications of the current building-construction strategy

The adopted strategy has important engineering consequences.

First, it is scalable, because buildings can be generated in large numbers from a limited set of typological inputs.

Second, it is transparent, because each major modeling decision can be traced either to geometry or to a named workbook field.

Third, it is coherent with stock-level analysis, because it balances geometric realism and input-data practicality.

Fourth, it remains extendable, because future versions may replace or enrich the current assumptions on windows, zones, and HVAC systems without changing the overall data flow.


### 4.6 Simulation execution subsystem

This subsystem is responsible for the operational management of EnergyPlus runs.

Key functions:

- `run_command()`
- `run_simulations_for_building()`
- `execute_simulation()`
- `energyplus_paths()`

### 4.7 Reporting subsystem

The reporting subsystem converts raw simulation outputs into usable engineering deliverables.

It extracts summary values from `eplustbl.csv`, aggregates weather-specific outputs, writes workbook summaries, and merges DXF files when available. In optimization mode, it also writes scenario-level results, intervention tables, Pareto-point exports, and an embedded Pareto chart.

Key functions:

- `extract_table_summary()`
- `extract_timeseries_summary()`
- `aggregate_energy_results()`
- `write_summary_reports()`
- `write_optimization_reports()`

### 4.8 Optimization and LCA subsystem

The optimization and LCA subsystem transforms typological differences into a discrete scenario space, evaluates each scenario through simulation and simplified LCA calculations, and finally identifies the non-dominated solutions.

Key functions:

- `build_scenarios()`
- `active_intervention_columns()`
- `lca_factor_table()`
- `mapping_rows_for()`
- `lca_amount_for_mapping()`
- `pareto_flags()`
- `execute_optimization()`

### 4.9 User-interface subsystem

The application exposes the workflow through:

- a command-line interface (`parse_args()`);
- a Tkinter-based GUI (`SimulationGui`).

The GUI executes long-running tasks in a background thread and uses a queue to move log messages back to the interface safely.

## 5. File and data interfaces

### 5.1 Mandatory inputs

The minimum execution context for a simulation run consists of:

- a shapefile containing building footprints and attributes;
- an EnergyPlus base IDF template;
- one or more EPW weather files;
- a valid EnergyPlus installation directory;
- a typology workbook or a template workbook.

### 5.2 Shapefile contract

The shapefile must provide, at minimum, the attributes used by the code:

- `edificio-t`: typology identifier;
- `num_piani`: number of floors.

The polygons are assumed to represent building footprints. They are translated internally using the configured coordinate offsets so that generated IDF coordinates remain numerically well behaved.

### 5.3 Legacy typology workbook contract

The legacy typology workbook must contain the following columns:

- `Tipologia`
- `Perc_WWR`
- `altezza interpiano [m]`
- `altezza davanzale`
- `Altezza finestra`
- `nome della costruzione del muro esterno`
- `nome della costruzione del solaio controterra`
- `nome della costruzione del tetto`
- `nome della costruzione del solaio interpiano`
- `nome della costruzione del soffitto`
- `nome della costruzione della finestra`
- `nome della costruzione del Telaio`
- `Schedule impianto`

### 5.4 Optimization template contract

The optimization template adds a controlled external interface for decision support. The workbook currently relies on the following sheets.

#### 5.4.1 `Tipologie_Base`

Contains the baseline constructions and geometric defaults for each typology.

#### 5.4.2 `Tipologie_Intervento`

Contains the post-intervention constructions and geometric defaults for the same typology set.

#### 5.4.3 `Variabili_Ottimizzazione`

Defines which intervention families are active, which typology field each variable controls, and optional min/max/step values useful for LCA quantity estimation.

#### 5.4.4 `Mapping_EnergyPlus_LCA`

Connects EnergyPlus construction names or energy carriers to:

- an LCA material label;
- an LCA category;
- an LCA phase;
- a quantity expression;
- optional density and cover overrides.

#### 5.4.5 `Early_LCA_ClimateChange`

Provides the early-stage factors used by the optimization engine. At minimum, rows should contain:

- Material
- Category
- Unitary
- Average density
- Cover
- Phase
- Climate change

#### 5.4.6 `Parametri_Analisi`

Contains global analysis parameters such as the operational horizon and disposal assumptions.

### 5.5 Validation of template consistency

The code checks the presence of required optimization sheets before launching optimization mode. It also verifies that `Tipologie_Base` and `Tipologie_Intervento` contain the same typology set. This is essential because the scenario generator works by selectively substituting the changed columns between the two sheets.

## 6. Mathematical model

### 6.1 General optimization problem

The tool implements a discrete two-objective optimization problem on a cluster of buildings.

Let the set of active intervention families be:

$$
\mathcal{I} = \{i_1, i_2, \ldots, i_n\}
$$

Each candidate scenario is any subset of active interventions:

$$
S \subseteq \mathcal{I}
$$

Accordingly, the maximum number of candidate scenarios is:

$$
N_{scen} = 2^{|\mathcal{I}|}
$$

The implemented optimization problem is a bi-objective minimization problem:

$$
\min_{S \subseteq \mathcal{I}} \; \bigl(E(S),\,L(S)\bigr)
$$

where:

- \(E(S)\) is the total annual energy demand of the scenario;
- \(L(S)\) is the total climate-change impact over production, disposal, and use.

### 6.2 Energy objective

The current implementation uses EnergyPlus outputs to derive annual energy demand. The tool aggregates the following quantities:

- natural gas heating demand;
- electric cooling demand;
- electric fan demand;
- electric pump demand.

The total final energy demand is written as:

$$
E(S) = E_{gas}(S) + E_{el}(S)
$$

with the electrical component given by:

$$
E_{el}(S) = E_{cool}(S) + E_{fan}(S) + E_{pump}(S)
$$

where:

- $(E_{gas}(S))$ is the annual natural-gas demand of the scenario;
- $(E_{cool}(S))$ is the annual electricity used for cooling;
- $(E_{fan}(S))$ is the annual fan electricity demand;
- $(E_{pump}(S))$ is the annual pump electricity demand.

All these quantities are extracted from EnergyPlus summary outputs and converted to MWh before the final aggregation is written to the optimization report.

When multiple weather files are present, the software uses an arithmetic mean across weather cases. If the weather-file set is indexed by \(w = 1, \ldots, N_w\), the current aggregation rule is:

$$
\bar{E}(S) = \frac{1}{N_w}\sum_{w=1}^{N_w} E_w(S)
$$

This is a simple and transparent rule that can be generalized in future versions through weighted weather aggregation.

### 6.3 LCA objective

The total climate-change impact is expressed as:

$$
L(S) = L_{prod}(S) + L_{disp}(S) + L_{use}(S)
$$

where:

- $(L_{prod}(S))$ accounts for the production of newly installed materials and components;
- $(L_{disp}(S))$ accounts for disposal of removed materials or components when available in the mapping;
- $(L_{use}(S))$ accounts for the operational phase through annual simulated energy use and the chosen time horizon.

The generic impact contribution for one mapped item $(k)$ is:

$$
L_k = Q_k \cdot EF_k
$$

where:
- $(Q_k)$ is the calculated quantity associated with the mapping row;
- $(EF_k)$ is the climate-change factor from the early database.

At scenario level, the production and disposal terms can therefore be interpreted as:

$$
L_{prod}(S) = \sum_{k \in \mathcal{K}_{prod}(S)} Q_k \, EF_k
$$

$$
L_{disp}(S) = \sum_{k \in \mathcal{K}_{disp}(S)} Q_k \, EF_k
$$

The operational use-stage term is derived from simulated annual energy demand and the selected analysis horizon \(H\) (in years):

$$
L_{use}(S) = H \cdot \Bigl(E_{gas}(S)\,EF_{gas} + E_{el}(S)\,EF_{el}\Bigr)
$$

where $(EF_{gas})$ and $(EF_{el})$ are the early-database climate-change factors associated with natural gas and electricity.

### 6.4 Quantity evaluation rules

The current implementation uses an explicit and intentionally closed set of quantity expressions, such as:

- `area * spessore * densita`
- `area * spessore_rimosso`
- `area_vetro`
- `area_telaio o area_finestra`
- `area_vetro * spessore_vetro * densita`
- `area / cover`
- `energia_termica_annua`
- `energia_elettrica_annua`

This design improves traceability and avoids introducing uncontrolled spreadsheet expressions into the execution engine.

### 6.5 Baseline comparison and delta indicators

Once all scenarios are evaluated, the software identifies the baseline scenario and computes the following additional indicators for each scenario:

- absolute energy saving with respect to baseline;
- percentage energy saving with respect to baseline;
- final total LCA;
- absolute LCA reduction with respect to baseline;
- percentage LCA reduction with respect to baseline.

If $(S_0)$ denotes the baseline scenario and $(S)$ a generic intervention scenario, the exported deltas are:

$$
\Delta E(S) = E(S_0) - E(S)
$$

$$
\Delta E_{\%}(S) = \frac{E(S_0) - E(S)}{E(S_0)} \cdot 100
$$

$$
\Delta L(S) = L(S_0) - L(S)
$$

$$
\Delta L_{\%}(S) = \frac{L(S_0) - L(S)}{L(S_0)} \cdot 100
$$

These indicators are exported explicitly because they are often more interpretable than raw objective values for reporting and ranking purposes.

### 6.6 Pareto front definition

A scenario $(S_a)$ dominates another scenario $(S_b)$ if:

$$
E(S_a) \leq E(S_b), \qquad L(S_a) \leq L(S_b)
$$

with at least one strict improvement:

$$
E(S_a) < E(S_b) \quad \text{or} \quad L(S_a) < L(S_b)
$$

All non-dominated scenarios constitute the Pareto front. The code exports both the full scenario table and the Pareto-only subset.

## 7. Simulation workflow in detail

The simulation workflow can be described as the following deterministic sequence.

### 7.1 Configuration loading

A `SimulationConfig` object is built either from defaults, CLI arguments, or GUI inputs.

### 7.2 Input validation

The software checks:

- typology source availability;
- required template sheets in optimization mode;
- shapefile and IDF-template existence;
- weather-file availability;
- EnergyPlus and ReadVarsESO executable paths;
- consistency of start and end indices.

### 7.3 Geometry preparation

For each selected building:

1. polygon vertices are cleaned and re-ordered;
2. wall adjacency is detected;
3. floor levels are computed;
4. window parameters are adjusted to satisfy wall-length constraints and WWR targets.

### 7.4 IDF generation

The software writes:

- zones;
- floors and ceilings;
- roofs;
- exposed walls;
- fenestration surfaces;
- a hard-coded HVAC demand-side snippet.

The generated text is appended to the base IDF template and written to a per-building `.idf` file.

### 7.5 EnergyPlus execution

For each building and weather file:

1. a weather-specific output directory is created;
2. EnergyPlus is launched;
3. an `.rvi` file is written;
4. ReadVarsESO is executed for the selected report profiles.

### 7.6 Post-processing

The software extracts the selected metrics from `eplustbl.csv` and `eplusout.csv`, writes summary workbooks, and optionally merges DXFs.

## 8. Optimization workflow in detail

### 8.1 Scenario generation

The optimization workflow begins by comparing baseline and intervention sheets. Only columns that satisfy both conditions are considered:

- the corresponding optimization variable is active;
- the baseline and intervention values are different.

The software then builds all combinations of those columns and creates one `ScenarioDefinition` for each combination.

### 8.2 Scenario simulation

Each scenario is simulated by reusing the same simulation engine but replacing the typology data with the scenario-specific sheet built in memory.

### 8.3 Cluster-level energy aggregation

After simulation, the software aggregates scenario energy over the full building cluster and over all weather files. The current aggregation returns cluster-level total area, gas demand, electricity demand, and total final energy.

### 8.4 Scenario-level LCA evaluation

For each scenario and for each building, the code:

1. computes the simplified geometry summary;
2. compares baseline and scenario typology values;
3. retrieves production mappings for installed components;
4. retrieves disposal mappings for removed components;
5. computes use-phase impacts from annual simulated energy demand;
6. sums all contributions into the final scenario LCA result.

### 8.5 Report generation

The optimization report writer exports:

- complete scenario results;
- interventions by typology;
- Pareto front table;
- Pareto points table;
- optimization summary;
- Pareto summary;
- embedded Pareto chart in Excel.

## 9. Output specification

### 9.1 Standard simulation outputs

The standard simulation workflow produces:

- one generated IDF per building;
- one weather-specific output directory per building;
- `outputtbl.csv`;
- `output.xlsx`;
- `merged.dxf` when DXF output and report dependencies are available.

### 9.2 Optimization outputs

The optimization workflow produces, at minimum:

- `optimization_results.csv`;
- `optimization_results.xlsx`;
- `optimization_interventions_by_typology.csv`;
- `pareto_points.csv`.

The workbook contains the following sheets:

- `ScenarioResults`
- `ParetoFront`
- `ParetoPoints`
- `InterventiTipologie`
- `ParetoSummary`
- `OptimizationSummary`
- `ParetoChart`

### 9.3 Interpretation of key result fields

The most relevant fields for reporting are:

- `interventi`: high-level description of the intervention combination;
- `energy_total_mwh`: final energy demand of the scenario;
- `energy_saved_mwh_vs_baseline`: absolute energy saving with respect to baseline;
- `energy_saved_pct_vs_baseline`: percentage energy saving with respect to baseline;
- `lca_final_kgco2eq`: final scenario climate-change impact;
- `lca_reduction_kgco2eq_vs_baseline`: absolute reduction in climate-change impact with respect to baseline;
- `pareto`: whether the scenario belongs to the non-dominated set.

## 10. GUI and CLI interfaces

### 10.1 CLI interface

The command-line interface supports both direct simulation and optimization. The main switches are:

- `--run-defaults`
- `--excel`
- `--input-template`
- `--typology-sheet`
- `--optimize`
- `--shapefile`
- `--idf-template`
- `--weather`
- `--weather-dir`
- `--energyplus-dir`
- `--output-dir`
- `--start-index`
- `--end-index`
- `--skip-reports`

### 10.2 GUI interface

The GUI mirrors the same logic through text entries, file and directory selectors, report toggles, and an optimization checkbox. The GUI is implemented in Tkinter and uses a worker thread plus queue-driven log propagation to keep the interface responsive.

## 11. Error handling philosophy

The application follows a fail-fast style on configuration and execution problems. Typical failure conditions include:

- missing files or missing template sheets;
- missing Python dependencies;
- invalid scenario definitions;
- failed external command execution;
- unavailable weather files or EnergyPlus executables.

Errors are raised as Python exceptions in CLI mode and surfaced as GUI message boxes in GUI mode. This makes the workflow explicit and easier to debug than silent fallback logic.

## 12. Assumptions, approximations, and limitations

The current tool is fully operational but intentionally bounded.

### 12.1 Geometric assumptions

- Buildings are represented through 2D footprints extruded by the number of floors and floor height.
- Wall adjacency is approximated through buffered overlap of sampled segments.
- Window layout is distributed evenly along usable wall segments.

### 12.2 HVAC assumptions

The HVAC model is currently a hard-coded fan-coil demand-side snippet. The tool is therefore best understood as a geometry-and-envelope-oriented model generator rather than a general HVAC configurator.

### 12.3 Optimization assumptions

The optimization is currently discrete. It compares baseline and intervention constructions already present in the typology sheets. Continuous generation of new EnergyPlus constructions for arbitrary insulation thicknesses is not yet implemented.

### 12.4 LCA assumptions

The LCA layer is intentionally simplified and early-stage oriented. It relies on geometric proxies, restricted quantity expressions, and a small external factor database. It is excellent for comparative screening, but not a substitute for a fully detailed project LCA compliant with all project-level inventory requirements.

### 12.5 Computational assumptions

The current execution flow is serial. No building-level or scenario-level parallelism is implemented yet.

## 13. Recommended engineering roadmap

The most relevant future developments are:

- refactor the single-file implementation into modules such as `geometry`, `idf`, `simulation`, `optimization`, `lca`, `reporting`, and `ui`;
- support parametric insulation families directly in EnergyPlus material layers;
- enrich end-of-life mappings and multi-indicator LCA handling;
- introduce weighted weather aggregation and optional ranking methods beyond Pareto filtering;
- add regression tests on workbook parsing and output generation;
- add parallel execution strategies for large building clusters.

## 14. Reproducibility and operational traceability

The software already supports good reproducibility practices because:

- the typology and optimization logic are externalized in spreadsheets;
- scenario output directories are isolated and timestamped;
- CLI and GUI access use the same underlying engine;
- baseline and scenario deltas are exported explicitly;
- the Pareto front is exported both numerically and graphically.

For rigorous project workflows, it is recommended to archive alongside the results:

- the exact version of `main.py` used;
- the template workbook used;
- the base IDF template;
- the shapefile version;
- the weather files used;
- the EnergyPlus version.

## 15. Final remarks

The current Python tool should be regarded as a robust engineering prototype already capable of supporting technically meaningful district-scale scenario analysis. Its main strength lies in the explicit integration of geometry processing, dynamic simulation, early-stage LCA, and Pareto-based comparison within one transparent and configurable workflow. This makes it especially suitable for research and applied decision support in building-stock retrofit studies where both energy and climate-oriented environmental objectives must be considered together.

## 16. References

1. Applied Sciences (2025), *AI-Driven Multi-Objective Optimization and Decision-Making for Urban Building Energy Retrofit: Advances, Challenges, and Systematic Review*. URL: https://www.mdpi.com/2076-3417/15/16/8944
2. Sustainability (2025), *Integrating Building Information Modeling and Life Cycle Assessment to Enhance the Decisions Related to Selecting Construction Methods at the Conceptual Design Stage of Buildings*. URL: https://www.mdpi.com/2071-1050/17/7/2877
3. Renewable and Sustainable Energy Reviews (2024), *Building renovations and life cycle assessment*. URL: https://www.sciencedirect.com/science/article/pii/S1364032124005008
4. Sustainability (2024), *Multi-Objective Optimization of Building Envelope Retrofits Considering Future Climate Scenarios: An Integrated Approach Using Machine Learning and Climate Models*. URL: https://www.mdpi.com/2071-1050/16/18/8217
5. Heliyon (2025), *A review on multi-objective optimization of building performance: current trends, tools, indicators, and future directions*. URL: https://www.sciencedirect.com/science/article/pii/S2405844025008606
6. Springer (2023), *Life Cycle Assessment at the Early Stage of Building Design*. URL: https://link.springer.com/chapter/10.1007/978-3-031-29515-7_42
7. Environment, Development and Sustainability (2023-2024), *Embodied energy assessment: a comprehensive review of methods and software tools*. URL: https://link.springer.com/article/10.1007/s10668-023-04015-0