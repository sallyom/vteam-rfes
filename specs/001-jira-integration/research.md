# Research: Jira Integration via Atlassian MCP Server

**Date**: 2025-10-07
**Feature**: 001-jira-integration

## Executive Summary

Research confirms **direct Jira REST API integration** is preferred over Atlassian MCP Server for backend artifact publishing. MCP Server is designed for interactive AI assistants, while vTeam needs automated artifact publishing after session completion.

## 1. Integration Architecture

### Decision: Direct Jira Cloud REST API v3

**Rationale**:
- MCP Server targets interactive AI tools (Claude Desktop, IDEs) with stdio-based communication
- Direct REST API provides better control, reliability, and performance for backend automation
- Simpler implementation with fewer moving parts
- Production-ready Go libraries available

**Alternatives Considered**:
- Via Atlassian MCP Server: Adds unnecessary proxy layer, designed for different use case
- Custom MCP client: Over-engineering for artifact publishing requirements

**Implementation Notes**:
- Use `github.com/andygrunwald/go-jira` or `github.com/ctreminiom/go-atlassian`
- Official MCP Go SDK (`github.com/modelcontextprotocol/go-sdk`) available for future AI features
- REST API v3 endpoint: `https://{domain}.atlassian.net/rest/api/3/`

## 2. Authentication

### Decision: OAuth 2.0 Authorization Code Grant (3LO)

**Rationale**:
- More secure than API tokens with granular permission scopes
- Rotating refresh tokens (90-day validity)
- Supports TLS 1.2+ encryption
- Atlassian-supported authentication standard

**Required Scopes**:
- `write:jira-work` (covers write:issue, write:comment, write:attachment)
- `read:jira-work` (covers read:issue)
- `read:me` (for user identity)

**Alternatives Considered**:
- API Tokens: Less secure, no granular permissions
- Via MCP Server: Requires Node.js proxy, managed in ~/.mcp-auth/, adds complexity

**Implementation Notes**:
- Access tokens: 1-hour lifetime
- Refresh tokens: 90-day validity with rotation
- Critical: Always replace stored refresh token with new token from refresh response
- Authorization endpoint: `https://auth.atlassian.com/authorize`
- Token endpoint: `https://auth.atlassian.com/oauth/token`
- Store tokens encrypted in secrets management (Vault, environment variables)

## 3. Publishing Methods

### Decision: Attachments (Primary) + Comments (Secondary)

**Rationale**:
- Attachments preserve markdown files as-is for download
- Comments provide visibility and audit trail
- Description updates risk overwriting important content

**Publishing Strategy**:
1. Upload spec.md, tasks.md, plan.md as attachments via multipart/form-data
2. Add comment summarizing published artifacts in Atlassian Document Format (ADF)
3. Optionally update description for new issues only (configurable)

**API Endpoints**:
- Attachments: `POST /rest/api/3/issue/{issueKey}/attachments`
- Comments: `POST /rest/api/3/issue/{issueKey}/comment`
- Description: `PUT /rest/api/3/issue/{issueKey}` (update field)

**Alternatives Considered**:
- Comments only: Loses downloadable files
- Description only: Destructive to existing content
- Custom fields: Complex configuration, field mapping required

**Implementation Notes**:
- Attachment header: `X-Atlassian-Token: no-check` (CSRF protection)
- Multipart parameter: `file` (required name)
- Multiple files: Send multiple `file` parameters in single request
- Size limits: 10MB recommended per attachment, 1GB max per file
- Naming convention: `spec-{session-id}.md`, `tasks-{session-id}.md`, `plan-{session-id}.md`

## 4. Markdown to Jira Format

### Decision: Use summonio/markdown-to-adf for Comments

**Rationale**:
- Jira Cloud v3 requires Atlassian Document Format (ADF) for comments/descriptions
- Attachments preserve markdown without conversion
- Go-native library avoids cross-language dependencies
- Based on goldmark parser (most popular Go markdown library)

**Conversion Strategy**:
- Attachments: Upload raw markdown files (no conversion)
- Comments: Convert summary to ADF using `github.com/summonio/markdown-to-adf`
- Fallback: Wrap plain text in ADF paragraph if conversion fails

**Alternatives Considered**:
- JavaScript libraries (marklassian, @atlaskit): Require Node.js subprocess
- Manual ADF construction: Error-prone, not scalable
- No conversion: Jira would render as plain text

**Implementation Notes**:
```go
import (
    "github.com/summonio/markdown-to-adf/renderer"
    "github.com/yuin/goldmark"
)

md := goldmark.New(goldmark.WithRenderer(renderer.NewRenderer()))
var buf bytes.Buffer
md.Convert([]byte(markdown), &buf)
```

## 5. Rate Limiting

### Decision: Exponential Backoff with Jitter + Client-Side Throttling

**Rationale**:
- Atlassian enforcing stricter rate limits in 2025 (August-November rollout)
- Hybrid model: Concurrency limits + cost-based limits
- `429 Too Many Requests` responses may not include `Retry-After` header

**Rate Limit Strategy**:
1. Monitor for 429 responses
2. Check `Retry-After` header if present, else use exponential backoff
3. Implement jitter (randomization) to prevent thundering herd
4. Max 5 retries with max delay of 64 seconds
5. Client-side rate limiting: Spread batch operations over time

**Best Practices**:
- Batch attachments in single request (3 files = 1 API call)
- Specify exact fields needed: `?fields=key,summary,status`
- Cache issue metadata
- Future: Subscribe to webhooks to reduce polling

**Alternatives Considered**:
- No retry logic: Poor UX for transient errors
- Fixed delay: Doesn't adapt to server state
- Unlimited retries: Could loop forever

**Implementation Notes**:
- Base delay: 1 second
- Formula: `delay = base * 2^attempt + random(0, base * 0.3)`
- Max delay cap: 64 seconds
- Retry on: 429, 5xx errors
- Don't retry: 4xx errors (except 401 after token refresh)

## 6. Error Handling

### Decision: Status-Based Classification with User-Friendly Messages

**Rationale**:
- Users need actionable error messages without technical jargon
- Some errors (401, 429, 5xx) can be recovered automatically
- Developers need detailed logs for troubleshooting

**Error Categories**:

| Status | Type | Recovery | User Action |
|--------|------|----------|-------------|
| 401 | Authentication expired | Auto refresh token | Re-authenticate if refresh fails |
| 403 | Permission denied | None | Contact Jira admin |
| 404 | Issue not found | None | Verify issue key |
| 400 | Bad request | None | Check configuration |
| 429 | Rate limit | Auto retry with backoff | Wait (automatic) |
| 5xx | Server error | Auto retry 3x | Queue for background retry |
| Network | Connectivity | Auto retry 3x | Check network |
| 413 | Payload too large | None | Reduce artifact size |

**Error Response Structure**:
```go
type UserError struct {
    Message   string  // User-friendly message
    Details   string  // More context
    Action    string  // What user should do
    Code      string  // Error code for docs
    Technical string  // Technical details (logged only)
    Retryable bool    // Can retry?
}
```

**Alternatives Considered**:
- Generic messages: Poor UX, no actionable guidance
- Raw Jira errors: Too technical
- Silent failures: User unaware of issues

**Implementation Notes**:
- Log technical details with context (session_id, issue_key, user_id)
- Show simplified messages to users with action buttons
- Mark failed operations for background retry
- Send notifications for persistent failures
- Monitor error rates and alert on thresholds

## 7. Size Limits and Constraints

### Jira Cloud Limits:
- **Attachment size**: 1GB default, 2GB maximum per file
- **Performance guardrails**: 10MB per attachment (recommended), 3000 attachments per issue
- **Comment length**: ~32,000 characters (ADF JSON)
- **Request timeout**: 30 seconds recommended

### Implementation Constraints:
- Validate artifact size before upload
- Show clear error if exceeds 10MB (configurable warning threshold)
- Offer selective publishing (e.g., skip plan.md if too large)
- Truncate comments if ADF exceeds length limit

## 8. Security Considerations

### OAuth Token Storage:
- Store refresh tokens encrypted
- Never log tokens
- Use secure secrets management (Vault, AWS Secrets Manager, encrypted env vars)
- Implement token rotation tracking

### Communication:
- HTTPS only (TLS 1.2+)
- Validate SSL certificates
- No credential transmission in query parameters

### Permissions:
- Principle of least privilege: Request only needed scopes
- Jira permissions still apply (scopes don't override)
- User-specific OAuth (not shared service account) for audit trail

## Implementation Phases

### Phase 1 - MVP:
1. OAuth 2.0 authentication flow with token refresh
2. Attachment upload (spec.md, tasks.md, plan.md)
3. Basic error handling (401, 403, 404, 429)
4. Success/failure notifications

### Phase 2 - Enhanced:
1. Comment posting with markdown→ADF conversion
2. Background retry queue for transient failures
3. Audit logging
4. Client-side rate limiting
5. Admin monitoring dashboard

### Phase 3 - Advanced:
1. Webhook subscriptions (reduce polling)
2. Bulk publishing (multiple sessions)
3. Description updates (optional, configurable)
4. Advanced error recovery
5. Performance optimization (caching, connection pooling)

## Key Dependencies

### Go Libraries:
- `github.com/andygrunwald/go-jira` - Jira REST API client
- `github.com/summonio/markdown-to-adf` - Markdown to ADF converter
- `github.com/yuin/goldmark` - Markdown parser (dependency of markdown-to-adf)
- `golang.org/x/oauth2` - OAuth 2.0 client

### External Services:
- Atlassian Cloud (Jira)
- OAuth 2.0 endpoints (auth.atlassian.com)

### Configuration Requirements:
- OAuth client ID and secret (from Atlassian Developer Console)
- OAuth callback URL configuration
- Secrets storage (encrypted)

## Testing Approach

### Unit Tests:
- OAuth token refresh logic
- Error classification and handling
- Markdown to ADF conversion
- Rate limit backoff calculation

### Integration Tests:
- End-to-end artifact publishing with test Jira instance
- OAuth flow with mock provider
- Retry mechanism with simulated failures
- Rate limit handling with test rate limiter

### Contract Tests:
- Validate request/response formats against Jira API specification
- Test all error response codes
- Verify attachment upload multipart format

### Manual Testing:
- Test with various artifact sizes (small, medium, large, oversized)
- Test with different Jira permission scenarios
- Test rate limiting under load
- Test network failure recovery

## Documentation Requirements

1. **Setup Guide**: OAuth app creation with screenshots
2. **User Guide**: Linking sessions to Jira issues
3. **Admin Guide**: Secrets configuration, monitoring
4. **Troubleshooting**: Common errors and resolutions
5. **API Reference**: Backend endpoints for UI integration
6. **Architecture Diagrams**: OAuth flow, publishing workflow

## References

- Jira Cloud REST API v3: https://developer.atlassian.com/cloud/jira/platform/rest/v3/
- OAuth 2.0 3LO: https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/
- Atlassian Document Format: https://developer.atlassian.com/cloud/jira/platform/apis/document/structure/
- Rate Limiting: https://developer.atlassian.com/cloud/jira/platform/rate-limiting/
- go-jira Library: https://github.com/andygrunwald/go-jira
- markdown-to-adf: https://github.com/summonio/markdown-to-adf
