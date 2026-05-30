Partitioning GitHub’s relational databases to handle scale

[![YouTube Thumbnail](https://img.youtube.com/vi/Tq1fif3rcnQ/maxresdefault.jpg)](hhttps://youtu.be/Tq1fif3rcnQ?si=408fiuCe9LybXPpB)

![Mind Map](MindMaps/002.png)

## 1.0 The Challenge: A Monolithic Database at Hyper-Growth

In the lifecycle of a rapidly growing platform, architectures that prioritize initial development speed often accrue a "scalability debt" that must eventually be paid. For GitHub, this debt manifested in its core database infrastructure. The platform began with a monolithic architecture centered on a single, powerful MySQL database cluster. This served well for years, but hyper-growth eventually turned this into an operational bottleneck.

GitHub's initial infrastructure revolved around the main database cluster **mysql1**, which stored data for core features including user profiles, repositories, issues, and pull requests. Despite improvements such as read replicas and ProxySQL, the architecture remained fundamentally monolithic.

### Key Challenges

- **High Query Volume:** By 2019, mysql1 averaged **950,000 queries/s**, heavily stressing the hardware.
- **Haphazard Query Patterns:** Lack of clear data boundaries led to unpredictable and complex query behavior.
- **Noisy Neighbor Problem:** Spikes in one feature (e.g., pull requests) degraded performance in others.
- **Cascading Outages:** The monolithic database represented a single point of failure for the entire platform.

Vertical scaling reached its limits, making architectural decomposition essential.

---

## 2.0 A Two-Tiered Partitioning Philosophy: Virtual Separation Before Physical Migration

To safely dismantle the monolithic database, GitHub used a **two-tiered strategy** that separated application-level work from infrastructure-level operations.

### Two Core Tiers

1. **Virtual Partitioning** — Application-level separation of logical table groups using schema domains and linters.
2. **Physical Partitioning** — Actual migration of self-contained table groups to new database clusters.

This approach allowed GitHub to solve the toughest application logic issues before touching production data, dramatically reducing cutover risk.

---

## 3.0 Phase 1: Virtual Partitioning via Schema Domains and Linters

Virtual partitioning simulated the post-migration state inside the existing monolith. The focus: create, codify, and enforce **strict boundaries** between table groups.

### Schema Domains

Schema domains are logical groupings of tightly coupled tables supporting a feature set.

Example: **gists** domain

- `gists`
- `gist_comments`
- `starred_gists`

Domains were defined in `db/schema-domains.yml`.

### SQL Linters

Two custom linters enforced boundaries:

- **Query Linter:** Prevents cross-domain joins in dev/test; violations require explicit exemptions.
- **Transaction Linter:** Detects multi-domain transactions in production via sampling.

### Techniques for Resolving Violations

- **Application-Side Joins:** Replace DB joins with multiple single-domain queries.
- **ActiveRecord Modifications:** Preferring `preload` over `includes`, adding `disable_joins`.
- **Handling Polymorphic Tables:** Splitting tables or duplicating read-only data with sync.

A domain reached "virtually partitioned" status once it had **zero exemptions**.

---

## 4.0 Phase 2: The Mechanics of a Zero-Downtime Physical Migration

Once virtual isolation was complete, GitHub performed high-precision physical migrations with near-zero downtime.

### Primary Method: Custom Write-Cutover Process

This method used **ProxySQL** and **MySQL GTID replication**.

#### Two Phases

1. **Pre-Cutover Traffic Redirection**

   - Apps connected to the new cluster's ProxySQL.
   - ProxySQL routed all traffic back to the old cluster.

2. **Cutover Event** (milliseconds)

   1. Block writes on source cluster.
   2. Ensure destination cluster catches up via GTID.
   3. Stop replication and reroute traffic to the new cluster.
   4. Re-enable writes.

This resulted in only a handful of errors.

### Alternative Approach: Vitess

Vitess is being adopted for future scaling, but the team chose the custom approach for this mission-critical migration.

---

## 5.0 Quantifiable Results and Strategic Impact

| Metric               | 2019 (Pre-Partitioning) | 2021 (Post-Partitioning) |
| -------------------- | ----------------------- | ------------------------ |
| Total Avg. Queries/s | 950,000                 | 1,200,000                |
| Average Host Load    | Heavily strained        | **50% reduction**        |

Despite a **26% increase in overall query volume**, per-host load was **cut in half**.

This significantly reduced database-related incidents and improved platform-wide reliability.

---

## 6.0 Conclusion: Key Learnings from GitHub's Partitioning Journey

### Strategic Takeaways

- **Phased, Risk-Averse Execution:** Virtual partitioning first, physical migration second.
- **Invest in Custom Tooling:** Schema domains and SQL linters enabled safe decomposition.
- **Pragmatic Technology Choices:** Blend reliable MySQL methods with modern tools like ProxySQL and Vitess.
- **Measure Everything:** Sub-second GTID lag and detailed observability ensured confidence.

GitHub's multi-year effort shows that breaking apart a monolithic database requires disciplined boundaries, automated verification, and methodical execution—not just big machines or hopeful migrations.

This approach created a scalable, future-proof foundation that continues to support GitHub's ongoing growth.
