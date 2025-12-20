---
id: github-integration
alias: GitHub Integration System
type: walkthrough
is_base: false
version: 1
tags:
  - github
  - authentication
  - api-integration
description: Comprehensive guide to the GitHub integration system including OAuth authentication with PKCE, API client, keychain storage, React components, and library publishing integration
complexity: comprehensive
format: architecture
---
# GitHub Integration System

This walkthrough provides a comprehensive understanding of how BlueKit integrates with GitHub, covering authentication, API operations, token management, and the library publishing system.

## Overview

The GitHub integration enables BlueKit to:
- Authenticate users via OAuth 2.0 with PKCE
- Store tokens securely in the OS keychain
- Make authenticated API calls to GitHub
- Publish artifacts to GitHub repositories
- Fetch commit history and repository information

The system is split across three main layers:
1. **Backend (Rust)**: OAuth flow, API client, keychain management
2. **IPC Layer**: Type-safe communication between frontend and backend
3. **Frontend (React)**: UI components and state management

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Frontend (React)                           │
│  ┌──────────────────┐  ┌──────────────────┐                │
│  │ GitHubAuthProvider│  │ GitHubAuthScreen │                │
│  │ (Context)        │  │ (OAuth UI)       │                │
│  └──────────────────┘  └──────────────────┘                │
│           │                      │                           │
│           └──────────┬───────────┘                           │
│                      │ IPC                                     │
└──────────────────────┼───────────────────────────────────────┘
                       │
┌──────────────────────┼───────────────────────────────────────┐
│                      │ IPC Layer (TypeScript)                │
│  ┌──────────────────┐  ┌──────────────────┐                │
│  │  auth.ts        │  │  github.ts       │                │
│  │  (Auth IPC)     │  │  (API IPC)       │                │
│  └──────────────────┘  └──────────────────┘                │
│           │                      │                           │
└───────────┼──────────────────────┼───────────────────────────┘
            │                      │
┌───────────┼──────────────────────┼───────────────────────────┐
│           │                      │ Backend (Rust)             │
│  ┌────────▼────────┐  ┌─────────▼─────────┐                │
│  │ auth.rs         │  │ github.rs         │                │
│  │ (OAuth Flow)    │  │ (API Client)      │                │
│  └────────┬────────┘  └─────────┬─────────┘                │
│           │                      │                           │
│  ┌────────▼────────┐  ┌─────────▼─────────┐                │
│  │ oauth_server.rs │  │ keychain.rs       │                │
│  │ (Local Server)  │  │ (Token Storage)   │                │
│  └─────────────────┘  └───────────────────┘                │
└─────────────────────────────────────────────────────────────┘
```

## Authentication Flow

### OAuth 2.0 with PKCE

BlueKit uses GitHub's Authorization Code Flow with PKCE (Proof Key for Code Exchange) for secure authentication. This is the recommended method for desktop applications.

#### Flow Steps

1. **Generate PKCE Parameters**
   - Code verifier: 128-character random string
   - Code challenge: SHA256 hash of verifier (base64url encoded)
   - State: 32-character random string for CSRF protection

2. **Start Authorization**
   - Frontend calls `invokeAuthStartAuthorization()`
   - Backend generates PKCE parameters and state
   - Creates authorization URL with all parameters
   - Starts local HTTP server on localhost:8080 (or next available port)
   - Returns authorization URL to frontend

3. **User Authorization**
   - Frontend opens authorization URL in browser
   - User authorizes on GitHub
   - GitHub redirects to `http://localhost:8080/oauth/callback?code=...&state=...`

4. **OAuth Callback**
   - Local server receives callback
   - Validates state parameter
   - Emits Tauri event `oauth-callback` with code, state, verifier, and redirect_uri
   - Returns HTML success page to browser

5. **Token Exchange**
   - Frontend receives `oauth-callback` event
   - Calls `invokeAuthExchangeCode()` with code, state, verifier, and redirect_uri
   - Backend exchanges authorization code for access token
   - Stores token in OS keychain
   - Returns `AuthStatus::Authorized` with token

6. **Token Storage**
   - Token stored in OS keychain (macOS Keychain, Windows Credential Manager, Linux Secret Service)
   - Token includes: `access_token`, `token_type`, `scope`, `expires_at`

### Key Components

#### Backend: `auth.rs`

```rust
// Generates PKCE code verifier (128 chars)
pub fn generate_code_verifier() -> String

// Generates code challenge from verifier (SHA256 + base64url)
pub fn generate_code_challenge(verifier: &str) -> String

// Generates random state for CSRF protection
pub fn generate_state() -> String

// Creates authorization URL with all parameters
pub fn generate_authorization_url(port: u16, state: String) -> Result<(String, String), String>

// Exchanges authorization code for access token
pub async fn exchange_code_for_token(
    code: &str,
    code_verifier: &str,
    redirect_uri: &str,
) -> Result<AuthStatus, String>
```

#### Backend: `oauth_server.rs`

The OAuth server is a lightweight HTTP server built with Axum that:
- Listens on localhost (default port 8080, tries next available if busy)
- Handles `/oauth/callback` route
- Validates state parameter
- Emits Tauri events to communicate with frontend
- Returns user-friendly HTML responses

Key features:
- Port fallback: Tries ports 8080-8089 if default is unavailable
- State validation: Prevents CSRF attacks
- Event emission: Uses Tauri's event system for IPC
- Error handling: Returns appropriate HTML error pages

#### Frontend: `GitHubAuthProvider.tsx`

React context provider that manages authentication state:

```typescript
interface GitHubAuthContextValue {
  isAuthenticated: boolean;
  isLoading: boolean;
  user: GitHubUser | null;
  token: GitHubToken | null;
  signIn: () => Promise<void>;
  signOut: () => Promise<void>;
  refreshAuth: () => Promise<void>;
  setToken: (token: GitHubToken) => void;
  setUser: (user: GitHubUser | null) => void;
}
```

**State Management:**
- Loads token from keychain on mount
- Fetches user info from GitHub API when authenticated
- Provides authentication state to all child components
- Handles token refresh and sign-out

**Lifecycle:**
1. On mount: `loadStoredToken()` checks keychain
2. If token exists: Fetches user info from GitHub API
3. If no token: Sets `isAuthenticated = false`
4. On sign-in: Reloads token and user info
5. On sign-out: Deletes token from keychain and clears state

#### Frontend: `GitHubAuthScreen.tsx`

UI component that orchestrates the OAuth flow:

1. **Start Authorization**
   - Calls `invokeAuthStartAuthorization()` on mount
   - Receives authorization URL
   - Opens URL in default browser

2. **Listen for Callback**
   - Sets up Tauri event listener for `oauth-callback`
   - Handles both success and error cases

3. **Exchange Code**
   - Extracts code, state, verifier, and redirect_uri from event
   - Calls `invokeAuthExchangeCode()`
   - On success: Sets token via `setToken()` and calls `onSuccess()`
   - On error: Shows error toast

## GitHub API Client

### Overview

The `GitHubClient` struct provides a type-safe wrapper around GitHub's REST API. All requests are authenticated using the token stored in the keychain.

### Client Creation

```rust
// Create client with explicit token
pub fn new(token: String) -> Self

// Create client from keychain (most common)
pub fn from_keychain() -> Result<Self, String>
```

The `from_keychain()` method:
1. Creates `KeychainManager`
2. Retrieves stored token
3. Creates `GitHubClient` with token
4. Returns error if token not found

### Request Handling

All API requests go through the private `request()` method which:

1. **Builds Request**
   - Constructs URL: `https://api.github.com{endpoint}`
   - Adds headers:
     - `Authorization: Bearer {token}`
     - `Accept: application/vnd.github.v3+json`
     - `User-Agent: BlueKit/1.0`

2. **Sends Request**
   - Uses `reqwest::Client` for HTTP
   - Handles JSON body if provided
   - Supports all HTTP methods (GET, POST, PUT, DELETE)

3. **Error Handling**
   - **429 (Rate Limit)**: Extracts rate limit info, calculates reset time
   - **401 (Unauthorized)**: Returns authentication error
   - **403 (Forbidden)**: Returns permission error
   - **404 (Not Found)**: Returns not found error
   - **Other errors**: Returns status code and error text

4. **Response Parsing**
   - Deserializes JSON response to requested type
   - Returns typed result

### Available Operations

#### User Operations

```rust
// Get authenticated user info
pub async fn get_user(&self) -> Result<GitHubUser, String>

// Get user's repositories
pub async fn get_user_repos(&self) -> Result<Vec<GitHubRepo>, String>
```

#### Repository Operations

```rust
// Create new repository
pub async fn create_repo(
    &self,
    name: &str,
    description: Option<&str>,
    private: bool,
) -> Result<GitHubRepo, String>
```

#### File Operations

```rust
// Get file contents (decodes base64)
pub async fn get_file_contents(
    &self,
    owner: &str,
    repo: &str,
    path: &str,
) -> Result<String, String>

// Create or update file
pub async fn create_or_update_file(
    &self,
    owner: &str,
    repo: &str,
    path: &str,
    content: &str,
    message: &str,
    sha: Option<&str>, // Required for updates
) -> Result<GitHubFileResponse, String>

// Delete file
pub async fn delete_file(
    &self,
    owner: &str,
    repo: &str,
    path: &str,
    message: &str,
    sha: &str, // Required
) -> Result<GitHubFileResponse, String>

// Get file SHA (for checking existence)
pub async fn get_file_sha(
    &self,
    owner: &str,
    repo: &str,
    path: &str,
) -> Result<Option<String>, String>
```

**File Operations Notes:**
- Content is base64 encoded for GitHub API
- SHA is required for updates and deletes (prevents conflicts)
- Commit messages are automatically prefixed with `[BlueKit]` in some contexts

#### Tree Operations

```rust
// Get directory tree
pub async fn get_tree(
    &self,
    owner: &str,
    repo: &str,
    tree_sha: &str, // Can be branch name, commit SHA, or tree SHA
) -> Result<GitHubTreeResponse, String>
```

#### Commit Operations

```rust
// Get commits from repository
pub async fn get_commits(
    &self,
    owner: &str,
    repo: &str,
    branch: Option<&str>,
    per_page: Option<u32>, // Max 100
    page: Option<u32>,
) -> Result<Vec<GitHubCommit>, String>
```

### Data Types

The client uses strongly-typed structs for all API responses:

- `GitHubUser`: User profile information
- `GitHubRepo`: Repository metadata
- `GitHubCommit`: Commit information with author details
- `GitHubFileResponse`: File operation response with commit info
- `GitHubTreeResponse`: Directory tree structure
- `GitHubContentResponse`: File or directory content metadata

All types implement `Serialize` and `Deserialize` for JSON handling.

## Keychain Storage

### Overview

Tokens are stored securely in the OS keychain to persist across app restarts. The `KeychainManager` provides a cross-platform abstraction.

### Platform Support

- **macOS**: Uses `security` framework (Keychain Services)
- **Windows**: Uses Credential Manager API
- **Linux**: Uses Secret Service API (via `secret-service` crate)

### KeychainManager API

```rust
pub struct KeychainManager;

impl KeychainManager {
    // Create new manager instance
    pub fn new() -> Result<Self, String>
    
    // Store token in keychain
    pub fn store_token(&self, token: &GitHubToken) -> Result<(), String>
    
    // Retrieve token from keychain
    pub fn retrieve_token(&self) -> Result<GitHubToken, String>
    
    // Delete token from keychain
    pub fn delete_token(&self) -> Result<(), String>
}
```

### Token Structure

```rust
pub struct GitHubToken {
    pub access_token: String,
    pub token_type: String,      // Usually "bearer"
    pub scope: String,           // Comma-separated scopes
    pub expires_at: Option<i64>,  // Unix timestamp (None for non-expiring tokens)
}
```

### Storage Details

- **Service Name**: "BlueKit"
- **Account Name**: "GitHub Token"
- **Data**: JSON-serialized `GitHubToken`
- **Access**: Only accessible by the application

### Error Handling

- Token not found: Returns error (user not authenticated)
- Keychain access denied: Returns permission error
- Platform-specific errors: Wrapped in `String` error type

## IPC Layer

### Type Safety

The IPC layer provides type-safe wrappers around Tauri commands. All functions use TypeScript types that match Rust types exactly.

### Authentication IPC (`src/ipc/auth.ts`)

```typescript
// Start OAuth flow
invokeAuthStartAuthorization(): Promise<string>

// Exchange code for token
invokeAuthExchangeCode(
  code: string,
  state: string,
  codeVerifier: string,
  redirectUri: string
): Promise<AuthStatus>

// Get current auth status
invokeAuthGetStatus(): Promise<AuthStatus>
```

### GitHub API IPC (`src/ipc/github.ts`)

```typescript
// User operations
invokeGitHubGetUser(): Promise<GitHubUser>
invokeGitHubGetRepos(): Promise<GitHubRepo[]>

// File operations
invokeGitHubGetFile(owner: string, repo: string, path: string): Promise<string>
invokeGitHubCreateOrUpdateFile(
  owner: string,
  repo: string,
  path: string,
  content: string,
  message: string,
  sha?: string
): Promise<GitHubFileResponse>
invokeGitHubDeleteFile(
  owner: string,
  repo: string,
  path: string,
  message: string,
  sha: string
): Promise<GitHubFileResponse>
invokeGitHubGetFileSha(owner: string, repo: string, path: string): Promise<string | null>

// Tree operations
invokeGitHubGetTree(owner: string, repo: string, treeSha: string): Promise<GitHubTreeResponse>
```

### Timeout Handling

All IPC functions use `invokeWithTimeout()` which:
- Sets appropriate timeouts (5-15 seconds depending on operation)
- Provides better error messages
- Prevents hanging requests

## Frontend Integration

### React Context Pattern

The `GitHubAuthProvider` uses React Context to provide authentication state throughout the app:

```typescript
// Wrap app with provider
<GitHubAuthProvider>
  <App />
</GitHubAuthProvider>

// Use in components
const { isAuthenticated, user, signOut } = useGitHubAuth();
```

### State Flow

1. **Initial Load**
   - Provider mounts
   - Calls `loadStoredToken()`
   - If token exists: Fetches user info
   - Updates `isAuthenticated` and `user` state

2. **Sign In**
   - User initiates OAuth flow
   - `GitHubAuthScreen` handles flow
   - On success: Calls `setToken()`
   - Provider fetches user info
   - State updates propagate

3. **Sign Out**
   - User clicks sign out
   - Calls `signOut()`
   - Deletes token from keychain
   - Clears all state

4. **Token Refresh**
   - Can manually refresh with `refreshAuth()`
   - Checks keychain and fetches user info
   - Updates state if changed

### UI Components

#### `GitHubAuthScreen.tsx`

Full-screen authentication screen that:
- Shows loading state while starting auth
- Opens browser for authorization
- Shows success/error messages
- Handles OAuth callback

#### `SignInPopover.tsx`

Popover-based sign-in for inline authentication:
- Compact UI for sign-in button
- Same OAuth flow as full screen
- Closes automatically on success

### Type Definitions

All GitHub types are defined in `src/types/github.ts`:
- `GitHubToken`: Token structure
- `GitHubUser`: User information
- `GitHubRepo`: Repository information
- `GitHubCommit`: Commit information
- `AuthStatus`: Authentication status union type
- Library-related types for publishing system

## Library Publishing Integration

### Overview

The GitHub integration powers the library publishing system, allowing users to publish BlueKit artifacts (kits, walkthroughs, blueprints, etc.) to GitHub repositories.

### Publishing Flow

1. **Resource Selection**
   - User selects artifacts to publish
   - System reads file content from local filesystem
   - Calculates content hash for tracking

2. **Workspace Selection**
   - User selects or creates GitHub workspace
   - Workspace maps to a GitHub repository (owner/repo)

3. **Path Mapping**
   - Local paths mapped to remote paths in repository
   - Format: `{artifact_type}s/{filename}.md`
   - Example: `kits/my-kit.md`

4. **Conflict Detection**
   - Checks if file exists in repository
   - Gets file SHA if exists
   - Determines if update or new file

5. **Publishing**
   - Uses `GitHubClient` to create/update file
   - Commit message: `[BlueKit] Publish: {name} by {user}`
   - Stores commit SHA for tracking

6. **Catalog Management**
   - Creates/updates catalog entry in database
   - Tracks variations (different versions of same artifact)
   - Links to GitHub commit SHA

### Key Integration Points

#### Publishing Command

```rust
// In library/publishing.rs
pub async fn publish_resource(
    db: &DatabaseConnection,
    options: PublishOptions,
) -> Result<PublishResult, String> {
    // Get GitHub client from keychain
    let github_client = GitHubClient::from_keychain()?;
    
    // Get user info for commit message
    let user_info = github_client.get_user().await?;
    
    // Check if file exists
    let file_sha = github_client
        .get_file_sha(&workspace.github_owner, &workspace.github_repo, &remote_path)
        .await?;
    
    // Create or update file
    let github_response = github_client
        .create_or_update_file(
            &workspace.github_owner,
            &workspace.github_repo,
            &remote_path,
            &content,
            &commit_message,
            file_sha.as_deref(),
        )
        .await?;
    
    // Store commit SHA in database
    // ...
}
```

### Commit Fetching

The system can fetch commit history from GitHub repositories:

```rust
// In commands.rs
pub async fn fetch_commits_from_github(
    git_url: &str,
    branch: Option<&str>,
    per_page: Option<u32>,
    page: Option<u32>,
) -> Result<Vec<GitHubCommit>, String> {
    // Parse GitHub URL
    let (owner, repo) = parse_github_url(git_url)?;
    
    // Get client from keychain
    let client = GitHubClient::from_keychain()?;
    
    // Fetch commits
    client.get_commits(&owner, &repo, branch.as_deref(), per_page, page).await
}
```

This is used for:
- Displaying commit history in UI
- Linking local commits to GitHub commits
- Tracking published artifacts

## Error Handling

### Authentication Errors

- **Token not found**: User needs to sign in
- **Token expired**: User needs to re-authenticate
- **Invalid token**: Clear token and prompt re-auth
- **OAuth errors**: Show user-friendly error messages

### API Errors

- **Rate limiting**: Extract reset time, show countdown
- **401 Unauthorized**: Prompt re-authentication
- **403 Forbidden**: Check token permissions
- **404 Not Found**: Resource doesn't exist
- **Network errors**: Retry with exponential backoff (future)

### Keychain Errors

- **Access denied**: Show permission error
- **Not found**: User not authenticated
- **Platform errors**: Wrap in user-friendly messages

## Security Considerations

### PKCE

- Prevents authorization code interception attacks
- Code verifier never sent to authorization server
- Code challenge proves possession of verifier

### State Parameter

- Prevents CSRF attacks
- Random 32-character string
- Validated on callback

### Token Storage

- Stored in OS keychain (encrypted)
- Only accessible by application
- Never logged or exposed

### Token Scope

Requested scopes:
- `repo`: Full repository access
- `user`: User profile access
- `read:org`: Read organization info
- `write:org`: Write organization info
- `user:follow`: Follow/unfollow users

### HTTPS Only

- All API calls use HTTPS
- OAuth redirect uses localhost (secure)
- No tokens in URLs or logs

## Environment Variables

Required for OAuth:
- `GITHUB_CLIENT_ID`: OAuth app client ID
- `GITHUB_CLIENT_SECRET`: OAuth app client secret

These are set at build/runtime and not stored in code.

## Testing Considerations

### Mocking

- Keychain operations can be mocked for testing
- GitHub API can use test tokens
- OAuth server can be tested with local redirects

### Integration Tests

- Test full OAuth flow with test GitHub app
- Test API operations with test repository
- Test error handling for all error cases

## Future Enhancements

Potential improvements:
- Token refresh for expiring tokens
- Retry logic with exponential backoff
- Batch API operations
- Webhook support for real-time updates
- GraphQL API support
- Repository cloning via API

## Summary

The GitHub integration provides a complete, secure, and type-safe way to:
1. Authenticate users via OAuth 2.0 with PKCE
2. Store tokens securely in OS keychain
3. Make authenticated API calls
4. Publish artifacts to GitHub repositories
5. Fetch repository and commit information

The architecture separates concerns cleanly:
- **Backend**: OAuth, API client, keychain
- **IPC**: Type-safe communication
- **Frontend**: React components and state management

All components work together to provide a seamless GitHub integration experience.
