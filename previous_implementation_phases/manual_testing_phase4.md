# Manual Testing Guide: Phase 4 — Sharing & Permissions

This guide walks you through the manual verification of the Sharing and Permissions system implemented in Phase 4.

## Prerequisites
- The backend server is running (usually at `http://localhost:8000`).
- You have `curl` or a tool like Postman installed.
- (Optional) `jq` installed for nicer JSON formatting.

---

## 1. Environment Setup

We need two users to test sharing: **Alice** (Owner) and **Bob** (Recipient).

### 1.1 Register & Login Alice
```bash
# Register Alice
curl -s -X POST http://localhost:8000/api/v1/auth/register/ \
     -H "Content-Type: application/json" \
     -d '{"email": "alice+v2@vaultshare.io", "password": "password123", "username": "Alice"}'

# Login Alice
ALICE_TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/login/ \
     -H "Content-Type: application/json" \
     -d '{"email": "alice+v2@vaultshare.io", "password": "password123"}' | jq -r '.data.access')
```

### 1.2 Register & Login Bob
```bash
# Register Bob
curl -s -X POST http://localhost:8000/api/v1/auth/register/ \
     -H "Content-Type: application/json" \
     -d '{"email": "bob+v2@vaultshare.io", "password": "password123", "username": "Bob"}'

# Login Bob
BOB_TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/login/ \
     -H "Content-Type: application/json" \
     -d '{"email": "bob+v2@vaultshare.io", "password": "password123"}' | jq -r '.data.access')
```

---

## 2. File Creation (Alice)

Alice uploads a document that she will later share.

```bash
# Alice uploads 'confidential.txt'
# Note: replace /path/to/local_file.txt with a valid local file path
FILE_ID=$(curl -s -X POST http://localhost:8000/api/v1/files/ \
     -H "Authorization: Bearer $ALICE_TOKEN" \
     -F "file=@/Users/antoniopuppedossantos/projects/secjur-vault-share/local_file.txt" \
     -F "name=confidential.txt" | jq -r '.data.id')
```

---

## 3. Sharing & Permission Verification

### 3.1 Initial State (Bob should have no access)
```bash
# Bob tries to download Alice's file - EXPECT 404 (File not found)
curl -v -X GET http://localhost:8000/api/v1/files/$FILE_ID/download/ \
     -H "Authorization: Bearer $BOB_TOKEN"
```

### 3.2 Alice shares with Bob as VIEWER
```bash
# Alice shares as VIEWER
curl -s -X POST http://localhost:8000/api/v1/shares/files/$FILE_ID/ \
     -H "Authorization: Bearer $ALICE_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"recipient_email": "bob+v2@vaultshare.io", "permission": "VIEWER"}'
```

### 3.3 Verify Bob's Access (VIEWER)
```bash
# Bob checks 'Shared with Me' list
curl -s -X GET http://localhost:8000/api/v1/shared-with-me/ \
     -H "Authorization: Bearer $BOB_TOKEN"

# Bob downloads the file - EXPECT 200 SUCCESS
curl -v -X GET http://localhost:8000/api/v1/files/$FILE_ID/download/ \
     -H "Authorization: Bearer $BOB_TOKEN"

# Bob tries to DELETE the file - EXPECT 403 FORBIDDEN
# (Only OWNERS can delete files in this implementation)
curl -v -X DELETE http://localhost:8000/api/v1/files/$FILE_ID/ \
     -H "Authorization: Bearer $BOB_TOKEN"
```

---

## 4. Revocation

Alice decides to remove Bob's access.

```bash
# Bob finds the share_id in his 'Shared with Me' list
SHARE_ID=$(curl -s -X GET http://localhost:8000/api/v1/shared-with-me/ \
     -H "Authorization: Bearer $BOB_TOKEN" | jq -r '.data.files[0].id')

# Alice revokes the share
curl -v -X DELETE "http://localhost:8000/api/v1/shares/$SHARE_ID/?type=file" \
     -H "Authorization: Bearer $ALICE_TOKEN"

# Verify Bob lost access - EXPECT 404
curl -v -X GET http://localhost:8000/api/v1/files/$FILE_ID/download/ \
     -H "Authorization: Bearer $BOB_TOKEN"
```

---

## 5. Audit Log Verification

Alice can verify that all actions on her file were recorded.

```bash
# Alice checks audit logs for her file
curl -s -X GET "http://localhost:8000/api/v1/audit/?target_id=$FILE_ID&target_type=file" \
     -H "Authorization: Bearer $ALICE_TOKEN"
```
Check for entries with:
- `UPLOAD`
- `SHARE`
- `DOWNLOAD`
- `REVOKE`



