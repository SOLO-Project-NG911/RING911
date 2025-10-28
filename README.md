# RING911

RING911 is the reference implementation for the Next Generation 9-1-1 Core Services (NGCS) as defined in NENA STA-010.  
It is part of the **SOLO** project, which also includes **CONG**, the conformance suite.

## Overview
- 100% implementation of STA-010 normative requirements  
- Written in Rust  
- Built for clarity, conformance, and operational reliability  
- Structured for two profiles:  
  - `ref`: minimal, testable reference implementation  
  - `prod`: full production deployment (HA, security, observability)

## Repository Layout
    ring/
      crates/              # Rust crates (core logic, adapters, transports)
      profiles/            # Deployment profiles (ref, prod)
      docs/                # Design documents

## Key Documents
- `solo_project_definition.md` — vision, architecture, and standards mapping  
- `profiles_layering.md` — runtime layering and profile separation

## License
Apache 2.0, see LICENSE.

Comments, issues, and pull requests are welcome.
