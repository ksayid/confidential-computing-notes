---
title: Open Policy Agent
parent: Tools & Frameworks
nav_order: 2
---

> "Treat policy as a separate concern....just like DB, messaging, monitoring, logging, orchestration, CI/CD ..." - Torin Sandall, Co-Creator, OPA

## Introduction

Open Policy Agent (OPA) is an open source, general‑purpose policy engine. Instead
of baking authorization or configuration rules directly into your
applications, you define them in a high‑level declarative language called Rego
and let OPA evaluate those policies at runtime. OPA can run as a sidecar,
as a daemon, or be embedded as a library, giving you flexibility in how policies
are deployed and updated.

Separating policy from application logic means you can change or audit rules
without rebuilding your services. A single OPA instance can answer a variety of
questions: "Is this Kubernetes resource allowed?", "Does this user have access to
the API?", or "Is this infrastructure configuration compliant?". Because Rego is
data-driven, policy updates are just another file change—no need to redeploy the
service itself.

## Typical Use Cases

* **Kubernetes Admission Control** – Integrations such as Gatekeeper rely on OPA
  to validate resource manifests before they are persisted to the cluster.
* **API Authorization** – Microservices can query OPA to check whether a user is
  permitted to invoke a particular endpoint or action.
* **Configuration Validation** – CI/CD pipelines often execute OPA policies to
  ensure infrastructure-as-code or container images conform to organizational
  standards.

