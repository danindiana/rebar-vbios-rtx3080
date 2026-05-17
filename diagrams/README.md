# Export Control by Software Means — Graphviz Diagram Pack

Five diagrams, each provided as `.dot`, `.png`, and `.svg`.

These diagrams explain a plausible mechanism by which hardware capabilities can be exposed, withheld, or tiered through firmware/driver/software policy. They are framed as mechanism diagrams, not proof of intent for any specific consumer GPU launch decision.

## Files

1. `01_policy_to_registers.*` — legal/policy thresholds mapped to configurable hardware behavior.
2. `02_rebar_firmware_gate.*` — ReBAR as a firmware/platform-gated capability.
3. `03_bar_aperture_overhead.*` — CPU/driver overhead difference between 256 MiB and 8192 MiB BAR1.
4. `04_evidence_ladder.*` — proven vs plausible vs unproven claim boundary.
5. `05_selective_enablement_plane.*` — selective enablement control plane across firmware, driver, SKU, and market.

## Claim boundary

- Proven in the worlock RTX 3080 case: correct ROM identity and BAR1 expansion after proper VBIOS/platform enablement.
- Proven publicly: export-control rules can be expressed as performance/transfer thresholds, and vendors have produced compliant lower-capability variants.
- Plausible: firmware/software control surfaces can function as product-segmentation or compliance levers.
- Unproven: specific export-control intent behind the original RTX 30 256 MiB BAR1 configuration.
