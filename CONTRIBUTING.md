# Contributing to Stratum-Core

Contributions to **Stratum-Core** are welcomed. This repository defines the foundational topology, isolation boundaries, and security hardening baselines for the Stratum platform.

---

## 1. Architectural Invariants

Every contribution must maintain the following architectural principles established by **Sandra Gabriela Puerto Torres**:

1. **Isolation by Design:** Segmentation is enforced at the Docker network level (`bridge` / `internal: true`), never through ad-hoc access control lists.
2. **Single Responsibility per Component:**
   * `dmz/`: Provisions and anchors `stratum_dmz`. No business services or proxies.
   * `gateway/`: Terminates ingress traffic via Cloudflare Tunnel and NPM. No database credentials.
   * `database/`: Manages persistence engines behind an air-gapped boundary proxy (`nginx-database`).
3. **Hardening Baseline:**
   * `read_only: true` on container root filesystems wherever technically feasible.
   * `cap_drop: [ALL]` with minimal, explicit capability additions (`cap_add`).
   * `no-new-privileges: true` on all services.
   * `tmpfs` mounts for temporary and volatile runtime storage.
   * Zero host-published ports (all ingress traffic passes through authenticated Cloudflare Tunnels).

---

## 2. Contribution Workflow

1. **Fork the Repository:** Create a fork on GitHub.
2. **Create a Topic Branch:**
   ```bash
   git checkout -b feat/component-improvement
   ```
3. **Validate Compose Specifications:**
   Ensure your Compose files are valid and conform to Compose Specification v2:
   ```bash
   docker compose config
   ```
4. **Documentation Sync:**
   If a change impacts network subnets, environment variables, or service behavior, update the corresponding `README.md` and Document Control table.
5. **Commit Conventions:** Use concise, imperative commit messages:
   * `feat(gateway): update npm healthcheck start period`
   * `fix(database): pin mongo version to 7.0 series`
   * `docs(dmz): clarify anchor container lifecycle`
6. **Submit a Pull Request:** Submit your PR against `main` with detailed justification and validation steps.

---

## 3. Security Considerations

Do not submit pull requests that relax hardening baselines (`cap_drop`, `read_only`, `tmpfs`, or `no-new-privileges`) without explicit technical justification and CVE mitigation analysis.

For security vulnerability disclosures, refer to [SECURITY.md](SECURITY.md).
