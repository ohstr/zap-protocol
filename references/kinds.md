# Zap Protocol: Event Kinds & Data Formats

## The 3-Element Tag Standard
To ensure universal legacy identity resolution without a hard requirement for a Nostr wallet, all identity tags (`p` for Recipient, `P` for Sender) MUST use the following array structure:
`["p/P", "<hex_or_ConnectionKey>", "<lidp_name>"]`

- **[0] Key**: `p` (Recipient), `P` (Sender)
- **[1] Identity**: 64-character hex pubkey OR ConnectionKey.
- **[2] Provider**: The source. Used values: `nostr`, `discord`, `email`, `x`.

## Kind 5520: Native Zap Request
Used by native Nostr users to zap another user or LNURL destination.
* **Initiator:** Signed by Sender's Private Key.
* **Mechanism:** LNURL-Pay.
* **Required Tags:**
    * `p`: Recipient identity (e.g., `["p", "<hex_or_ConnectionKey>", "<optional_lidp_name>"]`). If the 3rd parameter is omitted or empty, it defaults to `nostr` (pubkey).
    * `amount`: Total value in milli-units.
    * `lnurl`: Bech32 LNURL of the recipient.
    * `relays`: Target relays.
    * `chain`: `flokicoin` or `bitcoin`.
* **Optional Tags:**
    * `e`: Target event ID if zapping a specific note.
    * `a`: Target coordinate if zapping a replaceable event.
    * `k`: Stringified kind of the target event.

## Kind 5523: On-Behalf Request (Proxy)
Used by bots (Proxy Agents) to represent users on legacy platforms.
* **Initiator:** Signed by Proxy Agent's Key (e.g., Zapf Discord Bot).
* **Mechanism:** LNURL-Pay.
* **Required Tags:**
    * `P`: True Sender identity (3-element format, e.g., `["P", "<ConnectionKey>", "<lidp_name>"]`).
    * `p`: Recipient identity (3-element format).
    * `amount`: Value in milli-units.
    * `lnurl`: Recipient LNURL endpoint.
    * `chain`: Network context.

**Validation / Identity Resolution (via Kind 35521):**
When parsing a `5523` event (or its resultant `5521`), the ConnectionKey is the primary identifier. To display a "Verified" UI experience, clients MUST query for a **Kind 35521 (Identity Connection)** where the `#d` tag matches the ConnectionKey. If no 35521 exists, display the fallback LIDP identity (e.g., Provider Avatar).

## Kind 5521: Universal Zap Receipt
The final proof of settlement issued by the ZSP.
* **Initiator:** Signed by ZSP's Private Key.
* **Required Tags:**
    * `p`: Mirror Recipient. (Exception: Appended POST-settlement in 5522 if resolved via claim link).
    * `P`: Mirror Sender.
    * `amount`: Settled milli-units extracted from the BOLT11.
    * `preimage`: Cryptographic proof (32-byte hex).
    * `description`: Stringified JSON of the Request event (5520/5523).
    * `chain`: Settled network.
* **Optional Tags:**
    * `r`: Resolved Recipient (Used to map a `p` ConnectionKey back to a native pubkey if the ZSP knows the 35521 connection).
    * `R`: Resolved Sender (Used to map a `P` ConnectionKey back to a native pubkey if the ZSP knows the 35521 connection). Enables "Social Bridge" UX.

### Identity Tag Matrix for Agents (5521 Generation)

When generating a 5521 Zap Receipt, use this matrix to correctly populate identity tags based on the parent intent kind:

| Intent Kind | Sender (`P`) | Recipient (`p`) | Resolved Recipient (`r`) | Resolved Sender (`R`) |
| --- | --- | --- | --- | --- |
| **5520** | **Pubkey** | Pubkey OR ConnKey | If `p` is ConnKey | N/A (Already Pubkey) |
| **5523** | **ConnKey** | **ConnKey** | If resolved | **If resolved** |

---

## Kind 35521: Identity Connection (User-Signed)

The user's public declaration that they own a LIDP account. A parameterized replaceable event (30000–39999): keyed by `pubkey + d-tag`.

- `d` tag: bare ConnectionKey — **NEVER** prefixed with `lidp:`. Including a prefix breaks `#d` relay queries from third-party clients.
- `lidp` tag: provider name (e.g., `"discord"`, `"github"`)
- `e` tag: `["e", attestationEventId, iaRelayUrl]` — 3-element reference to Kind 35522. Consumers must fetch the full event from the IA relay to verify. **Never embed the attestation JSON.**
- `expiration` tag: optional — mirrored from the Kind 35522 expiration (NIP-40) so relays can auto-prune expired events.
- `content`: always cleartext JSON — `display_name`, `picture`, `user_id`, `username`. Empty (`""`) if no profile data.
- Multiple `e` tags allowed — one per IA attesting the same connection (multi-IA stacking)

When querying relays: use `#d` filter with the bare ConnectionKey. Use `#lidp` to filter by provider.

## Kind 35522: IA Attestation (IA-Signed)

The Identity Authority's cryptographic proof that a user owns a LIDP account.

- `d` tag: bare ConnectionKey (same as Kind 35521 `d` tag)
- `p` tag: user's Nostr pubkey — **frontend MUST verify `p == outerKind35521.pubkey`** to prevent embedding attacks
- `lidp` tag: provider name
- `evidence` tag: cleartext v1 JSON string — see `identity-connection.md` Section B. Readable by anyone; enables cross-IA re-verification.
- `expiration` tag: NIP-40 unix timestamp — default 90 days, configurable via `IA_ATTESTATION_EXPIRY_DAYS`
- Content: `""` (empty)
- Signed by IA private key. Always call `verifyEvent(event)` before trusting.

## Kind 5: Attestation Revocation (NIP-09)

Published by the IA to revoke a Kind 35522. Clients performing deep verification must check for this.

- Tags: `["e", attestationEventId]`
- Content: `"revoked"`
- Signed by the same IA key that issued the Kind 35522

For full event JSON, encryption details, and security rules, see `identity-connection.md`.
