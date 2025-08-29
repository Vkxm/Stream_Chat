---
# Stream Chat
*Client (Mobile/Web App)*

* UI: React Native / Flutter
* Chat encryption: libsignal / WebCrypto
* File encryption: AES-256-GCM (client-side)
* Progressive streaming: MediaSource API (Web) + native player (Mobile)
* WebRTC: P2P calls/stream sync

*Backend (Supabase / Node.js + Postgres)*

* Auth service (JWT + refresh tokens)
* Chatroom metadata store (rooms, members, keys)
* File metadata store (filename, encrypted key, expiry, storage path)
* Pre-signed short TTL links for downloads
* Admin dashboard (reports, takedowns)

*Storage Layer*

* S3-compatible storage (Wasabi/Backblaze/S3)
* File chunks encrypted client-side before upload

*Streaming Infra*

* WebRTC P2P mesh (small groups ≤4 users)
* TURN server (Coturn) for NAT traversal
* Optional SFU (mediasoup/LiveKit) for larger groups

---

## 2. *Data Flow*

### (a) *Chat Message Flow*

1. User types message → encrypts with Signal protocol.
2. Message sent to Supabase Realtime.
3. Recipient client pulls message → decrypts using session key.

### (b) *File Upload & Share Flow*

1. User selects file → client generates AES key.
2. Encrypt file chunks locally.
3. Upload encrypted chunks → S3 storage.
4. AES key encrypted with recipient’s public key → stored in DB.
5. Recipient fetches file → decrypts key with private key → decrypts chunks → plays.

### (c) *Streaming Flow*

1. Uploader sends encrypted chunks to storage while upload continues.
2. Recipient fetches progressive chunks (pre-signed URL).
3. Decrypt + buffer → playback.
4. For co-watch → WebRTC data channel syncs playback position.

---

## 3. *Database Schema (Postgres / Supabase)*

### users

sql
id UUID (PK)  
email TEXT UNIQUE  
public_key TEXT  
created_at TIMESTAMP  


### chatrooms

sql
id UUID (PK)  
name TEXT  
owner_id UUID (FK → users.id)  
created_at TIMESTAMP  


### chatroom_members

sql
id UUID (PK)  
chatroom_id UUID (FK → chatrooms.id)  
user_id UUID (FK → users.id)  
role TEXT (admin/member)  
joined_at TIMESTAMP  


### messages

sql
id UUID (PK)  
chatroom_id UUID (FK → chatrooms.id)  
sender_id UUID (FK → users.id)  
cipher_text TEXT  
created_at TIMESTAMP  


### files

sql
id UUID (PK)  
uploader_id UUID (FK → users.id)  
chatroom_id UUID (FK → chatrooms.id)  
file_name TEXT  
storage_path TEXT  
encrypted_key TEXT  
expiry TIMESTAMP  
created_at TIMESTAMP  


---

## 4. *API Endpoints (REST/GraphQL)*

### *Auth*

* POST /auth/register → create user
* POST /auth/login → login & JWT
* GET /auth/me → current user

### *Chatrooms*

* POST /chatrooms → create chatroom
* POST /chatrooms/:id/join → join room
* GET /chatrooms/:id/messages → fetch messages
* POST /chatrooms/:id/message → send encrypted message

### *Files*

* POST /files/upload → request pre-signed URL + upload metadata
* GET /files/:id → fetch file metadata + pre-signed download link
* DELETE /files/:id → delete file

### *Streaming (WebRTC)*

* POST /webrtc/offer → exchange SDP offer
* POST /webrtc/answer → exchange SDP answer
* POST /webrtc/candidate → ICE candidate exchange

### *Admin / Reports*

* POST /report → report file/message
* GET /admin/reports → view reports
* DELETE /admin/files/:id → force delete file

---

## 5. *Encryption Details*

* *Chat*: Signal Protocol (X3DH + Double Ratchet).
* *Files*: AES-256-GCM (per-file Data Encryption Key).
* *Key Wrapping*: DEK encrypted with recipient’s X25519 public key.
* *Transport*: HTTPS + TLS 1.3.
* *Streaming (WebRTC)*: DTLS-SRTP + Insertable Streams for E2E.

---

## 6. *Tech Stack*

* *Frontend:* React Native (Expo) / Flutter
* *Backend:* Node.js (NestJS/Express) OR Supabase (faster MVP)
* *DB:* Postgres (Supabase)
* *Storage:* S3-compatible (Wasabi/Backblaze)
* *Streaming:* WebRTC (mediasoup / LiveKit for SFU)
* *TURN/STUN:* Coturn server
* *Encryption:* libsignal, WebCrypto, Insertable Streams

---

## 7. *DevOps & Deployment*

* *Infra*: Docker + Kubernetes (later scale)
* *CI/CD*: GitHub Actions → Deploy to AWS / GCP / Fly.io
* *Monitoring*: Prometheus + Grafana
* *Error tracking*: Sentry
* *Logs*: Encrypted logs for debugging without exposing user data

---

## 8. *Developer To-Do (MVP)*

* [ ] Auth system (JWT + refresh)
* [ ] Signal protocol integration for chat
* [ ] File encryption + pre-signed upload/download
* [ ] Progressive decryption + playback
* [ ] WebRTC P2P chatroom sync (small groups)
* [ ] Basic admin dashboard

---