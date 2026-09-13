# Project 1 — Secure cross-account document storage

Source artifacts for the AWS architecture documented in the [engineering platform](../../engineering-platform) portfolio entry. This repo holds the real policy JSON and reproduction steps; the platform entry holds the narrative and lessons learned.

## Architecture

Three-account AWS Organization:

| Account | ID | Role |
|---|---|---|
| Management | `500481070920` | Org management |
| Don App | `291827353880` | Application identity requesting access |
| Data | `906099689108` | Hosts the S3 bucket and customer-managed KMS key |

`Project1TestUser` (App account) assumes `CrossAccountDocumentRole` (Data account) via STS to retrieve a KMS-encrypted document from `project1-sensitive-data`, without ever holding Data-account credentials.

Two roles enforce separation of duties in the Data account:
- `CrossAccountDocumentRole` — document access only (`s3:GetObject`, `kms:Decrypt`)
- `Project1KMSAdminRole` — key administration only

## Files

- `policies/trust-policy.json` — trust relationship on `CrossAccountDocumentRole`
- `policies/assume-role-policy.json` — scoped assume-role permission on the App-account side (`Project1AssumeDocumentRole`)
- `policies/document-access-role-policy.TODO.json` — placeholder; needs the real saved policy pasted in

## Reproducing the test

```bash
# Assume the cross-account role
aws sts get-caller-identity --profile project1-data-role

# Retrieve and decrypt the object
aws s3api get-object \
  --bucket project1-sensitive-data \
  --key "Project 1 Test file.txt" \
  downloaded.txt \
  --profile project1-data-role
```

A successful response shows `"ServerSideEncryption": "aws:kms"` and the customer-managed key ARN — confirming both `s3:GetObject` and `kms:Decrypt` were required and granted.

## Known gaps / next hardening steps

- Trust policy currently trusts the App account root rather than the specific principal ARN — planned tightening.
- Bucket versioning means pre-KMS object versions (encrypted under SSE-S3) remain retrievable without `kms:Decrypt` — see the platform entry's "Outcomes & Lessons" for the full discussion.
