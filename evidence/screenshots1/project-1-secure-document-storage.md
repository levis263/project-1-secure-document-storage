---
title: "Secure Document Storage"          # TODO: confirm against your actual schema
kind: project                              # TODO: confirm exact enum value (lowercase per your own build rule)
status: "in-progress"                      # TODO: confirm your platform's actual status vocabulary
summary: "Cross-account, least-privilege document storage on AWS using IAM, STS, S3 and KMS."
date: 2026-09-13
stack: ["IAM", "STS", "S3", "KMS", "AWS Organizations"]
---

## Overview

This project implements a secure cross-account document storage architecture on AWS, built across a real three-account AWS Organization rather than simulated within a single account.

An application identity in the App account requires controlled access to sensitive documents stored in a separate Data account. The design uses IAM, STS, S3, and KMS to establish a controlled authorization boundary between the application and the data, separating:

- Identity authentication from authorization
- Role assumption from resource access
- KMS key administration from cryptographic key usage
- Application access from key administration

## Objective

Enable an authorised application identity to retrieve a sensitive document from another AWS account without holding permanent Data-account credentials.

The implementation targets:
1. No public access to the S3 bucket
2. Explicit authorisation for document access
3. Cross-account access via temporary STS credentials, not permanent credentials
4. Encryption at rest using a customer-managed KMS key
5. Separation of KMS key administration from document access
6. Least-privilege permissions on the document-access role
7. A testable, evidenced end-to-end access path

## Architecture & Account Design

### Account structure

| Account | Account ID | Role in the architecture |
|---|---|---|
| Management | `500481070920` | Administrative and management activities |
| Don App Account | `291827353880` | Represents the application/user requesting access |
| Data Account | `906099689108` | Hosts the sensitive S3 data and customer-managed KMS key |

The application and data environments are deliberately separated: the App account never receives direct access to the S3 bucket or permanent Data-account credentials.

### Security boundary

The Data account is the protected security boundary. The sensitive document and the KMS key both live there. Belonging to a trusted AWS Organization is not sufficient for access — a specific IAM role and controlled permissions are required to cross the boundary.

### Cross-account access flow

The App account holds the IAM principal `Project1TestUser`. The Data account holds the role `CrossAccountDocumentRole`, whose trust policy permits the appropriate principal from the App account to request assumption.

```text
┌──────────────────────────────┐
│       Don App Account        │
│   Project1TestUser           │
└──────────────┬───────────────┘
               │ sts:AssumeRole
               ▼
┌──────────────────────────────┐
│         AWS STS              │
│ Temporary credentials        │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│        Data Account          │
│ CrossAccountDocumentRole     │
└──────────────┬───────────────┘
               │ s3:GetObject
               ▼
┌──────────────────────────────┐
│       S3 Bucket              │
│ project1-sensitive-data      │
└──────────────┬───────────────┘
               │ SSE-KMS
               ▼
┌──────────────────────────────┐
│       AWS KMS                │
│ Customer-managed KMS key     │
└──────────────────────────────┘
```

### Role separation

Two distinct roles exist in the Data account:

- **`CrossAccountDocumentRole`** — the document-access role. Permissions are limited to `s3:GetObject` and `kms:Decrypt`. It does not administer the KMS key.
- **`Project1KMSAdminRole`** — the KMS administration role. Responsible for administering the customer-managed key, not for consuming encrypted documents.

This implements separation of duties: the principal that administers the encryption control cannot itself decrypt the protected data.

### Design principle

The application never receives permanent Data-account credentials. It authenticates with its own identity and requests temporary credentials via STS by assuming the cross-account role — limiting both the scope and lifetime of the credentials used against the protected data.

**Known hardening gap:** the trust policy on `CrossAccountDocumentRole` currently trusts the App account root rather than the specific `Project1TestUser`/role ARN. The boundary is enforced correctly today only because the App-account-side IAM policy independently scopes who can call `AssumeRole`. Tightening the trust policy to the exact principal ARN is a planned next step, giving the boundary two independent points of enforcement instead of one.

## Implementation & Evidence

| Objective | Status | Evidence |
|---|---|---|
| Bucket not public | Done | Block Public Access enabled on `project1-sensitive-data` |
| Explicit authorisation | Done | Assume-role permission scoped to one role ARN in the App account |
| Temporary STS credentials | Done | `aws sts get-caller-identity` returns `assumed-role/CrossAccountDocumentRole/...` |
| KMS encryption at rest | Done | `get-object` response shows `ServerSideEncryption: aws:kms` with the customer-managed key ARN |
| KMS admin/usage separation | Designed — `Project1KMSAdminRole` vs `CrossAccountDocumentRole` | Not yet screenshot-evidenced (key policy grants not captured) |
| Least-privilege access role | Done | `CrossAccountDocumentRole` confirmed limited to `s3:GetObject` + `kms:Decrypt` |
| End-to-end tested | Done | Assume-role → `s3api get-object` → successful decrypt, twice, via CLI |

## Outcomes & Lessons

- **Deliberately tested both SSE-S3 and SSE-KMS before settling on SSE-KMS**, in order to actually create, use, and manage a customer-managed KMS key end to end — key creation, key policy grants, and the `kms:Decrypt` permission requirement — rather than just enabling encryption as a checkbox. Confirming the object retrieval worked under both modes made the difference between the two concrete: SSE-S3 requires no key management or `kms:Decrypt` permission at all, while SSE-KMS adds an explicit, auditable, revocable control point over decryption.
- **Bucket-level encryption changes aren't retroactive**, confirmed directly as a result of that comparison: re-encrypting the test file from SSE-S3 to SSE-KMS created a new object version but left the original SSE-S3 version intact in the bucket's version history — verified by checking both versions in the console. The older version remains retrievable with `s3:GetObject` alone, without `kms:Decrypt`. Enforcing "KMS-only" access on a versioned bucket in practice requires actively deleting or re-encrypting prior versions, not just changing the encryption setting going forward — otherwise a bucket policy denying `GetObject` on non-KMS-encrypted objects would be needed to close the gap.
- **Trusting an account root in a cross-account trust policy is functional but not defense-in-depth.** It works because the assuming side scopes down independently, but a stricter design trusts the specific principal ARN on both sides of the boundary.
