---
title: "Slava Sarzhan on evaluating open source"
date: 2026-05-21
quote: |
  How to evaluate “open source” in 2026

  The bigger picture here is that “open source” today often exists more in theory than in practice. It pays to look past the badge and check the operating reality. Three questions to ask before you commit to an operator:

  1. Are the container images publicly redistributable?

  If you cannot pull the official images without authentication, or you cannot mirror them to your private registry without a license review, your air-gapped and GitOps stories are constrained from day one. This is the question that turned out to be the most consequential one for MinIO, Bitnami, and Crunchy users in 2025.

  2. Are core operational features in the open-source build, or behind a paywall?

  Backup, monitoring, HA, and security features should be in the build everyone uses, not gated behind an enterprise tier. A “community edition” that omits the feature most teams actually need is a marketing build, not a real open-source build.

  3. Is the governance and roadmap public?

  A project where you can see the issues, the PRs, and the roadmap is one you can plan around. The Percona PG Operator’s public roadmap is an example of what this looks like in practice. A project run inside a vendor’s private tracker, by contrast, gives you no visibility.

  These are not gotchas. They are the questions that decide whether a project will still serve you the same way in three years.
source: "Not All Open Source Is Equal: Choosing a PostgreSQL Operator for Kubernetes in 2026"
source_url: "https://www.percona.com/blog/not-all-open-source-is-equal-choosing-postgresql-operator-kubernetes-2026/"
tags: ["open-source"]
---
