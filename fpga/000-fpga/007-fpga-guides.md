
## Digital Design/RTL

- [FSM](007-fpga-guides-fsm.md)
	
- [Verilog RTL coding](007-fpga-guides-verilog-rtl-coding.md)
	
- [RTL Coding discipline](007-fpga-guides-rtl-coding-discipline.md)
	- [RTL Naming Conventions](007-fpga-guides-rtl-naming-conventions.md)
	- [RTL Modular Hierarchy](007-fpga-guides-rtl-modular-hierarchy.md)
	- [RTL Reset Consistency](007-fpga-guides-rtl-reset-consistency.md)
	- [RTL No Unintended Latches](007-fpga-guides-rtl-no-unintended-latches.md)
	- [RTL Deterministic Behavior](007-fpga-guides-rtl-deterministic-behavior.md)
	
- [RTL flow control](007-fpga-guides-rtl-flow-control.md)
	- [Valid/ready contract](007-fpga-guides-rtl-valid-ready-contract.md)
	- [Stall propagation](007-fpga-guides-rtl-stall-propagation.md)
	- [Skid buffers](007-fpga-guides-rtl-skid-buffers.md)
	- [Avoiding combinational loops](007-fpga-guides-rtl-avoiding-combinational-loops.md)
	
- [Pipeline design basics](007-fpga-guides-rtl-pipeline-design-basics.md)
	- [Stage balancing](007-fpga-guides-rtl-pipeline-stage-balancing.md)
	- [Latency tracking](007-fpga-guides-rtl-pipeline-latency-tracking.md)
	- [Throughput vs Latency Tradeoff](007-fpga-guides-rtl-pipeline-throughput-vs-latency-tradeoff.md)
	- [Register Insertion Strategy](007-fpga-guides-rtl-pipeline-register-Insertion-strategy.md)
	
- [Combinational depth control](007-fpga-guides-rtl-combinational-depth-control.md)
	- [Сritical path reduction](007-fpga-guides-rtl-сritical-path-reduction.md)
	- [Logic splitting](007-fpga-guides-rtl-logic-splitting.md)
	- [Retiming awareness](007-fpga-guides-rtl-retiming-awareness.md)
	- [Fanout management](007-fpga-guides-rtl-fanout-management.md)
## High-Speed/Interface Knolage

- [FPGA clocking architecture](007-fpga-guides-clocking-architecture.md)
	- [FPGA Clock Generation](007-fpga-guides-сlock-generation.md)
	- [FPGA Phase Alignment](007-fpga-guides-phase-alignment.md)
	- [FPGA Clock Routing](007-fpga-guides-clock-routing.md)
	- [FPGA Jitter Impact on Timing](007-fpga-guides-jitter-Impact-on-timing.md)
	
- [SerDes](007-fpga-guides-serdes.md)
	- [Line rate](007-fpga-guides-serdes-line-rate.md)
	- [Encoding (8b/10b)](007-fpga-guides-serdes-encoding-8b-10b.md)
	- [Reference Clock](007-fpga-guides-serdes-reference-clock.md)
	- [Lane Structure](007-fpga-guides-serdes-lane-structure.md)
	- [Signal Integrity Basics](007-fpga-guides-serdes-signal-integrity-basics.md)
	
* [DDR controller architecture](007-fpga-guides-ddr-controller-architecture.md)
	* [Address mapping](007-fpga-guides-ddr-address-mapping.md)
	* [Burst behavior](007-fpga-guides-ddr-burst-behavior.md)
	* [Read/write latency](007-fpga-guides-ddr-read-write-latency.md)
	* [Controller buffering](007-fpga-guides-ddr-controller-buffering.md)
	
- [DDR training](007-fpga-guides-ddr-training.md)
	- [Calibration phases](007-fpga-guides-ddr-training-calibration-phases.md)
	- [Delay alignment](007-fpga-guides-ddr-training-delay-alignment.md)
	- [Failure symptoms](007-fpga-guides-ddr-training-failure-symptoms.md)
	- [Timing sensitivity](007-fpga-guides-ddr-training-timing-sensitivity.md)
## FPGA/Prototyping

- [FPGA design flow](007-fpga-guides-design-flow.md)
	
- TCL
	- [TCL constraint injection](007-fpga-guides-tcl-constraint-injection.md)
	- [TCL project reproducibility](007-fpga-guides-tcl-project-reproducibility.md)
	- [TCL automating builds](007-fpga-guides-tcl-automating-builds.md)
	
- [ILA Debug](007-fpga-guides-ila-debug.md)
	- [Probe selection](007-fpga-guides-ila-probe-selection.md)
	- [Trigger strategy](007-fpga-guides-ila-trigger-strategy.md)
	- [Hypothesis-driven debug](007-fpga-guides-ila-hypothesis-driven-debug.md)
	- [Signal visibility limits](007-fpga-guides-ila-signal-visibility-limits.md)
	
- [Transceiver Configuration](007-fpga-guides-transceiver-configuration.md)
	- [Reference clock setup](007-fpga-guides-transceiver-reference-clock-setup.md)
	- [Сonstraint injection](007-fpga-guides-transceiver-сonstraint-injection.md)
## System-Level/SoC Awareness

- [Address mapping](007-fpga-guides-system-level-address-mapping.md)
	- [Address space layout](007-fpga-guides-system-level-address-space-layout.md)
	- [Memory-mapped I/O](007-fpga-guides-system-level-address-memory-mapped-I-O.md)
	- [Alignment and decoding](007-fpga-guides-system-level-address-alignment-and-decoding.md)
	
- [Clock domain strategy](007-fpga-guides-system-level-clock-domain-strategy.md)
	- [Domain partitioning](007-fpga-guides-system-level-cds-domain-partitioning.md)
	- [Crossing points planning](007-fpga-guides-system-level-cds-crossing-points-planning.md)
	- [Clock tree thinking](007-fpga-guides-system-level-cds-clock-tree-thinking.md)


## Programming/Automation

- [Debug mindset](007-fpga-guides-debug-mindset.md)
	- [Hypothesis-Driven Debugging](007-fpga-guides-hypothesis-driven-debugging.md)
	- [Isolation First](007-fpga-guides-isolation-first.md)
	- [Minimal Reproduction](007-fpga-guides-minimal-reproduction.md)
	
- [Basic scripting for build automation](007-fpga-guides-basic-scripting-for-build-automation.md)
	- [Scripted runs](007-fpga-guides-scripted-runs.md)
	- [Build reproducibility](007-fpga-guides-build-reproducibility.md)
	- [Automation of regression](007-fpga-guides-automation-of-regression.md)