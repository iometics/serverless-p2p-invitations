# Self-Contained Peer-to-Peer Connection Invitations: A Serverless Approach to Network Establishment

**Version:** 1.1  
**Date:** February 2, 2026  
**Status:** Public Domain - Defensive Publication  
**License:** CC0 1.0 Universal

---

## Abstract

This specification describes a method for establishing direct peer-to-peer network connections without requiring centralized coordination, signaling, or rendezvous servers. Unlike traditional WebRTC or P2P architectures that depend on server-mediated signaling for Session Description Protocol (SDP) exchange, this approach embeds complete connection establishment parameters within self-contained, cryptographically secure invitation tokens. These tokens can be transmitted through any out-of-band channel (email, SMS, instant messaging, QR codes, NFC, etc.) and contain all information necessary for peers to discover, authenticate, and establish encrypted direct connections.

The protocol uses a progressive enhancement strategy: connections begin with a simple single-token invitation that works for most NAT configurations. When symmetric NAT or other challenging network conditions prevent direct connection, the system automatically escalates to a bidirectional token exchange (RSVP response), ensuring maximum compatibility while maintaining simplicity for common cases.

The key innovation is the elimination of the signaling server dependency while maintaining security, NAT traversal capability, and ease of use. This enables truly decentralized peer-to-peer applications for small ad-hoc networks where participants can establish secure connections by simply sharing a token through their preferred communication channel.

**Keywords:** peer-to-peer networking, serverless architecture, NAT traversal, connection invitation, decentralized systems, WebRTC, cryptographic protocols, symmetric NAT, hole punching, ICE

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Architecture](#2-architecture)
3. [Invitation Token Format](#3-invitation-token-format)
4. [Connection Protocol](#4-connection-protocol)
5. [RSVP Response Mechanism](#5-rsvp-response-mechanism)
6. [Security Considerations](#6-security-considerations)
7. [Implementation Guidance](#7-implementation-guidance)
8. [Comparison to Existing Approaches](#8-comparison-to-existing-approaches)
9. [Use Cases](#9-use-cases)
10. [Extensions and Variations](#10-extensions-and-variations)
11. [Standards and Interoperability](#11-standards-and-interoperability)
12. [Future Work](#12-future-work)
13. [Conclusion](#13-conclusion)
14. [Appendices](#appendices)

---

## 1. Introduction

### 1.1 Problem Statement

Traditional peer-to-peer connection establishment protocols face several challenges:

1. **Signaling Server Dependency**: WebRTC and similar protocols require a signaling server to exchange SDP offers and answers between peers. This creates:
   - Single point of failure
   - Privacy concerns (server sees connection metadata)
   - Operational overhead (server maintenance, scaling, costs)
   - Trust requirements (server operator must be trusted)

2. **Complex Setup**: Users must either:
   - Use a hosted service with account creation
   - Deploy and maintain their own signaling infrastructure
   - Configure firewall rules and port forwarding

3. **Discoverability Problem**: Peers behind NAT or firewalls cannot easily find each other without intermediary infrastructure

4. **Bootstrap Challenge**: How do two peers who want to communicate discover each other's network location in the first place?

5. **Symmetric NAT Traversal**: Many networks (mobile carriers, universities, corporate environments) use symmetric NAT which requires both peers to send packets simultaneously—impossible with asymmetric signaling approaches

### 1.2 Solution Overview

This specification presents a serverless approach with progressive enhancement:

**Phase 1: Simple Invitation (Most Common Case)**

1. **Initiating peer** generates a self-contained invitation token containing:
   - Network addressing information (IP addresses, ports, ICE candidates)
   - Cryptographic key material for secure connection establishment
   - Connection metadata and protocol parameters
   - ICE role assignment and tie-breaker value
   - Optional relay server information (if fallback is desired)

2. **Token distribution** happens through any out-of-band channel:
   - Email, SMS, instant messaging
   - QR code (visual encoding)
   - NFC tap (proximity-based)
   - Voice (spoken codes)
   - Platform-native share dialogs

3. **Joining peer** imports the token and attempts connection:
   - Sends ICE connectivity checks (STUN Binding Requests) to provided addresses
   - Performs NAT traversal using included ICE candidates
   - Attempts to establish connection with assigned ICE role

**Phase 2: RSVP Response (Symmetric NAT Fallback)**

4. If initial connection attempts fail (indicating symmetric NAT or other restrictive network):
   - System prompts invitee to send RSVP response token
   - RSVP contains invitee's connection information and complementary ICE role
   - Both peers now have each other's candidates
   - Simultaneous connection attempts succeed through NAT hole punching
   - ICE role conflict resolution ensures deterministic behavior

This progressive approach provides:
- **Simplicity** for common cases (one-way token share)
- **Reliability** for challenging networks (automatic escalation to RSVP)
- **User control** (explicit prompt before RSVP)
- **Correctness** (ICE role management prevents deadlocks)
- **Best of both worlds** (simple when possible, robust when necessary)

### 1.3 Relationship to SDP Offer/Answer Model

This specification adopts terminology optimized for user understanding while maintaining semantic alignment with established protocols:

**Token Type Mapping:**
- **Invitation token** ≅ SDP Offer (RFC 3264)
- **RSVP token** ≅ SDP Answer (RFC 3264)

The invitation/RSVP terminology provides clarity for end users while the underlying semantics follow the well-established offer/answer exchange pattern. Implementers familiar with WebRTC will recognize:
- Invitation = createOffer()
- RSVP = createAnswer()
- Both contain ICE candidates and cryptographic parameters

This alignment ensures the protocol fits naturally into existing ICE/SDP mental models while using more accessible language in user-facing contexts.

### 1.4 Key Advantages

- **Zero Server Infrastructure**: No signaling or coordination servers required
- **Privacy-Preserving**: Connection parameters never transit through third-party servers
- **Platform Agnostic**: Works with any token transmission mechanism
- **Progressive Enhancement**: Simple path first, complex path only when needed
- **Symmetric NAT Support**: Handles most challenging network configurations
- **ICE Compliance**: Proper role management and conflict resolution per RFC 8445
- **Simple User Experience**: Share once for easy networks, RSVP for hard networks
- **Truly Decentralized**: No central authority or coordination point
- **Cost-Free Operation**: No server hosting or maintenance costs

---

## 2. Architecture

### 2.1 System Components

```
┌─────────────────────────────────────────────────────────────┐
│                    INVITATION TOKEN                         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ • Token Type: "invitation"                           │  │
│  │ • Network Addresses (IPv4/IPv6)                      │  │
│  │ • ICE Candidates (local, reflexive, relay)           │  │
│  │ • ICE Role: "controlling"                            │  │
│  │ • ICE Tie-Breaker: uint64                            │  │
│  │ • Public Key / Key Exchange Material                 │  │
│  │ • Protocol Version & Capabilities                    │  │
│  │ • Token ID & Expiration                              │  │
│  │ • Connection Mode (single-token / rsvp-required)     │  │
│  │ • Optional: Relay Server Endpoints                   │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │  Out-of-Band Channel │
              │  (Email, SMS, QR,    │
              │   Messaging, etc.)   │
              └──────────────────────┘
                         │
                         ▼
                  ┌──────────────┐
                  │  Peer B      │
                  │  (Invitee)   │
                  └──────────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │  ICE Connectivity    │
              │  Checks (STUN)       │
              └──────────────────────┘
                         │
           ┌─────────────┴─────────────┐
           │                           │
      SUCCESS                       FAILURE
           │                           │
           ▼                           ▼
    ┌──────────────┐         ┌──────────────────┐
    │  Connection  │         │  RSVP Prompt     │
    │  Established │         │  "Need response" │
    └──────────────┘         └──────────────────┘
                                     │
                                     ▼
                          ┌─────────────────────┐
                          │  RSVP TOKEN         │
                          │  Type: "rsvp"       │
                          │  Role: "controlled" │
                          │  (Invitee → Inviter)│
                          └─────────────────────┘
                                     │
                                     ▼
                          ┌─────────────────────┐
                          │  Simultaneous       │
                          │  ICE Checks         │
                          │  (Both directions)  │
                          │  Role conflict      │
                          │  resolution if      │
                          │  needed             │
                          └─────────────────────┘
                                     │
                                     ▼
                              ┌──────────────┐
                              │  Connection  │
                              │  Established │
                              └──────────────┘
```

### 2.2 Connection Establishment Flow

#### 2.2.1 Phase 1: Initial Invitation (Single-Token Mode)

```
1. TOKEN GENERATION
   - Peer A (Inviter) initiates network setup
   - System gathers local network information
   - Queries STUN servers for reflexive addresses
   - Generates cryptographic key material
   - Assigns ICE role: "controlling"
   - Generates random 64-bit tie-breaker value
   - Packages into structured token
   - Encodes for transmission

2. TOKEN DISTRIBUTION
   - User selects sharing mechanism
   - Token transmitted through out-of-band channel
   - Recipient receives token

3. CONNECTION ATTEMPT (Invitee-Initiated)
   - Peer B (Invitee) imports token
   - Validates token structure and signature
   - Extracts connection candidates
   - Notes inviter's ICE role (controlling)
   - Assigns own ICE role: "controlled"
   - Begins sending ICE connectivity checks to Peer A
   - Attempts connection in priority order:
     a. Direct via local addresses (same network)
     b. Direct via reflexive addresses (NAT traversal)
     c. Via relay addresses (if provided)
   
4. OUTCOME DETERMINATION
   - If connection succeeds (most common):
     → Proceed to key exchange
     → Establish encrypted session
     → Done
   
   - If connection fails after timeout (10-15 seconds):
     → Proceed to Phase 2 (RSVP)
```

#### 2.2.2 Phase 2: RSVP Response (Bidirectional Mode)

```
5. RSVP PROMPT
   - System detects failed connection attempts
   - Prompts user: "Connection needs response from inviter"
   - User confirms to send RSVP

6. RSVP TOKEN GENERATION
   - Peer B generates response token
   - Token type: "rsvp"
   - Contains Peer B's connection candidates
   - ICE role: "controlled" (complementary to inviter)
   - Generates own tie-breaker value
   - Links to original invitation (in_reply_to)
   - Shares back to Peer A via OOB channel

7. BIDIRECTIONAL CONNECTION
   - Peer A receives and processes RSVP
   - Both peers now have each other's candidates
   - Both peers perform simultaneous ICE connectivity checks
   - ICE role conflict resolution (if both send simultaneously)
   - Packets from both sides punch through NAT
   - First successful candidate pair wins

8. SESSION ESTABLISHMENT
   - Cryptographic handshake
   - Establish encrypted data channel
   - Connection complete
```

### 2.3 Decision Flow Diagram

```
┌─────────────────────────────────────────┐
│  Inviter generates invitation token     │
│  Type: "invitation"                     │
│  Mode: single-token (try simple first)  │
│  ICE Role: "controlling"                │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│  Inviter shares token via OOB channel   │
│  (Email, SMS, QR, messaging, etc.)      │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│  Invitee receives and validates token   │
│  Assigns self ICE role: "controlled"    │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│  Invitee begins ICE connectivity checks │
│  Sends STUN requests to inviter         │
│  Timeout: 10-15 seconds                 │
└────────────────┬────────────────────────┘
                 │
         ┌───────┴────────┐
         ▼                ▼
    CONNECTION        CONNECTION
      SUCCEEDS          FAILS
         │                │
         ▼                ▼
   ┌─────────┐    ┌──────────────────┐
   │ SUCCESS │    │ Prompt for RSVP  │
   │  Done!  │    │ "Send response?" │
   └─────────┘    └────────┬─────────┘
                           │
                    ┌──────┴──────┐
                    ▼             ▼
                  YES            NO
                    │             │
                    ▼             ▼
         ┌──────────────┐   ┌─────────┐
         │ Generate RSVP│   │ Abandon │
         │ Type: "rsvp" │   └─────────┘
         │ Role:        │
         │ "controlled" │
         │ Share to     │
         │ inviter      │
         └──────┬───────┘
                │
                ▼
         ┌──────────────────────┐
         │ Inviter receives RSVP│
         │ Both have ICE roles  │
         │ Both attempt checks  │
         │ simultaneously       │
         └──────┬───────────────┘
                │
                ▼
         ┌──────────────┐
         │ NAT hole     │
         │ punching +   │
         │ role conflict│
         │ resolution   │
         └──────┬───────┘
                │
                ▼
         ┌──────────────┐
         │   SUCCESS    │
         │    Done!     │
         └──────────────┘
```

---

## 3. Invitation Token Format

### 3.1 Token Structure

The invitation token is a structured data object containing all information necessary for connection establishment. The canonical representation is JSON, though implementations may use alternative encodings.

```json
{
  "version": "1.1",
  "token_type": "invitation",
  "token_id": "550e8400-e29b-41d4-a716-446655440000",
  "created_at": "2026-02-02T10:30:00Z",
  "expires_at": "2026-02-03T10:30:00Z",
  "network_id": "my-app-network-xyz",
  
  "connection_mode": {
    "initial": "single-token",
    "rsvp_supported": true,
    "rsvp_timeout_seconds": 15
  },
  
  "initiator": {
    "peer_id": "peer-a-unique-identifier",
    "public_key": "base64-encoded-public-key",
    "capabilities": ["data-channel", "audio", "video"]
  },
  
  "connection_info": {
    "ice_candidates": [
      {
        "type": "host",
        "protocol": "udp",
        "address": "192.168.1.100",
        "port": 54321,
        "priority": 2130706431
      },
      {
        "type": "srflx",
        "protocol": "udp", 
        "address": "203.0.113.45",
        "port": 54321,
        "related_address": "192.168.1.100",
        "related_port": 54321,
        "priority": 1694498815
      }
    ],
    "ufrag": "ice-username-fragment",
    "pwd": "ice-password",
    "ice_role": "controlling",
    "ice_tie_breaker": "1827364950273645"
  },
  
  "crypto": {
    "key_exchange": "ecdh-curve25519",
    "cipher": "chacha20-poly1305",
    "fingerprint": "sha-256 AB:CD:EF:...",
    "dtls_setup": "actpass"
  },
  
  "relay": {
    "enabled": true,
    "servers": [
      {
        "urls": ["turn:relay.example.com:3478"],
        "username": "temporary-credentials",
        "credential": "temporary-password",
        "credential_type": "password"
      }
    ]
  },
  
  "metadata": {
    "app_id": "my-application",
    "protocol_version": "1.0",
    "description": "Connection to project workspace"
  },
  
  "signature": "base64-encoded-signature-of-above-fields"
}
```

### 3.2 Token Fields

#### 3.2.1 Core Fields

- **version**: Protocol version (semantic versioning). Current: "1.1"
- **token_type**: Token classification ("invitation" or "rsvp")
  - This field MUST be present in all tokens
  - Value "invitation" indicates this is an offer (SDP terminology)
  - Value "rsvp" indicates this is an answer (SDP terminology)
- **token_id**: Unique identifier (UUID v4 recommended)
- **created_at**: ISO 8601 timestamp of token creation
- **expires_at**: ISO 8601 timestamp when token becomes invalid
- **network_id**: Identifier for logical network/room (application-specific)

#### 3.2.2 Connection Mode Configuration

The `connection_mode` object controls the progressive enhancement behavior:

- **initial**: Starting mode ("single-token" or "bidirectional")
  - `"single-token"`: Try invitee-initiated connection first
  - `"bidirectional"`: Require RSVP from the start
  
- **rsvp_supported**: Whether RSVP fallback is available (boolean)
  - `true`: Can escalate to RSVP if needed
  - `false`: Single-token only (will fail on symmetric NAT)

- **rsvp_timeout_seconds**: How long to wait before prompting for RSVP
  - Typical: 10-15 seconds
  - Shorter for real-time apps, longer for background transfers

**Recommended defaults:**
```json
{
  "initial": "single-token",
  "rsvp_supported": true,
  "rsvp_timeout_seconds": 12
}
```

#### 3.2.3 Initiator Information

- **peer_id**: Unique identifier for the inviting peer
- **public_key**: Public key for key exchange (format depends on crypto.key_exchange)
- **capabilities**: Array of supported features/protocols

#### 3.2.4 Connection Information

The `connection_info` object contains ICE-like candidate information:

**ICE Candidates**: Array of network addresses where peer can be reached
- `type`: Candidate type (host, srflx, relay)
  - `host`: Local address on peer's network
  - `srflx`: Server reflexive address (from STUN)
  - `relay`: Relayed address (from TURN)
- `protocol`: Transport protocol (udp, tcp)
- `address`: IP address (IPv4 or IPv6)
- `port`: Port number
- `priority`: Candidate priority for selection algorithm (per RFC 8445)
- `related_address/related_port`: Base address for derived candidates

**ICE Parameters**:
- `ufrag`: ICE username fragment for connectivity checks
- `pwd`: ICE password for connectivity checks
- `ice_role`: ICE agent role ("controlling" or "controlled")
  - Invitation tokens: MUST be "controlling"
  - RSVP tokens: MUST be "controlled"
  - This prevents ambiguity and establishes clear semantics
- `ice_tie_breaker`: Random 64-bit unsigned integer (as string)
  - Used for role conflict resolution per RFC 8445 §7.3.1.1
  - MUST be randomly generated with cryptographic randomness
  - Format: Decimal string representation (e.g., "1827364950273645")

**ICE Role Assignment Rules:**
```
Invitation token:  ice_role = "controlling"
RSVP token:        ice_role = "controlled"

This asymmetry ensures:
- Clear role assignment in single-token mode
- Complementary roles in bidirectional mode
- Deterministic conflict resolution via tie-breaker
```

#### 3.2.5 Cryptographic Parameters

- **key_exchange**: Key exchange algorithm
  - Options: "ecdh-curve25519", "ecdh-p256", "x25519"
- **cipher**: Symmetric cipher for data encryption
  - Options: "chacha20-poly1305", "aes-256-gcm"
- **fingerprint**: Hash of certificate/key for verification
- **dtls_setup**: DTLS role (active, passive, actpass)

#### 3.2.6 Relay Configuration (Optional)

- **enabled**: Whether relay fallback is supported
- **servers**: Array of TURN/relay server configurations
  - Similar to WebRTC's RTCIceServer format

#### 3.2.7 Metadata

- **app_id**: Application identifier
- **protocol_version**: Application protocol version
- **description**: Human-readable description
- Additional application-specific fields

#### 3.2.8 Signature

Cryptographic signature over all preceding fields to ensure integrity and authenticity.

### 3.3 RSVP Token Format

When connection fails and RSVP is needed, the invitee generates a response token with the same structure, with these key differences:

```json
{
  "version": "1.1",
  "token_type": "rsvp",
  "token_id": "rsvp-response-uuid",
  "in_reply_to": "550e8400-e29b-41d4-a716-446655440000",
  "created_at": "2026-02-02T10:30:15Z",
  "expires_at": "2026-02-02T11:30:15Z",
  
  "connection_mode": {
    "initial": "bidirectional",
    "rsvp_supported": false
  },
  
  "responder": {
    "peer_id": "peer-b-unique-identifier",
    "public_key": "peer-b-public-key",
    "capabilities": ["data-channel"]
  },
  
  "connection_info": {
    "ice_candidates": [
      // Peer B's candidates
    ],
    "ufrag": "peer-b-ufrag",
    "pwd": "peer-b-password",
    "ice_role": "controlled",
    "ice_tie_breaker": "7364829104738291"
  },
  
  "crypto": {
    // Peer B's crypto parameters
  },
  
  "metadata": {
    "app_id": "my-application",
    "protocol_version": "1.0",
    "description": "RSVP to Alice's invitation"
  },
  
  "signature": "rsvp-token-signature"
}
```

**Key differences in RSVP tokens:**
- **token_type**: Set to "rsvp" (indicates this is an answer)
- **in_reply_to**: Links to original invitation token_id (replaces "response_to")
- **responder**: Contains invitee's peer information (replaces "initiator")
  - Clarifies that this peer is responding, not initiating the session
  - Original initiator is still Peer A from invitation token
- **expires_at**: Shorter expiration (typically 1 hour vs 24 hours for invitations)
- **connection_mode.initial**: Set to "bidirectional"
- **rsvp_supported**: Set to false (no further escalation)
- **ice_role**: Set to "controlled" (complementary to invitation's "controlling")
- **ice_tie_breaker**: New random value for this peer

### 3.4 Offer/Answer Semantics

**Mapping to SDP Terminology:**

This specification uses user-friendly terminology while maintaining semantic equivalence to established protocols:

| This Spec | SDP/ICE Term | Description |
|-----------|--------------|-------------|
| Invitation | Offer | First token, proposes connection |
| RSVP | Answer | Response token, accepts connection |
| initiator | Offerer | Peer creating invitation |
| responder | Answerer | Peer creating RSVP |
| ice_role: "controlling" | ICE controlling | Role in invitation |
| ice_role: "controlled" | ICE controlled | Role in RSVP |

Implementers familiar with WebRTC will recognize:
```javascript
// Invitation ≅ 
const offer = await peerConnection.createOffer();

// RSVP ≅
const answer = await peerConnection.createAnswer();
```

This alignment ensures the protocol integrates naturally with existing ICE/SDP implementations while providing clearer semantics for end users and application developers.

### 3.5 Token Encoding

#### 3.5.1 Text Format (Default)

For transmission via text-based channels (email, SMS, messaging):

1. Serialize token to JSON (minified, no whitespace)
2. Compress with DEFLATE or similar
3. Encode as Base64URL (URL-safe, no padding)
4. Optionally prefix with protocol identifier: `scpi://` or similar
5. Result: Compact string suitable for copying/pasting

**Example**:
```
scpi://eJyNVMtu2zAQ_JWFT02BlCzJsg-9tECRAkWRQ4EeihxWJNcSQpIqSdlqg_x7lyTlOE...
```

#### 3.5.2 QR Code Format

For visual transmission:

1. Use text format above
2. Encode as QR code (recommended: medium error correction)
3. QR code can be scanned by mobile devices
4. Size automatically adjusts to data volume

#### 3.5.3 Binary Format (Compact)

For size-constrained channels (NFC, audio, etc.):

1. Use binary encoding (Protocol Buffers, CBOR, or MessagePack)
2. Further compression optional
3. Base85 or Base91 encoding for denser representation

### 3.6 Token Security

#### 3.6.1 Confidentiality

Tokens may contain sensitive information:
- Internal network addresses (privacy concern)
- Temporary credentials (security concern)
- Network topology information

**Recommendations**:
- Treat tokens as sensitive data
- Use encrypted channels when possible (HTTPS, encrypted messaging)
- Implement token expiration
- Use single-use tokens when feasible

#### 3.6.2 Authenticity

The `signature` field prevents tampering:
- Computed over all token fields
- Uses initiator's private key (invitation) or responder's private key (RSVP)
- Verified by recipient using public_key field
- Prevents man-in-the-middle attacks during token transit

**Signature Algorithm**:
```
signature = Sign(private_key, Hash(canonical_json(token_fields)))
```

Where:
- `Hash`: SHA-256 or SHA-3
- `Sign`: Ed25519, ECDSA-P256, or similar
- `canonical_json`: Deterministic JSON serialization

#### 3.6.3 Token Lifetime

- `expires_at` enforces time-limited validity
- Recommended: 24-48 hours for invitation tokens
- Recommended: 1 hour for RSVP tokens
- After expiration, token is rejected during validation

---

## 4. Connection Protocol

### 4.1 Single-Token Connection Establishment

#### 4.1.1 Invitee-Side Process (Initial Attempt)

When receiving and processing an invitation token:

```
1. PARSE token from encoded format
2. VALIDATE token structure and signature
3. CHECK token_type == "invitation"
4. CHECK expiration (reject if expired)
5. CHECK connection mode
   - If mode.initial == "bidirectional": Skip to RSVP generation
   - If mode.initial == "single-token": Continue with single-token attempt
6. EXTRACT connection candidates
7. EXTRACT inviter's ice_role (should be "controlling")
8. ASSIGN own ice_role = "controlled"
9. GENERATE own ice_tie_breaker value
10. SORT candidates by priority
11. START timeout timer (mode.rsvp_timeout_seconds)
12. FOR EACH candidate (highest priority first):
     a. ATTEMPT connection
     b. SEND ICE connectivity check (STUN Binding Request) to inviter
     c. WAIT for response
     d. IF successful: PROCEED to key exchange
     e. IF failed: TRY next candidate
13. IF all candidates fail AND timeout expires:
     a. IF mode.rsvp_supported: PROMPT for RSVP
     b. ELSE: REPORT connection failure
```

#### 4.1.2 ICE Connectivity Check (Invitee → Inviter)

The invitee sends STUN Binding Requests to attempt NAT traversal:

```
Invitee → Inviter: STUN Binding Request
  - Destination: Each candidate address from token
  - Transaction ID: Random 96-bit value
  - USERNAME = concat(inviter_ufrag, ":", local_ufrag)
  - MESSAGE-INTEGRITY = HMAC-SHA1(inviter_pwd, request)
  - ICE-CONTROLLING or ICE-CONTROLLED attribute (RFC 8445 §7.3)
    → Invitee sends ICE-CONTROLLED with own tie-breaker
  
Inviter → Invitee: STUN Binding Response (if received)
  - Transaction ID (matching request)
  - XOR-MAPPED-ADDRESS (invitee's reflexive address)
  - MESSAGE-INTEGRITY = HMAC-SHA1(inviter_pwd, response)
  
If response received:
  - Connectivity confirmed
  - Proceed to key exchange
  - Success!
  
If no response after retry (3-5 attempts over 5 seconds):
  - Try next candidate
  - If all fail and timeout expires: Escalate to RSVP
```

**Why this works for most NATs:**
- **Full Cone NAT**: Inviter can receive from any source after binding
- **Restricted Cone NAT**: Works if inviter has sent to public internet (STUN query)
- **Port-Restricted Cone NAT**: Same as restricted cone
- **Does NOT work for Symmetric NAT**: Requires bidirectional flow (RSVP)

#### 4.1.3 Initiator-Side Behavior (Passive Listening)

The inviter (Peer A) remains passive during single-token phase:

```
1. After sharing invitation token
2. LISTEN on all local candidate addresses
3. WAIT for incoming ICE connectivity checks
4. VALIDATE incoming STUN Binding Requests:
   - Check USERNAME matches format
   - Verify MESSAGE-INTEGRITY with local pwd
   - Check ICE-CONTROLLED attribute (peer should be controlled)
5. RESPOND to valid requests with STUN Binding Response
6. PROCEED to key exchange when valid check received
```

**Note on NAT Binding Expiry:**

In some network configurations, NAT bindings may expire (typically 30-120 seconds after the last outbound packet) if no traffic is sent. This can cause Phase 1 to fail even with correct ICE connectivity checks from the invitee.

**Why this matters:**
- The inviter may have established a STUN binding when gathering candidates
- If the invitee's connection attempt arrives after this binding expires, the NAT will block it
- The NAT has no record of an outbound connection to the invitee's source

**Example scenario:**
```
T=0s:   Inviter queries STUN, NAT creates binding (expires at T=120s)
T=1s:   Inviter generates token with reflexive candidate
T=60s:  Invitee receives token, begins connection attempts
T=121s: NAT binding expired, invitee's packets blocked
Result: Connection fails despite correct protocol
```

The RSVP mechanism resolves this by having both peers send simultaneously, continuously refreshing NAT bindings on both sides. This is a key advantage of the bidirectional RSVP fallback: it works even when NAT state has expired.

**Recommendation for implementations:**
- Consider sending periodic STUN keepalives (every 15-30s) while waiting for connection
- This keeps NAT bindings fresh during the waiting period
- Tradeoff: Small bandwidth cost vs improved single-token success rate

#### 4.1.4 Key Exchange (After Connectivity Established)

Once connectivity is confirmed through ICE checks, peers establish shared encryption key:

**ECDH Key Exchange** (recommended):

```
1. Invitee generates ephemeral key pair (public_B, private_B)

2. Invitee → Initiator:
   {
     "type": "key_exchange",
     "public_key": public_B,
     "peer_id": invitee_peer_id
   }

3. Initiator:
   - Verifies invitee's message
   - Computes: shared_secret = ECDH(private_A, public_B)
   - Derives: session_key = HKDF(shared_secret, salt, info)

4. Invitee:
   - Receives initiator's public key from token
   - Computes: shared_secret = ECDH(private_B, public_A)
   - Derives: session_key = HKDF(shared_secret, salt, info)

5. Both peers now share session_key
```

**Key Derivation**:
```
session_key = HKDF-SHA256(
  input_key = shared_secret,
  salt = concat(ufrag_A, ufrag_B),
  info = "scpi-session-v1",
  output_length = 32 bytes
)
```

#### 4.1.5 Authenticated Encryption

Once session key is established, all application data is encrypted:

```
For each message:
  nonce = counter || random_bytes
  ciphertext = Encrypt(session_key, nonce, plaintext)
  tag = MAC(session_key, nonce || ciphertext)
  
  Send: nonce || ciphertext || tag
```

Recommended ciphers:
- ChaCha20-Poly1305 (preferred for performance)
- AES-256-GCM (hardware acceleration available)

---

## 5. RSVP Response Mechanism

### 5.1 RSVP Trigger Conditions

The system prompts for RSVP when single-token connection fails:

```
RSVP Required When:
  - All connection candidates attempted
  - All ICE connectivity checks failed (no STUN responses)
  - Timeout period elapsed (rsvp_timeout_seconds)
  - Token indicates rsvp_supported == true
  
RSVP Not Available When:
  - Token indicates rsvp_supported == false
  - Token type is already "rsvp"
  - Token expired
```

### 5.2 User Prompt Design

**Recommended user messaging:**

```
Simple Case (Connection Succeeded):
  ✓ "Connected to Alice's network"
  [No user action required]

RSVP Case (Connection Failed):
  ⚠ "Connection needs additional step"
  
  "Due to your network configuration, Alice needs 
   connection information from you to complete setup.
   
   Send response to Alice?"
   
   [Send Response]  [Cancel]
```

**Key messaging principles:**
- Don't use technical jargon (NAT, symmetric, ICE, etc.)
- Explain why (network configuration)
- Make it optional (user can cancel)
- Make it easy (one button click)

### 5.3 RSVP Token Generation

When user approves RSVP:

```python
def generate_rsvp_token(original_invitation):
    """Generate RSVP response to original invitation"""
    
    # Generate new keypair for this peer
    private_key, public_key = generate_key_pair()
    
    # Gather local network candidates
    candidates = gather_ice_candidates()
    
    # Create RSVP token
    rsvp = {
        'version': '1.1',
        'token_type': 'rsvp',  # Explicitly mark as answer
        'token_id': generate_uuid(),
        'in_reply_to': original_invitation['token_id'],  # Link to invitation
        'created_at': datetime.utcnow().isoformat() + 'Z',
        'expires_at': (datetime.utcnow() + timedelta(hours=1)).isoformat() + 'Z',
        
        'connection_mode': {
            'initial': 'bidirectional',
            'rsvp_supported': False  # No further escalation
        },
        
        'responder': {  # Note: 'responder' not 'initiator'
            'peer_id': generate_peer_id(),
            'public_key': base64_encode(public_key),
            'capabilities': get_capabilities()
        },
        
        'connection_info': {
            'ice_candidates': candidates,
            'ufrag': generate_random_string(8),
            'pwd': generate_random_string(24),
            'ice_role': 'controlled',  # Complementary to invitation's 'controlling'
            'ice_tie_breaker': str(random.randint(0, 2**64-1))  # Own tie-breaker
        },
        
        'crypto': {
            'key_exchange': 'ecdh-curve25519',
            'cipher': 'chacha20-poly1305',
            'fingerprint': compute_fingerprint(public_key)
        },
        
        'metadata': {
            'app_id': original_invitation['metadata']['app_id'],
            'protocol_version': original_invitation['metadata']['protocol_version'],
            'description': f"RSVP to {original_invitation['initiator']['peer_id']}"
        }
    }
    
    # Sign RSVP
    canonical = canonicalize_json(rsvp)
    signature = sign(private_key, sha256(canonical))
    rsvp['signature'] = base64_encode(signature)
    
    return encode_token(rsvp), private_key
```

### 5.4 RSVP Distribution

The RSVP token is shared back to the original inviter:

```
Methods (same as original invitation):
  - Reply to email/message thread
  - Share via messaging app
  - Display QR code for scanning
  - NFC tap (second tap)
  - Automatic via app (if both users online)
```

**Automatic RSVP flow** (best UX):

```
1. Invitee app detects need for RSVP
2. Prompts user for approval
3. On approval:
   - Generates RSVP token
   - Automatically sends via same channel as invitation
   - Example: Reply message in chat app
4. Inviter app receives RSVP
5. Auto-imports and begins bidirectional connection
6. Users see "Connecting..." on both sides
7. Connection established
```

### 5.5 Bidirectional Connection Establishment

Once both peers have each other's tokens, they perform simultaneous connection:

#### 5.5.1 Initiator Process (After Receiving RSVP)

```
Peer A (Original Inviter):

1. RECEIVE RSVP token via OOB channel
2. VALIDATE RSVP:
   - Check token_type == "rsvp"
   - Verify in_reply_to matches original token_id
   - Validate signature
   - Check expiration
3. EXTRACT responder's candidates from RSVP
4. NOTE responder's ice_role (should be "controlled")
5. CONFIRM own ice_role is "controlling"
6. CREATE candidate pairs (local × remote)
7. SORT pairs by priority
8. BEGIN simultaneous ICE connectivity checks:
   - Start LISTENING on local candidates
   - Start SENDING to remote candidates with ICE-CONTROLLING attribute
9. FIRST successful pair wins
10. HANDLE any role conflicts via tie-breaker
11. PROCEED to key exchange
```

#### 5.5.2 Invitee Process (After Sending RSVP)

```
Peer B (Original Invitee):

1. GENERATE and SEND RSVP token
2. STORE original invitation + own RSVP locally
3. CREATE candidate pairs (local × remote)
4. SORT pairs by priority
5. BEGIN simultaneous ICE connectivity checks:
   - Start LISTENING on local candidates
   - Start SENDING to remote candidates with ICE-CONTROLLED attribute
6. FIRST successful pair wins
7. HANDLE any role conflicts via tie-breaker
8. PROCEED to key exchange
```

#### 5.5.3 Simultaneous ICE Connectivity Checks

**Key difference from single-token mode**: Both peers send actively

```
Time T:
  Peer A → Peer B: STUN Binding Request
    - To: B's candidate addresses
    - USERNAME = concat(B.ufrag, ":", A.ufrag)
    - MESSAGE-INTEGRITY = HMAC-SHA1(B.pwd, request)
    - ICE-CONTROLLING attribute with A's tie-breaker value
    
  Peer B → Peer A: STUN Binding Request
    - To: A's candidate addresses  
    - USERNAME = concat(A.ufrag, ":", B.ufrag)
    - MESSAGE-INTEGRITY = HMAC-SHA1(A.pwd, request)
    - ICE-CONTROLLED attribute with B's tie-breaker value

These packets create NAT bindings in both directions

Time T + RTT:
  Peer A receives B's request (now allowed through A's NAT)
  Peer B receives A's request (now allowed through B's NAT)
  
  Both respond with STUN Binding Responses
  
Connection established on first pair where both directions succeed
```

**Why this works for Symmetric NAT:**

```
Before simultaneous send:
  A's NAT: Blocks all inbound (no binding exists)
  B's NAT: Blocks all inbound (no binding exists)
  
After A sends to B:
  A's NAT: Creates binding (A's internal IP:port → B's external IP:port)
  B's NAT: Still blocking (hasn't seen outbound to A yet)
  
After B sends to A:
  B's NAT: Creates binding (B's internal IP:port → A's external IP:port)
  A's NAT: Binding already exists
  
Result:
  A → B: Passes through (B's NAT now has binding)
  B → A: Passes through (A's NAT already had binding)
  
  Bidirectional communication established!
```

### 5.6 ICE Role Conflict Resolution

Per RFC 8445 §7.3.1.1, when both peers receive simultaneous binding requests, they must resolve any role conflicts:

**Conflict Detection:**
```
Peer A (controlling) receives request with ICE-CONTROLLING attribute
Peer B (controlled) receives request with ICE-CONTROLLED attribute

This should not happen in our design (invitation=controlling, RSVP=controlled)
However, implementation bugs or network delays could cause it
```

**Resolution Algorithm:**

```
IF (received_role == local_role):
    # Role conflict detected
    IF (received_tie_breaker > local_tie_breaker):
        # Remote peer wins, we switch role
        IF (local_role == "controlling"):
            local_role = "controlled"
        ELSE:
            local_role = "controlling"
        
        # Discard this binding request
        # Update our future requests with new role
        CONTINUE
    ELSE:
        # We win, keep our role
        # Process binding request normally
        SEND binding response
```

**Example:**
```
Peer A: ice_role="controlling", tie_breaker=9999999999
Peer B: ice_role="controlling", tie_breaker=1111111111 (BUG!)

Peer A receives B's request with ICE-CONTROLLING
- Detects conflict (both controlling)
- Compares: 1111111111 < 9999999999
- A wins, stays controlling
- A processes request, sends response

Peer B receives A's request with ICE-CONTROLLING
- Detects conflict (both controlling)
- Compares: 9999999999 > 1111111111
- B loses, switches to controlled
- B discards request, sends new request as controlled
- Future requests from B use ICE-CONTROLLED

Connection proceeds with resolved roles
```

**Why tie-breaker is necessary:**

Without tie-breaker values, both peers might:
1. Detect conflict
2. Both decide to switch roles
3. Now both are controlled (or both controlling)
4. Conflict persists, connection fails

The tie-breaker provides deterministic resolution regardless of network timing.

### 5.7 Candidate Pair Priority

To minimize connection attempts, use ICE-style candidate pairing:

```python
def generate_candidate_pairs(local_candidates, remote_candidates):
    """Create and prioritize all candidate pairs per RFC 8445"""
    pairs = []
    
    for local in local_candidates:
        for remote in remote_candidates:
            # Compute pair priority (ICE algorithm RFC 8445 §6.1.2.3)
            G = min(local['priority'], remote['priority'])
            D = max(local['priority'], remote['priority'])
            priority = (1 << 32) * G + 2 * D + (1 if local['priority'] > remote['priority'] else 0)
            
            pairs.append({
                'local': local,
                'remote': remote,
                'priority': priority,
                'state': 'waiting'  # waiting → in_progress → succeeded/failed
            })
    
    # Sort by priority, highest first
    return sorted(pairs, key=lambda p: p['priority'], reverse=True)


# Attempt top N pairs simultaneously
async def attempt_top_pairs(pairs, max_parallel=5):
    """Try multiple pairs in parallel for faster connection"""
    tasks = []
    
    for pair in pairs[:max_parallel]:
        task = asyncio.create_task(
            attempt_bidirectional_connection(pair)
        )
        tasks.append(task)
    
    # Return first successful connection
    done, pending = await asyncio.wait(
        tasks, 
        return_when=asyncio.FIRST_COMPLETED
    )
    
    # Cancel remaining attempts
    for task in pending:
        task.cancel()
    
    # Return the winner
    return done.pop().result()
```

### 5.8 RSVP Timeout and Retry

Handle cases where RSVP doesn't arrive:

```python
# Invitee side: After sending RSVP
async def wait_for_connection_after_rsvp(rsvp_token, timeout=30):
    """Wait for inviter to process RSVP and connect"""
    try:
        connection = await asyncio.wait_for(
            wait_for_incoming_connection(),
            timeout=timeout
        )
        return connection
    except asyncio.TimeoutError:
        # Inviter may not have received RSVP
        prompt_user("Connection timeout. Send RSVP again?")


# Inviter side: Waiting for RSVP
async def wait_for_rsvp(original_invitation, timeout=60):
    """Wait for invitee to send RSVP token"""
    try:
        rsvp = await asyncio.wait_for(
            listen_for_rsvp_token(),
            timeout=timeout
        )
        return rsvp
    except asyncio.TimeoutError:
        notify_user("No response received from invitee")
        return None
```

---

## 6. Security Considerations

### 6.1 Threat Model

**Assumptions**:
- Attacker can observe token transmission (passive attack)
- Attacker can attempt to connect using captured token
- Attacker can modify token in transit (active attack)
- Network may be untrusted
- RSVP response can also be intercepted

**Protection Goals**:
- Confidentiality: Data encrypted end-to-end
- Authenticity: Verify peer identity
- Integrity: Detect tampering with token or data
- Forward secrecy: Past sessions protected even if keys compromised
- RSVP authenticity: Verify RSVP came from legitimate invitee

### 6.2 Token Security

#### 6.2.1 Signature Verification

Both invitation and RSVP tokens MUST be verified:

```
Invitation Token:
  1. Extract public_key from initiator object
  2. Recompute signature over token fields
  3. Compare with provided signature
  4. Reject if mismatch

RSVP Token:
  1. Extract public_key from responder object
  2. Verify signature on RSVP
  3. Check in_reply_to matches expected invitation
  4. Reject if any verification fails
```

This prevents:
- Token forgery
- Modification of connection parameters
- Impersonation attacks
- Rogue RSVP tokens

#### 6.2.2 RSVP Linking Validation

The inviter MUST validate RSVP links to original invitation:

```python
def validate_rsvp(rsvp_token, original_invitation):
    """Ensure RSVP is legitimate response"""
    
    # Check token type
    if rsvp_token.get('token_type') != 'rsvp':
        raise ValueError("Not an RSVP token")
    
    # Verify links to original
    if rsvp_token.get('in_reply_to') != original_invitation['token_id']:
        raise ValueError("RSVP does not match invitation")
    
    # Check timing (RSVP should be after invitation)
    rsvp_time = parse_iso8601(rsvp_token['created_at'])
    invite_time = parse_iso8601(original_invitation['created_at'])
    
    if rsvp_time < invite_time:
        raise ValueError("RSVP predates invitation")
    
    # Verify signature
    verify_signature(rsvp_token)
    
    # Check expiration
    if is_expired(rsvp_token):
        raise ValueError("RSVP expired")
    
    # Verify complementary ICE roles
    if original_invitation['connection_info']['ice_role'] == 'controlling':
        if rsvp_token['connection_info']['ice_role'] != 'controlled':
            raise ValueError("RSVP has incorrect ICE role")
    
    return True
```

#### 6.2.3 Single-Use RSVP Tokens

For enhanced security, track used RSVP tokens:

```python
# Inviter maintains set of processed RSVPs
processed_rsvps = set()

def process_rsvp(rsvp_token):
    rsvp_id = rsvp_token['token_id']
    
    if rsvp_id in processed_rsvps:
        raise ValueError("RSVP already processed")
    
    # Process RSVP...
    
    # Mark as used
    processed_rsvps.add(rsvp_id)
```

This prevents:
- RSVP replay attacks
- Multiple connections from same RSVP
- Resource exhaustion via repeated RSVPs

#### 6.2.4 Expiration Enforcement

```
Invitation tokens: 24-48 hours typical
RSVP tokens: 1 hour typical (shorter window)

Enforcement:
  current_time = now()
  if current_time > token.expires_at:
      reject("Token expired")
```

Shorter RSVP expiration limits window for:
- Token capture and reuse
- Delayed attacks
- Stale connection attempts

### 6.3 Connection Security

#### 6.3.1 Perfect Forward Secrecy

Key exchange uses ephemeral keys:
- New key pair generated for each connection
- Session keys never reused
- Compromise of long-term keys doesn't expose past sessions

#### 6.3.2 Man-in-the-Middle Protection

Signature verification provides authentication:
- Initiator signs invitation with private key
- Invitee verifies with public key from token
- Invitee signs RSVP with their private key
- Initiator verifies RSVP signature
- MITM cannot forge valid signatures

#### 6.3.3 Replay Protection

Encrypted frames include:
- Monotonically increasing counter
- Random nonce component
- Recipient rejects duplicate counter values

### 6.4 RSVP-Specific Security

#### 6.4.1 RSVP Spoofing Prevention

An attacker who captures the invitation could generate fake RSVP:

**Attack Scenario:**
```
1. Attacker captures invitation token
2. Generates own RSVP with attacker's candidates
3. Sends to inviter
4. Inviter connects to attacker instead of legitimate invitee
```

**Mitigation: Out-of-Band Verification**

```
Option 1: Secure OOB Channel
  - Use encrypted messaging (Signal, WhatsApp)
  - Attacker cannot intercept RSVP

Option 2: Identity Verification
  {
    "metadata": {
      "invitee_identity": "bob@example.com",
      "identity_proof": "signature-from-bob's-key"
    }
  }

Option 3: User Confirmation
  Inviter sees: "RSVP received from unknown peer"
  "Accept connection from peer-b-xyz?"
  [Yes] [No]
```

**Recommended approach**: Combination of encrypted channel + user confirmation

#### 6.4.2 RSVP Flooding Protection

Attacker sends many RSVPs to exhaust resources:

**Protection:**
```python
# Rate limit RSVP processing
rsvp_rate_limiter = RateLimiter(
    max_rsvps=10,
    window_seconds=60
)

def receive_rsvp(rsvp_token):
    if not rsvp_rate_limiter.allow():
        raise RateLimitError("Too many RSVPs")
    
    process_rsvp(rsvp_token)
```

Also:
- Limit simultaneous RSVP processing
- Timeout aggressively on failed RSVP connections
- Require user confirmation for each RSVP

### 6.5 Privacy Considerations

#### 6.5.1 IP Address Disclosure

Tokens contain IP addresses:
- **Host candidates**: Reveal local network topology
- **Reflexive candidates**: Reveal public IP
- **RSVP tokens**: Reveal invitee's IPs back to inviter

**Mitigation**:
- Encrypt tokens for high-sensitivity use cases
- Omit host candidates if privacy required
- Use VPN/Tor to obscure public IP
- User should understand IPs are shared in tokens

#### 6.5.2 Token Transmission Channel

Security depends on out-of-band channel:
- **Encrypted messaging**: Good (Signal, WhatsApp)
- **Email**: Moderate (may be logged)
- **SMS**: Poor (potentially intercepted)
- **Voice**: Poor (may be recorded)
- **Public posting**: Very poor (anyone can use)

#### 6.5.3 RSVP Metadata

RSVP reveals:
- Invitee accepted invitation (confirmation)
- Invitee's network characteristics
- Timing of response

Consider whether this metadata is sensitive for your use case.

### 6.6 Denial of Service

#### 6.6.1 Resource Exhaustion

Protection against malicious tokens:

```
- Rate limit token imports (e.g., 10 per minute)
- Limit simultaneous connection attempts (e.g., 5)
- Timeout aggressive candidates quickly (5 seconds)
- Validate token structure before expensive operations
- Limit RSVP processing (max 10 pending RSVPs)
```

#### 6.6.2 Amplification Attacks

Prevent using system for DDoS:

```
- Validate source addresses in ICE checks
- Require authentication before relay usage
- Rate limit by source IP at relay
- Don't reflect traffic to arbitrary destinations
```

---

## 7. Implementation Guidance

### 7.1 Complete Connection Flow Implementation

```python
class P2PConnectionManager:
    """Manages full connection lifecycle including RSVP fallback"""
    
    def __init__(self, config):
        self.config = config
        self.state = 'idle'
        
    async def create_invitation(self):
        """Generate and share invitation token"""
        
        # Generate invitation token
        invitation = generate_invitation_token(self.config)
        
        # Share via OOB channel (platform share dialog, etc.)
        await self.share_token(invitation)
        
        # Wait for either:
        # - Direct connection from invitee (single-token success)
        # - RSVP token from invitee (symmetric NAT case)
        
        result = await asyncio.wait_for(
            self._wait_for_connection_or_rsvp(invitation),
            timeout=300  # 5 minute overall timeout
        )
        
        return result
    
    async def _wait_for_connection_or_rsvp(self, invitation):
        """Wait for invitee to connect or send RSVP"""
        
        # Start listening for direct connections
        connection_task = asyncio.create_task(
            self._listen_for_connection(invitation)
        )
        
        # Start listening for RSVP tokens
        rsvp_task = asyncio.create_task(
            self._listen_for_rsvp(invitation)
        )
        
        # First one to complete wins
        done, pending = await asyncio.wait(
            [connection_task, rsvp_task],
            return_when=asyncio.FIRST_COMPLETED
        )
        
        # Cancel the other task
        for task in pending:
            task.cancel()
        
        result = done.pop().result()
        
        if isinstance(result, Connection):
            # Direct connection succeeded
            return result
        elif isinstance(result, RSVPToken):
            # Received RSVP, switch to bidirectional mode
            return await self._handle_rsvp(invitation, result)
    
    async def _listen_for_connection(self, invitation):
        """Wait for direct connection from invitee (ICE checks)"""
        # Listen on all candidates
        listeners = []
        for candidate in invitation['connection_info']['ice_candidates']:
            listener = await start_udp_listener(
                candidate['address'],
                candidate['port']
            )
            listeners.append(listener)
        
        # Wait for valid ICE connectivity check
        while True:
            for listener in listeners:
                packet = await listener.receive()
                if is_valid_ice_check(packet, invitation):
                    # Valid check received, verify role
                    if not has_ice_controlled_attribute(packet):
                        log_warning("Invitee should be ICE-CONTROLLED")
                    
                    # Proceed
                    await send_ice_response(listener, packet)
                    return await establish_session(listener, invitation)
    
    async def _listen_for_rsvp(self, invitation):
        """Wait for RSVP token from invitee"""
        # This is application-specific
        # Could be: webhook, polling, message queue, etc.
        rsvp = await self.rsvp_channel.receive()
        validate_rsvp(rsvp, invitation)
        return rsvp
    
    async def _handle_rsvp(self, invitation, rsvp):
        """Process RSVP and establish bidirectional connection"""
        
        # Validate ICE roles are complementary
        assert invitation['connection_info']['ice_role'] == 'controlling'
        assert rsvp['connection_info']['ice_role'] == 'controlled'
        
        # Generate candidate pairs
        pairs = generate_candidate_pairs(
            invitation['connection_info']['ice_candidates'],
            rsvp['connection_info']['ice_candidates']
        )
        
        # Attempt simultaneous connection on top pairs
        connection = await attempt_bidirectional_connection(
            pairs,
            invitation,
            rsvp
        )
        
        return connection
    
    async def join_invitation(self, invitation_token):
        """Join network using received invitation"""
        
        # Parse and validate invitation
        invitation = parse_token(invitation_token)
        validate_token(invitation)
        
        # Verify token type
        if invitation.get('token_type') != 'invitation':
            raise ValueError("Expected invitation token")
        
        # Check connection mode
        if invitation['connection_mode']['initial'] == 'bidirectional':
            # RSVP required from the start
            return await self._join_with_rsvp(invitation)
        
        # Try single-token connection first
        try:
            connection = await asyncio.wait_for(
                self._attempt_single_token_connection(invitation),
                timeout=invitation['connection_mode']['rsvp_timeout_seconds']
            )
            return connection
            
        except (ConnectionError, asyncio.TimeoutError):
            # Single-token failed
            if not invitation['connection_mode']['rsvp_supported']:
                raise ConnectionError("Connection failed and RSVP not supported")
            
            # Prompt for RSVP
            if not await self._prompt_user_for_rsvp():
                raise ConnectionError("User declined RSVP")
            
            return await self._join_with_rsvp(invitation)
    
    async def _attempt_single_token_connection(self, invitation):
        """Try to connect using only invitation token"""
        
        candidates = sorted(
            invitation['connection_info']['ice_candidates'],
            key=lambda c: c['priority'],
            reverse=True
        )
        
        # Extract inviter's ICE role and tie-breaker
        inviter_role = invitation['connection_info']['ice_role']
        assert inviter_role == 'controlling', "Invitation should have controlling role"
        
        # Assign own role as controlled
        own_role = 'controlled'
        own_tie_breaker = random.randint(0, 2**64-1)
        
        # Try each candidate
        for candidate in candidates:
            try:
                # Send ICE connectivity check
                sock = await connect_to_candidate(candidate)
                
                # Send STUN Binding Request with ICE-CONTROLLED attribute
                await send_ice_check(
                    sock, 
                    invitation,
                    ice_role=own_role,
                    tie_breaker=own_tie_breaker
                )
                
                # Wait for response
                response = await asyncio.wait_for(
                    receive_ice_response(sock),
                    timeout=3.0
                )
                
                # Success! Proceed to key exchange
                return await establish_session(sock, invitation)
                
            except (ConnectionError, asyncio.TimeoutError):
                # This candidate failed, try next
                continue
        
        # All candidates failed
        raise ConnectionError("All ICE connectivity checks failed")
    
    async def _prompt_user_for_rsvp(self):
        """Ask user if they want to send RSVP response"""
        
        # Platform-specific UI prompt
        return await show_dialog(
            title="Connection Needs Response",
            message="Due to your network configuration, the inviter "
                   "needs connection information from you to complete setup.\n\n"
                   "Send response?",
            buttons=["Send Response", "Cancel"]
        ) == "Send Response"
    
    async def _join_with_rsvp(self, invitation):
        """Join using bidirectional RSVP method"""
        
        # Generate RSVP token
        rsvp = generate_rsvp_token(invitation)
        
        # Verify complementary roles
        assert invitation['connection_info']['ice_role'] == 'controlling'
        assert rsvp['connection_info']['ice_role'] == 'controlled'
        
        # Share RSVP back to inviter
        await self.share_token(rsvp)
        
        # Generate candidate pairs
        pairs = generate_candidate_pairs(
            rsvp['connection_info']['ice_candidates'],
            invitation['connection_info']['ice_candidates']
        )
        
        # Attempt simultaneous connection
        connection = await attempt_bidirectional_connection(
            pairs,
            rsvp,
            invitation
        )
        
        return connection
```

### 7.2 Bidirectional Connection with Role Conflict Resolution

```python
async def attempt_bidirectional_connection(pairs, local_token, remote_token):
    """
    Attempt simultaneous connection on multiple candidate pairs.
    Both peers send and receive simultaneously for NAT hole punching.
    Includes ICE role conflict resolution per RFC 8445.
    """
    
    # Try top 5 pairs in parallel
    tasks = []
    for pair in pairs[:5]:
        task = asyncio.create_task(
            attempt_pair_bidirectional(pair, local_token, remote_token)
        )
        tasks.append(task)
    
    # Wait for first success
    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.FIRST_COMPLETED
    )
    
    # Cancel remaining attempts
    for task in pending:
        task.cancel()
    
    # Return successful connection
    return done.pop().result()


async def attempt_pair_bidirectional(pair, local_token, remote_token):
    """
    Attempt connection on single candidate pair.
    Sends ICE checks while also listening for incoming checks.
    Handles role conflicts via tie-breaker comparison.
    """
    
    # Extract ICE parameters
    local_role = local_token['connection_info']['ice_role']
    local_tie_breaker = int(local_token['connection_info']['ice_tie_breaker'])
    remote_role = remote_token['connection_info']['ice_role']
    remote_tie_breaker = int(remote_token['connection_info']['ice_tie_breaker'])
    
    # Create UDP socket bound to local candidate
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.bind((pair['local']['address'], pair['local']['port']))
    sock.setblocking(False)
    
    # Track current role (may change due to conflict resolution)
    current_role = local_role
    
    # Prepare STUN Binding Request
    def create_request(role):
        return create_stun_binding_request(
            username=f"{remote_token['connection_info']['ufrag']}:"
                    f"{local_token['connection_info']['ufrag']}",
            password=remote_token['connection_info']['pwd'],
            ice_role=role,
            tie_breaker=local_tie_breaker
        )
    
    stun_request = create_request(current_role)
    
    # Send repeatedly to punch hole
    async def keep_sending():
        nonlocal stun_request, current_role
        for _ in range(20):  # Send for 4 seconds
            try:
                # Regenerate request if role changed
                if stun_request is None:
                    stun_request = create_request(current_role)
                
                await asyncio.get_event_loop().sock_sendto(
                    sock,
                    stun_request,
                    (pair['remote']['address'], pair['remote']['port'])
                )
            except Exception as e:
                logger.debug(f"Send error: {e}")
            await asyncio.sleep(0.2)
    
    # Listen for response
    async def wait_for_response():
        nonlocal current_role, stun_request
        loop = asyncio.get_event_loop()
        start_time = time.time()
        
        while time.time() - start_time < 5.0:  # 5 second timeout
            try:
                data, addr = await asyncio.wait_for(
                    loop.sock_recvfrom(sock, 1024),
                    timeout=0.5
                )
                
                # Check if it's a STUN binding request (role conflict scenario)
                if is_stun_binding_request(data):
                    received_role, received_tie_breaker = extract_ice_role_and_tie_breaker(data)
                    
                    # Check for role conflict
                    if received_role == current_role:
                        logger.info(f"ICE role conflict detected: both {current_role}")
                        
                        # Resolve conflict via tie-breaker comparison
                        if received_tie_breaker > local_tie_breaker:
                            # Remote wins, we switch
                            logger.info(f"Switching role from {current_role}")
                            current_role = 'controlled' if current_role == 'controlling' else 'controlling'
                            stun_request = None  # Regenerate with new role
                            logger.info(f"New role: {current_role}")
                            continue
                        else:
                            # We win, stay in current role
                            logger.info(f"Keeping role: {current_role}")
                    
                    # Send response to their request
                    response = create_stun_binding_response(data, local_token)
                    await loop.sock_sendto(sock, response, addr)
                
                # Check if valid STUN response to our request
                elif is_valid_stun_response(data, local_token):
                    return sock  # Success!
                    
            except asyncio.TimeoutError:
                continue
        
        raise ConnectionError("No response received")
    
    # Run both simultaneously
    send_task = asyncio.create_task(keep_sending())
    recv_task = asyncio.create_task(wait_for_response())
    
    try:
        # Wait for response
        result = await recv_task
        send_task.cancel()
        return result
        
    except ConnectionError:
        send_task.cancel()
        raise


def create_stun_binding_request(username, password, ice_role, tie_breaker):
    """
    Create STUN Binding Request for ICE connectivity check.
    Includes ICE-CONTROLLING or ICE-CONTROLLED attribute per RFC 8445.
    """
    
    # STUN message structure
    msg_type = 0x0001  # Binding Request
    magic_cookie = 0x2112A442
    transaction_id = os.urandom(12)
    
    # Build message
    message = struct.pack('!HH', msg_type, 0)  # Type and length placeholder
    message += struct.pack('!I', magic_cookie)
    message += transaction_id
    
    # Add USERNAME attribute
    username_bytes = username.encode('utf-8')
    message += pack_stun_attribute(0x0006, username_bytes)
    
    # Add ICE-CONTROLLING (0x802A) or ICE-CONTROLLED (0x8029) attribute
    if ice_role == 'controlling':
        attr_type = 0x802A  # ICE-CONTROLLING
    else:
        attr_type = 0x8029  # ICE-CONTROLLED
    
    tie_breaker_bytes = struct.pack('!Q', tie_breaker)  # 64-bit value
    message += pack_stun_attribute(attr_type, tie_breaker_bytes)
    
    # Add MESSAGE-INTEGRITY attribute (HMAC-SHA1)
    # Update length field
    msg_length = len(message) - 20 + 24  # +24 for MESSAGE-INTEGRITY
    message = struct.pack('!HH', msg_type, msg_length) + message[4:]
    
    # Compute HMAC
    hmac_key = password.encode('utf-8')
    integrity = hmac.new(hmac_key, message, hashlib.sha1).digest()
    message += pack_stun_attribute(0x0008, integrity)
    
    return message


def extract_ice_role_and_tie_breaker(stun_data):
    """
    Extract ICE role and tie-breaker from STUN Binding Request.
    Returns: (role, tie_breaker) where role is 'controlling' or 'controlled'
    """
    # Skip STUN header (20 bytes)
    offset = 20
    
    while offset < len(stun_data):
        attr_type, attr_length = struct.unpack('!HH', stun_data[offset:offset+4])
        attr_value = stun_data[offset+4:offset+4+attr_length]
        
        if attr_type == 0x802A:  # ICE-CONTROLLING
            tie_breaker = struct.unpack('!Q', attr_value)[0]
            return ('controlling', tie_breaker)
        elif attr_type == 0x8029:  # ICE-CONTROLLED
            tie_breaker = struct.unpack('!Q', attr_value)[0]
            return ('controlled', tie_breaker)
        
        # Move to next attribute (with padding)
        offset += 4 + attr_length
        offset += (4 - (attr_length % 4)) % 4  # Padding to 4-byte boundary
    
    raise ValueError("No ICE role attribute found")


def is_valid_stun_response(data, local_token):
    """Validate received STUN Binding Response"""
    
    if len(data) < 20:
        return False
    
    # Parse STUN header
    msg_type, msg_length, magic_cookie = struct.unpack('!HHI', data[:8])
    
    # Check if Binding Response
    if msg_type != 0x0101:  # Binding Response
        return False
    
    # Verify magic cookie
    if magic_cookie != 0x2112A442:
        return False
    
    # Verify MESSAGE-INTEGRITY if present
    # (Full implementation omitted for brevity)
    
    return True


def pack_stun_attribute(attr_type, value):
    """Pack STUN attribute with proper padding"""
    length = len(value)
    padding = (4 - (length % 4)) % 4
    return struct.pack('!HH', attr_type, length) + value + (b'\x00' * padding)
```

### 7.3 Token Generation with Correct Fields

```python
def generate_invitation_token(config):
    """Generate invitation token with all required fields"""
    
    private_key, public_key = generate_key_pair()
    candidates = gather_ice_candidates(config)
    
    token = {
        'version': '1.1',
        'token_type': 'invitation',  # Explicit type
        'token_id': str(uuid.uuid4()),
        'created_at': datetime.utcnow().isoformat() + 'Z',
        'expires_at': (datetime.utcnow() + timedelta(hours=24)).isoformat() + 'Z',
        'network_id': config.get('network_id'),
        
        'connection_mode': {
            'initial': 'single-token',
            'rsvp_supported': True,
            'rsvp_timeout_seconds': 12
        },
        
        'initiator': {
            'peer_id': generate_peer_id(),
            'public_key': base64_encode(public_key),
            'capabilities': config.get('capabilities', ['data-channel'])
        },
        
        'connection_info': {
            'ice_candidates': candidates,
            'ufrag': secrets.token_urlsafe(6),
            'pwd': secrets.token_urlsafe(18),
            'ice_role': 'controlling',  # Invitation is always controlling
            'ice_tie_breaker': str(secrets.randbits(64))  # Random 64-bit value
        },
        
        'crypto': {
            'key_exchange': 'ecdh-curve25519',
            'cipher': 'chacha20-poly1305',
            'fingerprint': compute_fingerprint(public_key)
        },
        
        'relay': config.get('relay', {'enabled': False}),
        'metadata': config.get('metadata', {})
    }
    
    # Sign token
    canonical = canonicalize_json(token)
    signature = sign(private_key, hashlib.sha256(canonical.encode()).digest())
    token['signature'] = base64_encode(signature)
    
    return encode_token(token), private_key


def generate_rsvp_token(invitation):
    """Generate RSVP token in response to invitation"""
    
    private_key, public_key = generate_key_pair()
    candidates = gather_ice_candidates(config)
    
    token = {
        'version': '1.1',
        'token_type': 'rsvp',  # Explicit type
        'token_id': str(uuid.uuid4()),
        'in_reply_to': invitation['token_id'],  # Link to invitation
        'created_at': datetime.utcnow().isoformat() + 'Z',
        'expires_at': (datetime.utcnow() + timedelta(hours=1)).isoformat() + 'Z',
        'network_id': invitation['network_id'],
        
        'connection_mode': {
            'initial': 'bidirectional',
            'rsvp_supported': False
        },
        
        'responder': {  # Note: 'responder' not 'initiator'
            'peer_id': generate_peer_id(),
            'public_key': base64_encode(public_key),
            'capabilities': ['data-channel']
        },
        
        'connection_info': {
            'ice_candidates': candidates,
            'ufrag': secrets.token_urlsafe(6),
            'pwd': secrets.token_urlsafe(18),
            'ice_role': 'controlled',  # RSVP is always controlled
            'ice_tie_breaker': str(secrets.randbits(64))  # Own tie-breaker
        },
        
        'crypto': {
            'key_exchange': 'ecdh-curve25519',
            'cipher': 'chacha20-poly1305',
            'fingerprint': compute_fingerprint(public_key)
        },
        
        'relay': {'enabled': False},
        'metadata': invitation.get('metadata', {})
    }
    
    # Sign token
    canonical = canonicalize_json(token)
    signature = sign(private_key, hashlib.sha256(canonical.encode()).digest())
    token['signature'] = base64_encode(signature)
    
    return encode_token(token), private_key
```

### 7.4 Testing Considerations

```python
# Test ICE role assignment
def test_ice_role_assignment():
    """Verify invitation=controlling, RSVP=controlled"""
    invitation, _ = generate_invitation_token(config)
    parsed_invite = parse_token(invitation)
    assert parsed_invite['connection_info']['ice_role'] == 'controlling'
    
    rsvp, _ = generate_rsvp_token(parsed_invite)
    parsed_rsvp = parse_token(rsvp)
    assert parsed_rsvp['connection_info']['ice_role'] == 'controlled'


# Test role conflict resolution
def test_role_conflict_resolution():
    """Test tie-breaker resolves conflicts correctly"""
    
    # Simulate both peers thinking they're controlling
    peer_a_tie_breaker = 9999999999
    peer_b_tie_breaker = 1111111111
    
    # Both start as controlling (simulated bug)
    peer_a_role = 'controlling'
    peer_b_role = 'controlling'
    
    # Conflict resolution
    if peer_b_tie_breaker > peer_a_tie_breaker:
        peer_a_role = 'controlled'
    else:
        peer_b_role = 'controlled'
    
    # Verify deterministic resolution
    assert peer_a_role == 'controlling'
    assert peer_b_role == 'controlled'


# Test RSVP linking
def test_rsvp_linking():
    """Test RSVP properly links to invitation"""
    invitation, _ = generate_invitation_token(config)
    parsed_invite = parse_token(invitation)
    
    rsvp, _ = generate_rsvp_token(parsed_invite)
    parsed_rsvp = parse_token(rsvp)
    
    assert parsed_rsvp['token_type'] == 'rsvp'
    assert parsed_rsvp['in_reply_to'] == parsed_invite['token_id']
    assert 'responder' in parsed_rsvp
    assert 'initiator' not in parsed_rsvp
```

---

## 8. Comparison to Existing Approaches

### 8.1 vs. WebRTC with Signaling Server

| Aspect | WebRTC + Signaling | This Specification |
|--------|-------------------|-------------------|
| Server dependency | Required | None |
| Setup complexity | High (deploy server) | Low (share token) |
| Privacy | Server sees metadata | No server involvement |
| Scalability | Server bottleneck | Fully distributed |
| Operational cost | Server hosting | Zero |
| Connection time | Fast (server assists) | Similar (direct attempts) |
| Symmetric NAT | Requires signaling | RSVP handles it |
| ICE role management | Server coordinates | Tokens assign roles |
| User steps | 1 (join link) | 1-2 (join + RSVP if needed) |

### 8.2 vs. Magic Wormhole

| Aspect | Magic Wormhole | This Specification |
|--------|----------------|-------------------|
| Rendezvous server | Required for discovery | Not required |
| Code format | Human words (numeric + words) | Encoded token |
| Token size | Small (~20 chars) | Larger (~500-2000 chars) |
| Use case | File transfer | General P2P networking |
| Multiple connections | New code each time | Token reusable (optional) |
| PAKE security | Yes (SPAKE2) | ECDH with signatures |
| Symmetric NAT | Rendezvous assists | RSVP + ICE roles |
| Role management | Implicit | Explicit (controlling/controlled) |

### 8.3 vs. NFC/Bluetooth OOB Pairing

| Aspect | NFC/Bluetooth | This Specification |
|--------|---------------|-------------------|
| Range | Physical proximity | Internet-scale |
| Technology | Bluetooth/NFC | IP networking |
| Use case | Device pairing | Network connections |
| Sharing method | Tap/proximity | Any channel |
| Infrastructure | None | STUN/TURN optional |
| Bidirectional | Single tap usually | RSVP when needed |
| Role assignment | Implicit | Explicit ICE roles |

### 8.4 vs. Tailscale/WireGuard

| Aspect | Tailscale | This Specification |
|--------|-----------|-------------------|
| Coordination server | Required (control plane) | Not required |
| Account/auth | Required | Not required |
| Permanent network | Yes | Ad-hoc/temporary |
| Key management | Centralized | Per-token |
| Use case | Infrastructure VPN | Application P2P |
| Symmetric NAT | DERP relays | RSVP + simultaneous checks |
| Role conflict | Coordination server | Tie-breaker algorithm |

### 8.5 Key Differentiators

**This specification uniquely provides:**

1. **Progressive enhancement**: Simple path first, complex only when needed
2. **Zero infrastructure**: No servers at all (optional STUN/TURN for NAT)
3. **User-controlled RSVP**: Explicit prompt before bidirectional exchange
4. **Platform agnostic**: Works with any sharing mechanism
5. **Symmetric NAT support**: Built-in via RSVP mechanism
6. **Privacy-first**: No third party sees connection metadata
7. **ICE compliance**: Proper role management and conflict resolution per RFC 8445
8. **Explicit semantics**: Clear offer/answer model mapping

---

## 9. Use Cases

[Content remains unchanged from previous version - use cases are still valid]

---

## 10. Extensions and Variations

[Content remains unchanged from previous version]

---

## 11. Standards and Interoperability

### 11.1 Related Standards

This specification builds upon:

- **RFC 8445**: Interactive Connectivity Establishment (ICE)
  - ICE role assignment (controlling/controlled)
  - Tie-breaker for role conflict resolution (§7.3.1.1)
  - Candidate pair prioritization
- **RFC 5389**: Session Traversal Utilities for NAT (STUN)
  - STUN Binding Requests and Responses
  - ICE-CONTROLLING and ICE-CONTROLLED attributes
- **RFC 5766**: Traversal Using Relays around NAT (TURN)
- **RFC 3264**: An Offer/Answer Model with SDP
  - Offer/answer semantics (invitation ≅ offer, RSVP ≅ answer)
- **RFC 7748**: Elliptic Curves for Security (Curve25519)
- **RFC 8439**: ChaCha20-Poly1305 AEAD
- **RFC 8032**: Edwards-Curve Digital Signature Algorithm (EdDSA)

### 11.2 Deviations from Standard ICE

This specification differs from standard ICE (RFC 8445) in:

1. **Token-based distribution**: Candidates shared via OOB token vs. SDP signaling
2. **Progressive enhancement**: Single-token attempt before bidirectional
3. **User-controlled RSVP**: Explicit prompt for bidirectional mode
4. **No signaling server**: RSVP shared via same OOB channel as invitation
5. **Pre-assigned roles**: Roles determined by token type (invitation=controlling, RSVP=controlled) rather than negotiated

### 11.3 ICE Compliance Notes

**Compliant behaviors:**
- ICE-CONTROLLING and ICE-CONTROLLED attributes in STUN requests
- Tie-breaker values for role conflict resolution
- Candidate priority calculation
- Connectivity check procedures

**Non-standard (but compatible) behaviors:**
- Role assignment by token type (simplifies implementation)
- Progressive single-token attempt (optimization)
- Out-of-band candidate exchange (instead of SDP)

Implementations can interoperate with standard ICE agents by:
- Supporting role negotiation if needed
- Following standard conflict resolution
- Using standard STUN message formats

### 11.4 JSON Schema

Formal schemas available at:
```
https://[spec-location]/schema/invitation-v1.1.json
https://[spec-location]/schema/rsvp-v1.1.json
```

### 11.5 Test Vectors

Reference implementations should pass:
- Valid invitation token examples
- Valid RSVP token examples
- Invalid token examples (various failure modes)
- Signature verification test cases
- RSVP linking validation test cases
- ICE role conflict resolution scenarios
- Bidirectional connection scenarios

---

## 12. Future Work

### 12.1 Standardization

Potential paths:
- IETF Internet-Draft submission
- RFC proposal (potentially as ICE extension)
- W3C Community Group
- Integration with WebRTC standards

### 12.2 Protocol Enhancements

Areas for expansion:

**Improved RSVP UX:**
- Automatic RSVP via app-to-app messaging
- Background RSVP without user interaction
- Visual RSVP flow (mutual QR scanning)

**Advanced NAT Traversal:**
- NAT-PMP/UPnP integration for port mapping
- Prediction algorithms for symmetric NAT ports
- Multi-path connections (aggregate bandwidth)
- Consent-based hole punching

**Mobile Optimization:**
- Cellular network awareness
- Battery-efficient keep-alive for NAT binding refresh
- Fast dormancy handling
- Network transition support (WiFi ↔ cellular)

**Performance:**
- Parallel candidate attempts with early success
- Fast connection upgrade (relay → direct)
- Bandwidth estimation and adaptation
- Congestion control

**ICE Enhancements:**
- TCP candidate support
- IPv6 optimization
- mDNS candidates for local discovery
- Consent freshness (RFC 7675)

### 12.3 Ecosystem Development

- Reference implementations (JavaScript, Python, Go, Rust)
- Mobile SDKs (iOS, Android)
- Desktop libraries (Electron, Qt)
- Testing and debugging tools
- Public relay infrastructure (optional)
- RSVP exchange services (optional helpers)
- Interoperability test suite
- ICE role conflict simulator

### 12.4 Research Directions

- Optimal RSVP timeout algorithms
- Machine learning for NAT type prediction
- Privacy-preserving relay selection
- Token size optimization techniques
- Post-quantum cryptography migration
- ICE trickling over OOB channels

---

## 13. Conclusion

This specification presents a novel approach to peer-to-peer connection establishment that eliminates traditional signaling server dependencies while handling the full spectrum of network configurations. The key innovations are:

1. **Self-contained invitation tokens** that embed all connection parameters
2. **Progressive enhancement** from simple single-token to bidirectional RSVP
3. **User-controlled escalation** with explicit prompts for RSVP exchange
4. **Symmetric NAT support** via simultaneous bidirectional ICE connectivity checks
5. **ICE role management** with proper controlling/controlled assignment and tie-breaker resolution
6. **Zero infrastructure** requirement (optional STUN/TURN for NAT traversal)
7. **Semantic clarity** through explicit offer/answer model alignment

The protocol achieves an optimal balance:
- **Simplicity** for common cases (one token share, direct connection)
- **Reliability** for challenging networks (automatic RSVP fallback)
- **Correctness** (proper ICE role management prevents deadlocks)
- **User experience** that adapts to network conditions
- **Privacy** through end-to-end encryption and no server intermediaries
- **Standards compliance** (RFC 8445 ICE role conflict resolution)

**When to use this approach:**

✓ Small ad-hoc networks (2-100 participants)  
✓ Privacy-sensitive applications  
✓ Serverless architectures  
✓ IoT device provisioning  
✓ Peer-to-peer gaming and collaboration  
✓ Emergency/disaster communication  
✓ Applications where infrastructure cost must be minimized  
✓ Symmetric NAT environments

**When traditional approaches may be better:**

✗ Large-scale deployments (1000+ concurrent connections)  
✗ Complex matchmaking requirements  
✗ Centralized monitoring/logging needed  
✗ Guaranteed sub-50ms latency required  
✗ Users cannot exchange tokens out-of-band

This specification demonstrates that sophisticated P2P networking is achievable without centralized infrastructure, while the progressive RSVP mechanism with proper ICE role management ensures compatibility across all network configurations, from simple home networks to restrictive corporate and mobile carrier symmetric NATs.

---

## Appendices

## Appendix A: Complete Token Examples

### A.1 Invitation Token (Version 1.1)

```json
{
  "version": "1.1",
  "token_type": "invitation",
  "token_id": "550e8400-e29b-41d4-a716-446655440000",
  "created_at": "2026-02-02T10:30:00Z",
  "expires_at": "2026-02-03T10:30:00Z",
  "network_id": "my-game-session-abc123",
  
  "connection_mode": {
    "initial": "single-token",
    "rsvp_supported": true,
    "rsvp_timeout_seconds": 12
  },
  
  "initiator": {
    "peer_id": "alice-peer-12345",
    "public_key": "MC4CAQAwBQYDK2VuBCIEINTXMNgMQFbOPqNQcw+X+TqrhVJWHRGx8J6mTGN1gN9F",
    "capabilities": ["data-channel", "audio"]
  },
  
  "connection_info": {
    "ice_candidates": [
      {
        "type": "host",
        "protocol": "udp",
        "address": "192.168.1.100",
        "port": 54321,
        "priority": 2130706431
      },
      {
        "type": "srflx",
        "protocol": "udp",
        "address": "203.0.113.45",
        "port": 54321,
        "related_address": "192.168.1.100",
        "related_port": 54321,
        "priority": 1694498815
      }
    ],
    "ufrag": "8hhY",
    "pwd": "asd88fgpdd777uzjYhagZg",
    "ice_role": "controlling",
    "ice_tie_breaker": "1827364950273645"
  },
  
  "crypto": {
    "key_exchange": "ecdh-curve25519",
    "cipher": "chacha20-poly1305",
    "fingerprint": "sha-256 AB:CD:EF:01:23:45:67:89:AB:CD:EF:01:23:45:67:89:AB:CD:EF:01:23:45:67:89:AB:CD:EF:01:23:45:67:89",
    "dtls_setup": "actpass"
  },
  
  "relay": {
    "enabled": false
  },
  
  "metadata": {
    "app_id": "my-game-v1",
    "protocol_version": "1.0",
    "description": "Join Alice's game"
  },
  
  "signature": "lQgYh4HCj_G-o_MFju8Wm5P0JKZ9NVg_Zp9j0VmY8E4KwV_J9X2n5Xv9p3Pp9ZxVg4E_Jp9VmY8E4KwV9p3P"
}
```

**Encoded (Base64URL)**:
```
scpi://eJydVU1z2jAQ_SuMT02BlCzJsg69MECBAkWRQ4EeihzWJNcSQpIqSdlqg_x7V5LkOE3RYheDJJLv8ZEfrR_example_continues...
```

### A.2 RSVP Token (Version 1.1)

```json
{
  "version": "1.1",
  "token_type": "rsvp",
  "token_id": "rsvp-661f9410-f3ac-52e5-b827-557766551111",
  "in_reply_to": "550e8400-e29b-41d4-a716-446655440000",
  "created_at": "2026-02-02T10:30:15Z",
  "expires_at": "2026-02-02T11:30:15Z",
  "network_id": "my-game-session-abc123",
  
  "connection_mode": {
    "initial": "bidirectional",
    "rsvp_supported": false
  },
  
  "responder": {
    "peer_id": "bob-peer-67890",
    "public_key": "NC5DBRBxCGZFL3VxCDJRRw-Y-UrsiwKLYISHy9K7nUHN2hO0G",
    "capabilities": ["data-channel"]
  },
  
  "connection_info": {
    "ice_candidates": [
      {
        "type": "host",
        "protocol": "udp",
        "address": "10.0.0.50",
        "port": 48291,
        "priority": 2130706431
      },
      {
        "type": "srflx",
        "protocol": "udp",
        "address": "198.51.100.78",
        "port": 48291,
        "related_address": "10.0.0.50",
        "related_port": 48291,
        "priority": 1694498815
      }
    ],
    "ufrag": "7kpM",
    "pwd": "bce99hqree888vzlZibhaHh",
    "ice_role": "controlled",
    "ice_tie_breaker": "7364829104738291"
  },
  
  "crypto": {
    "key_exchange": "ecdh-curve25519",
    "cipher": "chacha20-poly1305",
    "fingerprint": "sha-256 12:34:56:78:9A:BC:DE:F0:12:34:56:78:9A:BC:DE:F0:12:34:56:78:9A:BC:DE:F0:12:34:56:78:9A:BC:DE:F0",
    "dtls_setup": "actpass"
  },
  
  "relay": {
    "enabled": false
  },
  
  "metadata": {
    "app_id": "my-game-v1",
    "protocol_version": "1.0",
    "description": "RSVP to Alice's invitation"
  },
  
  "signature": "mRhZi5IDk_H-p_NGkv9Xn6Q1KLa0OWh_aq0k1WnZ9F5LxW_K0Y3o6Yw0q4Qq0ayWh5F_Kq0WnZ9F5LxW0q4Q"
}
```

---

## Appendix B: Glossary

- **Bidirectional mode**: Connection establishment where both peers exchange tokens and send ICE checks simultaneously
- **Candidate**: A potential network address where a peer can be reached
- **Controlling (ICE role)**: The ICE agent role that makes final decisions on candidate pair selection
- **Controlled (ICE role)**: The ICE agent role that defers to the controlling agent
- **ECDH**: Elliptic Curve Diffie-Hellman key exchange
- **ICE**: Interactive Connectivity Establishment
- **ICE connectivity check**: STUN Binding Request sent to verify connectivity
- **Invitation token**: Initial token created by inviter (equivalent to SDP offer)
- **Invitee**: Peer receiving and using an invitation token
- **Initiator**: Peer creating and distributing an invitation token
- **NAT**: Network Address Translation
- **NAT hole punching**: Technique for establishing connections through NAT by simultaneous packet sending
- **Offer**: SDP terminology for initial connection proposal (≅ invitation)
- **Answer**: SDP terminology for response to offer (≅ RSVP)
- **Out-of-band (OOB)**: Communication channel external to main protocol
- **Peer**: Participant in a peer-to-peer network
- **Progressive enhancement**: Starting with simple approach, escalating to complex only when needed
- **Reflexive address**: Public IP address as seen by STUN server
- **Relay**: Server that forwards traffic when direct connection impossible
- **RSVP**: Response token sent by invitee back to inviter (equivalent to SDP answer)
- **SDP**: Session Description Protocol
- **Single-token mode**: Connection establishment using only invitation token (no RSVP)
- **STUN**: Session Traversal Utilities for NAT
- **STUN Binding Request**: ICE connectivity check packet
- **Symmetric NAT**: Most restrictive NAT type requiring bidirectional packet flow
- **Tie-breaker**: Random 64-bit value used to resolve ICE role conflicts
- **Token**: Self-contained invitation or RSVP containing connection parameters
- **TURN**: Traversal Using Relays around NAT

---

## Appendix C: Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-02-02 | Initial public release with RSVP mechanism |
| 1.1 | 2026-02-02 | • Added explicit token_type field ("invitation"/"rsvp")<br>• Renamed response_to → in_reply_to<br>• Added ICE role assignment (controlling/controlled)<br>• Added ice_tie_breaker for conflict resolution<br>• Changed "initiator" to "responder" in RSVP tokens<br>• Added role conflict resolution algorithm (RFC 8445)<br>• Clarified terminology: "ping" → "ICE connectivity check"<br>• Added NAT binding expiry discussion<br>• Added offer/answer semantics section<br>• Updated all examples and code |

---

## Appendix D: Acknowledgments

This work builds upon and is inspired by:

- **WebRTC** and the **IETF RTCWeb Working Group** for ICE, STUN, and TURN standards (RFC 8445, RFC 5389, RFC 5766)
- **Magic Wormhole** (Brian Warner) for demonstrating PAKE-based file transfer with human-transcribable codes
- **NFC Forum and Bluetooth SIG** for out-of-band pairing concepts
- **Tailscale** for demonstrating user-friendly NAT traversal
- **libsodium/NaCl** (Daniel J. Bernstein et al.) for modern cryptography libraries
- The reviewers who identified critical gaps in ICE role management and semantic clarity
- The open source community for countless P2P experiments and implementations

Special thanks to all who contributed ideas, code, feedback, and rigorous technical review.

---

**END OF SPECIFICATION VERSION 1.1**

---

**Maintained at**: [\[repository URL\]](https://github.com/iometics/serverless-p2p-invitations)  
**For questions, suggestions, or contributions**: [contact information]  
**Latest version**: 1.1  
**Discussion forum**: TBD  
**Reference implementation**: TBD

---

**This specification is dedicated to the public domain. Use it freely.**

---

## Summary of Changes in Version 1.1

**Major improvements:**

1. ✅ **Token type semantics**: Added explicit `token_type` field and clarified offer/answer mapping
2. ✅ **ICE role management**: Added `ice_role` and `ice_tie_breaker` fields with proper conflict resolution
3. ✅ **Consistent terminology**: Replaced "ping" with "ICE connectivity check" throughout
4. ✅ **Semantic clarity**: Changed RSVP tokens to use `responder` field instead of `initiator`
5. ✅ **NAT considerations**: Added discussion of NAT binding expiry and why RSVP resolves it
6. ✅ **RFC 8445 compliance**: Added proper role conflict resolution algorithm with tie-breaker
7. ✅ **Field renaming**: Changed `response_to` → `in_reply_to` for consistency

All implementation examples, test cases, and token examples have been updated to reflect these changes.