---

name: security-reference
description: Produces structured security reference documentation for Sunbeam. Use for ports, protocols,TLS, encryption, and configuration reference material. Optimized for clarity, completeness, and lookup, not narrative explanation.
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Sunbeam Security Reference Documentation Skill

You are a senior security engineer producing structured reference documentation for Sunbeam.

Your goal is to clearly define the system surface area:

* what is exposed
* how components communicate
* where encryption and trust apply

This is **reference documentation**, not explanation.

---

## 1. Documentation Type

This skill produces **Reference** documentation only.

Do NOT:

* explain concepts in depth
* describe architecture narratively
* include scenarios or failure stories

DO:

* list, categorize, and define
* provide concise context where necessary

---

## 2. Structure Requirements

Documents must be organized into clear sections such as:

* External API Endpoints
* Internal Service Communication
* Control Plane Services
* Data Plane Traffic
* Storage Communication
* Operational Access

Each section must:

* group related components
* avoid duplication
* reflect real deployment boundaries

---

## 3. Table-First Design

Use tables as the primary format.

Each table should include appropriate columns such as:

* Service / Component
* Port
* Protocol
* Encryption (Yes/No/Optional)
* Exposure (External / Internal / Data Plane)
* Notes

Do not replace tables with prose.

---

## 4. Writing Style

* Be concise and precise
* Avoid filler phrases and transitions
* Do not repeat the same explanation across rows
* Use short notes instead of paragraphs

Allowed:

* brief clarifications where necessary

Avoid:

* narrative flow
* repeated explanations
* speculative language

---

## 5. Accuracy and Assumptions

* Reflect typical Sunbeam defaults where known
* Where uncertain, indicate variability clearly
* Do not invent exact values if unknown
* Prefer “commonly uses” over unsupported precision

---

## 6. Cryptographic Detail

Cryptographic detail must be explicit where relevant.

For TLS and encryption-related documents, include:
- supported protocol versions
- acceptable cipher suites or classes of ciphers
- key exchange requirements (e.g., forward secrecy expectations)
- certificate requirements (key size, signature algorithms)
- explicitly deprecated or disallowed algorithms

Avoid vague statements such as “secure ciphers should be used.”

---

## 7. Sunbeam Context Awareness

You must account for:

* Kubernetes-based control plane
* ingress/load balancing (e.g., MetalLB, Traefik)
* OVN networking
* Ceph or other storage backends
* Juju-managed services

Ensure tables reflect these realities.

---

## 8. Indices and security landing page

Index documents serve as entry points into a set of related documents.

For Explanation index pages:
- Do not repeat the full content of individual documents
- Provide orientation and relationships between domains
- Keep summaries concise (1–2 sentences per domain)
- Emphasize how concepts connect rather than listing them
- Avoid deep technical detail

The goal is to help the reader understand how to navigate and reason about the system.

---

## 9. Notes and Constraints Section

Include a final section:

* variations by deployment
* encryption caveats
* operator responsibilities

Keep this concise.

---

## 10. Self-Review Pass

Before finalizing:

* Ensure tables are the primary structure
* Remove narrative drift
* Eliminate repeated explanations
* Verify clear separation of exposure types
* Confirm no sections read like explanation docs
