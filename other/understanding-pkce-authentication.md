---
id: understanding-pkce-authentication
alias: Understanding PKCE Authentication
type: walkthrough
is_base: false
version: 1
tags:
  - authentication
  - security
  - oauth
description: Beginner-friendly guide to PKCE (Proof Key for Code Exchange) - what it is, why it's needed, and how it works in the BlueKit GitHub OAuth flow
complexity: simple
format: guide
---
# Understanding PKCE Authentication

This walkthrough explains PKCE (Proof Key for Code Exchange) in simple terms, using the BlueKit GitHub authentication implementation as a real-world example.

## What is PKCE?

**PKCE** (pronounced "pixie") stands for **Proof Key for Code Exchange**. It's a security extension to OAuth 2.0 that makes authentication safer, especially for desktop and mobile apps.

### The Problem PKCE Solves

Imagine you're building a desktop app that needs to sign in with GitHub. Here's the challenge:

1. Your app can't keep a secret password (anyone could extract it from the app)
2. But you still need to prove you're the legitimate app
3. Traditional OAuth requires a "client secret" that must stay hidden

**PKCE solves this** by using a temporary, one-time secret that's generated on-the-fly. Even if someone intercepts it, it's useless after one use.

## The Two Key Pieces: Code Verifier and Code Challenge

PKCE uses two related values:

### 1. Code Verifier (The Secret)

The **code verifier** is a random string that your app generates and keeps secret. Think of it as a temporary password that only your app knows.

**In BlueKit's code:**
```rust
pub fn generate_code_verifier() -> String {
    use rand::Rng;
    const CHARSET: &[u8] = b"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._~";
    let mut rng = rand::thread_rng();
    (0..128)
        .map(|_| {
            let idx = rng.gen_range(0..CHARSET.len());
            CHARSET[idx] as char
        })
        .collect()
}
```

**What this does:**
- Creates a random 128-character string
- Uses only safe characters (letters, numbers, and `-._~`)
- Example result: `"aB3xK9mP2qR7sT4vW8yZ1cD5fG6hJ0kL3nM9pQ2rS7tU4vW8xY1zA5bC6dE7fG8hI9jK0"`

### 2. Code Challenge (The Public Proof)

The **code challenge** is a hashed version of the code verifier. It's safe to send publicly because you can't reverse a hash to get the original verifier.

**In BlueKit's code:**
```rust
pub fn generate_code_challenge(verifier: &str) -> String {
    let mut hasher = Sha256::new();
    hasher.update(verifier.as_bytes());
    let hash = hasher.finalize();
    URL_SAFE_NO_PAD.encode(hash)
}
```

**What this does:**
1. Takes the code verifier (the secret)
2. Hashes it using SHA256 (one-way encryption)
3. Encodes it in base64url format (URL-safe)
4. Example: `"E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"`

**Key Point:** You can't figure out the verifier from the challenge, but you can verify they match later!

## The Complete PKCE Flow

Here's how PKCE works in BlueKit's GitHub authentication:

### Step 1: Generate the Secrets

When the user clicks "Sign In", BlueKit generates:

```rust
let code_verifier = generate_code_verifier();  // Secret: "aB3xK9mP2qR7sT..."
let code_challenge = generate_code_challenge(&code_verifier);  // Public: "E9Melhoa2Owv..."
let state = generate_state();  // CSRF protection: "XyZ123AbC..."
```

**What happens:**
- `code_verifier`: Stored in memory (never sent to GitHub)
- `code_challenge`: Will be sent to GitHub
- `state`: Also sent (prevents CSRF attacks)

### Step 2: Build the Authorization URL

BlueKit creates a URL that includes the code challenge:

```rust
let url = format!(
    "https://github.com/login/oauth/authorize?client_id={}&redirect_uri={}&scope=repo,user,read:org&state={}&code_challenge={}&code_challenge_method=S256",
    client_id,
    redirect_uri,
    state,
    code_challenge  // ← The public challenge, not the secret verifier!
);
```

**Example URL:**
```
https://github.com/login/oauth/authorize?
  client_id=abc123&
  redirect_uri=http://localhost:8080/oauth/callback&
  scope=repo,user,read:org&
  state=XyZ123AbC&
  code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&
  code_challenge_method=S256
```

**Notice:**
- ✅ `code_challenge` is in the URL (safe to share)
- ❌ `code_verifier` is NOT in the URL (stays secret)

### Step 3: Store the Verifier

Before opening the browser, BlueKit stores the verifier linked to the state:

```rust
// Store verifier mapped to state
let mut state_map = oauth_state.lock().unwrap();
state_map.insert(state.clone(), code_verifier.clone());
```

**Why?** When GitHub redirects back, we'll need the verifier to prove we're legitimate.

### Step 4: User Authorizes on GitHub

1. Browser opens with the authorization URL
2. User sees GitHub's authorization page
3. User clicks "Authorize"
4. GitHub stores the code challenge (not the verifier)

### Step 5: GitHub Redirects Back

GitHub redirects to your app with an authorization code:

```
http://localhost:8080/oauth/callback?code=abc123xyz&state=XyZ123AbC
```

**Important:** This code is NOT the access token! It's just a temporary code.

### Step 6: Exchange Code for Token

Now comes the security magic. BlueKit sends BOTH:
- The authorization code (from GitHub)
- The code verifier (the secret we kept)

```rust
let params = [
    ("client_id", client_id),
    ("client_secret", client_secret),
    ("code", code.to_string()),           // From GitHub
    ("redirect_uri", redirect_uri),
    ("code_verifier", code_verifier),     // ← Our secret!
];
```

**What GitHub does:**
1. Receives the code and verifier
2. Hashes the verifier (same way we did)
3. Compares it to the challenge we sent earlier
4. If they match → GitHub knows we're legitimate!
5. Returns the access token

### Step 7: Success!

If the verifier matches the challenge, GitHub returns:

```json
{
  "access_token": "gho_abc123xyz...",
  "token_type": "bearer",
  "scope": "repo,user,read:org"
}
```

BlueKit stores this token securely in the OS keychain.

## Why This is Secure

### Scenario: Someone Intercepts the Authorization Code

**Without PKCE:**
- Attacker intercepts: `code=abc123xyz`
- Attacker could exchange it for a token
- ❌ Your account is compromised!

**With PKCE:**
- Attacker intercepts: `code=abc123xyz`
- Attacker tries to exchange it
- Attacker doesn't have the `code_verifier`
- GitHub hashes their (wrong) verifier
- It doesn't match the challenge
- ❌ Request rejected!

**The attacker can't create a valid verifier** because:
1. They don't know the original verifier
2. You can't reverse a hash to get the original
3. They can't guess a 128-character random string

## Visual Flow Diagram

```
┌─────────────┐
│   BlueKit   │
│    App      │
└──────┬──────┘
       │
       │ 1. Generate code_verifier (secret)
       │    "aB3xK9mP2qR7sT..."
       │
       │ 2. Hash it → code_challenge
       │    "E9Melhoa2Owv..."
       │
       │ 3. Store verifier in memory
       │    (linked to state)
       │
       ▼
┌─────────────────────────────────────┐
│  Build Authorization URL            │
│  (includes code_challenge)          │
└──────┬──────────────────────────────┘
       │
       │ 4. Open browser with URL
       │
       ▼
┌─────────────┐
│   GitHub    │
│   Website   │
└──────┬──────┘
       │
       │ 5. User authorizes
       │
       │ 6. GitHub stores code_challenge
       │
       │ 7. Redirect with code
       │    ?code=abc123&state=XyZ123
       │
       ▼
┌─────────────┐
│  BlueKit    │
│  OAuth      │
│  Server     │
└──────┬──────┘
       │
       │ 8. Look up verifier using state
       │
       │ 9. Exchange code + verifier
       │
       ▼
┌─────────────┐
│   GitHub    │
│    API      │
└──────┬──────┘
       │
       │ 10. GitHub verifies:
       │     - Hash verifier
       │     - Compare to stored challenge
       │     - If match → return token
       │
       ▼
┌─────────────┐
│   BlueKit   │
│    App      │
└─────────────┘
       │
       │ 11. Store token in keychain
       │
       ▼
    Success!
```

## Key Takeaways

1. **Code Verifier** = Secret random string (128 chars)
   - Generated by your app
   - Never sent to GitHub in the authorization URL
   - Stored in memory until needed

2. **Code Challenge** = Hashed version of verifier
   - Safe to send publicly
   - Included in authorization URL
   - GitHub stores it temporarily

3. **The Exchange** = Prove you have the verifier
   - Send code + verifier to GitHub
   - GitHub verifies they match
   - Only then do you get the token

4. **Security** = Even if intercepted, code is useless
   - Attacker needs the verifier
   - Verifier is never transmitted
   - Hash can't be reversed

## In BlueKit's Code

Here's where each piece lives:

- **Generate verifier**: `src-tauri/src/auth.rs` → `generate_code_verifier()`
- **Generate challenge**: `src-tauri/src/auth.rs` → `generate_code_challenge()`
- **Build URL**: `src-tauri/src/auth.rs` → `generate_authorization_url()`
- **Store verifier**: `src-tauri/src/commands.rs` → `auth_start_authorization()`
- **Exchange code**: `src-tauri/src/auth.rs` → `exchange_code_for_token()`

## Summary

PKCE is like a secret handshake:
1. You create a secret (verifier)
2. You create a public proof of that secret (challenge)
3. You show the proof to GitHub
4. Later, you prove you know the secret
5. GitHub verifies they match
6. Only then do you get access

This makes desktop app authentication secure without needing a permanent secret!
