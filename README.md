# Banking Database Performance Optimization (MySQL)

This project simulates a **production-scale banking database** and documents the process of optimizing a slow analytical query using real-world database techniques.

The goal was not just to make a query faster, but to understand *why* certain approaches fail at scale and what actually works in production systems.

---

## Dataset Overview

| Tables | Row Count |
|-----|----------|
| customers | ~49,000 |
| accounts | ~80,000 |
| transactions | ~6,000,000 |
| branches | 10 |

---

## Problem Statement

Retrieve the **top 50 customers by total account balance**, along with their **most recent transaction date within the last 30 days**.

This is a common reporting requirement in financial systems.

### Read full doucmentation here
