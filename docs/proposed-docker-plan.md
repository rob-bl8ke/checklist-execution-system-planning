## Plan: Fresh Compose Builds With Persistent DB

Adopt a two-mode container strategy: a dev-compose workflow for another machine that always builds latest source when run with --build, and a production-compose workflow that pulls immutable versioned images from a registry. Keep the named db-data volume in both modes so SQLite persists across restarts.

**Steps**
1. Phase 1 - Compose file separation and intent boundaries.
2. Keep the existing planning compose file as the base local-stack definition for persistent runtime wiring (ports, DB_PATH, named volume) and use it as the shared foundation.
3. Add a production-focused compose file in the planning repo that references registry images for API and UI (no build contexts), retains service wiring and db-data volume, and supports pinned tags for reproducibility. Depends on 2.
4. Add a containerized-dev override compose file in the planning repo that keeps build contexts for API and UI and is designed to be run with up --build for latest code on another machine. Parallel with 3.
5. Phase 2 - Command workflow and documentation.
6. Update planning README command sections to define three explicit run paths: native local development (no Docker), containerized dev on another machine (compose with override + --build), and production-like deploy (pull versioned images). Depends on 3 and 4.
7. Add reset guidance clarifying when to preserve volume versus full reset (down versus down --volumes) so stale data is not mistaken for stale code. Parallel with 6.
8. Phase 3 - CI/release alignment for production images.
9. Define image tagging contract (for example semver plus commit SHA) and reference tags in production compose file to avoid floating latest drift. Depends on 3.
10. Ensure registry publish flow exists (or is planned) so production compose can pull expected tags reliably. Depends on 9.

**Relevant files**
- c:/Code/rob-bl8ke/checklist-execution-system-planning/docker-compose.yml - Keep as shared baseline for service topology and persistent db-data volume.
- c:/Code/rob-bl8ke/checklist-execution-system-planning/docker-compose.prod.yml - New file for production deployable containers using image references.
- c:/Code/rob-bl8ke/checklist-execution-system-planning/docker-compose.dev.yml - New override for another-machine containerized development builds.
- c:/Code/rob-bl8ke/checklist-execution-system-planning/README.md - Update operational commands and environment-specific workflow documentation.
- c:/Code/rob-bl8ke/checklist-execution-system-api/Dockerfile - Reuse existing multi-stage build output expectations for registry image publishing.
- c:/Code/rob-bl8ke/checklist-execution-system-ui/Dockerfile - Reuse existing multi-stage build output expectations for registry image publishing.

**Verification**
1. Containerized dev validation: run compose with base plus dev override and --build, make a small source change in API/UI, rerun same command, confirm changed behavior appears without volume deletion.
2. Persistence validation: stop and restart stack with down then up, confirm prior SQLite data remains.
3. Reset validation: run down --volumes then up, confirm database resets while code still reflects latest build.
4. Production-path validation: run production compose file that references pinned image tags and verify containers start without local source build contexts.
5. Regression guard: verify API migration startup still runs and UI still serves built assets correctly in both dev-compose and prod-compose paths.

**Decisions**
- Accepted: --build is acceptable for containerized development on another machine.
- Accepted: production should pull versioned images from a registry.
- Included scope: compose orchestration strategy, command workflow, documentation, and verification approach.
- Excluded scope: changing application business logic, replacing SQLite, or introducing Docker-based live-reload development.

**Further Considerations**
1. Recommended tag policy: immutable tags per commit SHA plus optional semver aliases to balance traceability and release readability.
2. Recommended default profile: keep local native development as default and treat containerized development as explicit opt-in commands.
