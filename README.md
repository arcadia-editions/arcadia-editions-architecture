# arcadia-editions-architecture

The architecture world model for Arcadia Editions: the master manifest, cross-domain business flows, and architecture decision records.

This repository holds `zenwave-architecture.yml`, the single index of every domain, subdomain, service, and contract in the platform, plus the content that doesn't belong to any one service repository because it spans multiple bounded contexts or describes the architecture itself.

## Contents

- `zenwave-architecture.yml`: the architecture manifest — the index of the whole system, and the source `EventCatalog` generation reads from
- `business-flows/`: cross-domain business flows (`.zfl`) that don't belong to a single bounded context
- `adrs/`: architecture decision records
- `skills/`: AI agent skills used when working across this architecture

## Purpose

Use this repository to keep the architecture manifest, cross-domain flow documentation, and architecture decisions separate from the executable API and pipeline repositories, each of which owns its own domain model and contracts.
