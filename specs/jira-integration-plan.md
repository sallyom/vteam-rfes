# Implementation Plan: Jira Integration Restoration

## System Architecture

### Component Diagram
```
┌─────────────────────────────────────────────────────────────┐
│                         Frontend                             │
├─────────────────────────────────────────────────────────────┤
│  WorkspaceTab    JiraUploadDialog    JiraSyncIndicator      │
│       │                │                    │                │
│       └────────────────┴────────────────────┘                │
│                         │                                     │
│                   JiraAPIService                              │
└─────────────────────┬───────────────────────────────────────┘
                      │ HTTP/REST
┌─────────────────────┴───────────────────────────────────────┐
│                      Backend API                              │
├─────────────────────────────────────────────────────────────┤
│   SessionHandler    JiraHandler    ContentHandler            │
│        │               │                │                     │
│        └───────────────┴────────────────┘                    │
│                        │                                      │
│                  JiraIntegration                              │
│                        │                                      │
│                 ┌──────┴──────┬──────────┐                  │
│           JiraClient    K8sClient   FileSystem               │
└───────────────┬────────────┬──────────┬────────────────────┘
                │            │          │
            Jira API    Kubernetes   Storage

```

## Technical Design

### Backend Implementation

#### 1. Jira Integration Service
```go
// components/backend/jira/session_integration.go
package jira

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
)

type SessionJiraService struct {
    k8sClient     kubernetes.Interface
    dynamicClient dynamic.Interface
}

type UploadRequest struct {
    ArtifactPath string `json:"artifactPath"`
    IssueKey     string `json:"issueKey,omitempty"`  // Empty for new issue
    IssueType    string `json:"issueType"`
    Summary      string `json:"summary"`
    Description  string `json:"description"`
}

type JiraIssue struct {
    Key     string `json:"key"`
    Self    string `json:"self"`
    Summary string `json:"summary"`
}

func (s *SessionJiraService) UploadArtifact(ctx context.Context, sessionName string, req UploadRequest) (*JiraIssue, error) {
    // 1. Get Jira credentials from K8s secrets
    // 2. Read artifact content from filesystem
    // 3. Convert markdown to Jira wiki format
    // 4. Create or update Jira issue
    // 5. Update session CRD with jiraLinks
    return issue, nil
}
```

#### 2. API Handlers
```go
// components/backend/handlers/jira_session.go
func UploadArtifactToJira(c *gin.Context) {
    projectName := c.Param("projectName")
    sessionName := c.Param("sessionName")

    var req jira.UploadRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }

    service := getJiraService(c)
    issue, err := service.UploadArtifact(c.Request.Context(), sessionName, req)
    if err != nil {
        c.JSON(500, gin.H{"error": err.Error()})
        return
    }

    c.JSON(200, issue)
}
```

### Frontend Implementation

#### 1. Jira API Service
```typescript
// components/frontend/src/services/api/jira.ts
import { apiClient } from './client';

export interface JiraUploadRequest {
  artifactPath: string;
  issueKey?: string;
  issueType: string;
  summary: string;
  description: string;
}

export interface JiraIssue {
  key: string;
  self: string;
  summary: string;
}

export async function uploadArtifactToJira(
  projectName: string,
  sessionName: string,
  request: JiraUploadRequest
): Promise<JiraIssue> {
  return apiClient.post(
    `/projects/${projectName}/sessions/${sessionName}/jira/upload`,
    request
  );
}

export async function getJiraSyncStatus(
  projectName: string,
  sessionName: string
): Promise<Record<string, JiraIssue>> {
  return apiClient.get(
    `/projects/${projectName}/sessions/${sessionName}/jira/status`
  );
}
```

#### 2. UI Components
```tsx
// components/frontend/src/components/jira/JiraUploadDialog.tsx
import React, { useState } from 'react';
import { Dialog, DialogContent, DialogHeader, DialogTitle } from '@/components/ui/dialog';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { uploadArtifactToJira } from '@/services/api/jira';

interface JiraUploadDialogProps {
  open: boolean;
  onClose: () => void;
  artifactPath: string;
  projectName: string;
  sessionName: string;
  onSuccess: (issue: JiraIssue) => void;
}

export function JiraUploadDialog({
  open,
  onClose,
  artifactPath,
  projectName,
  sessionName,
  onSuccess
}: JiraUploadDialogProps) {
  const [summary, setSummary] = useState('');
  const [description, setDescription] = useState('');
  const [issueKey, setIssueKey] = useState('');
  const [loading, setLoading] = useState(false);

  const handleUpload = async () => {
    setLoading(true);
    try {
      const issue = await uploadArtifactToJira(projectName, sessionName, {
        artifactPath,
        summary,
        description,
        issueKey: issueKey || undefined,
        issueType: 'Story'
      });
      onSuccess(issue);
      onClose();
    } catch (error) {
      console.error('Failed to upload to Jira:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <Dialog open={open} onOpenChange={onClose}>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>Upload to Jira</DialogTitle>
        </DialogHeader>
        <div className="space-y-4">
          <div>
            <Label>Artifact Path</Label>
            <Input value={artifactPath} disabled />
          </div>
          <div>
            <Label>Issue Key (optional)</Label>
            <Input
              value={issueKey}
              onChange={(e) => setIssueKey(e.target.value)}
              placeholder="Leave empty to create new issue"
            />
          </div>
          <div>
            <Label>Summary</Label>
            <Input
              value={summary}
              onChange={(e) => setSummary(e.target.value)}
              placeholder="Issue summary"
            />
          </div>
          <div>
            <Label>Description</Label>
            <textarea
              className="w-full min-h-[100px] p-2 border rounded"
              value={description}
              onChange={(e) => setDescription(e.target.value)}
              placeholder="Issue description"
            />
          </div>
          <div className="flex justify-end gap-2">
            <Button variant="outline" onClick={onClose}>
              Cancel
            </Button>
            <Button onClick={handleUpload} disabled={loading}>
              {loading ? 'Uploading...' : 'Upload'}
            </Button>
          </div>
        </div>
      </DialogContent>
    </Dialog>
  );
}
```

## Database Schema Updates

### Session CRD Extension
```yaml
apiVersion: ambient-agent.redhat.com/v1alpha1
kind: AgenticSession
spec:
  # ... existing fields ...
  jiraLinks:
    - artifactPath: "specs/feature-1.md"
      issueKey: "PROJ-123"
      issueUrl: "https://jira.example.com/browse/PROJ-123"
      syncedAt: "2024-01-15T10:30:00Z"
```

## API Specifications

### REST Endpoints

#### Upload Artifact to Jira
```
POST /api/projects/{projectName}/sessions/{sessionName}/jira/upload
Content-Type: application/json

{
  "artifactPath": "specs/feature.md",
  "issueKey": "PROJ-123",  // Optional
  "issueType": "Story",
  "summary": "Feature Specification",
  "description": "Detailed feature description"
}

Response: 200 OK
{
  "key": "PROJ-123",
  "self": "https://jira.example.com/rest/api/2/issue/12345",
  "summary": "Feature Specification"
}
```

#### Get Jira Sync Status
```
GET /api/projects/{projectName}/sessions/{sessionName}/jira/status

Response: 200 OK
{
  "specs/feature.md": {
    "key": "PROJ-123",
    "self": "https://jira.example.com/rest/api/2/issue/12345",
    "summary": "Feature Specification",
    "syncedAt": "2024-01-15T10:30:00Z"
  }
}
```

## Security Design

### Authentication Flow
```
1. User configures Jira credentials in Project Settings
2. Credentials stored in K8s Secret (ambient-non-vertex-integrations)
3. Backend retrieves credentials per request
4. Uses Basic Auth with email:token for Jira API
5. No credentials exposed to frontend
```

### Authorization Matrix
| Action | Required Permission |
|--------|-------------------|
| Upload to Jira | Project Write |
| View Jira Status | Project Read |
| Configure Jira | Project Admin |

## Testing Strategy

### Unit Tests
- Jira client methods
- Markdown to Wiki conversion
- Session update logic
- API handlers

### Integration Tests
- End-to-end upload flow
- Credential retrieval
- Error scenarios
- Network failures

### E2E Tests
```typescript
describe('Jira Integration', () => {
  it('should upload artifact to new Jira issue', async () => {
    // 1. Create test artifact
    // 2. Open upload dialog
    // 3. Fill form
    // 4. Submit
    // 5. Verify sync status
  });

  it('should update existing Jira issue', async () => {
    // 1. Upload initial version
    // 2. Modify artifact
    // 3. Upload update
    // 4. Verify issue updated
  });
});
```

## Performance Considerations

### Optimization Strategies
1. **Lazy Loading**: Load Jira status only when artifacts tab is visible
2. **Caching**: Cache Jira issue data for 5 minutes
3. **Batch Operations**: Support bulk uploads in single API call
4. **Streaming**: Stream large files to Jira
5. **Background Sync**: Queue uploads for retry on failure

### Performance Targets
- Upload latency: < 3 seconds for files up to 1MB
- Status refresh: < 500ms
- UI responsiveness: No blocking during upload

## Rollout Plan

### Phase 1: Backend Foundation (Days 1-3)
- [ ] Uncomment and refactor jira/integration.go
- [ ] Create session-based handlers
- [ ] Add routing configuration
- [ ] Test with Postman/curl

### Phase 2: Core UI (Days 4-6)
- [ ] Create JiraUploadDialog component
- [ ] Add context menu to WorkspaceTab
- [ ] Implement Jira API service
- [ ] Basic error handling

### Phase 3: Enhanced Features (Days 7-9)
- [ ] Sync status indicators
- [ ] Bulk operations
- [ ] View in Jira functionality
- [ ] Polish UI/UX

### Phase 4: Testing & Documentation (Days 10-12)
- [ ] Write unit tests
- [ ] Integration testing
- [ ] User documentation
- [ ] Admin guide

## Monitoring & Observability

### Metrics to Track
- Upload success rate
- Average upload time
- Jira API errors
- User adoption rate

### Logging Strategy
```go
log.WithFields(log.Fields{
    "session": sessionName,
    "artifact": artifactPath,
    "jiraKey": issueKey,
    "duration": time.Since(start),
}).Info("Artifact uploaded to Jira")
```

### Alerts
- Jira API authentication failures
- Upload failure rate > 10%
- Response time > 5 seconds

## Migration Strategy

### From Old RFE System
1. Identify existing Jira links in RFE workflows
2. Migrate to session jiraLinks field
3. Update UI to use new endpoints
4. Deprecate old endpoints (3 month grace period)

## Documentation Requirements

### User Documentation
- How to configure Jira integration
- Uploading artifacts guide
- Troubleshooting common issues

### Developer Documentation
- API reference
- Architecture overview
- Extension points

### Admin Documentation
- Security configuration
- Performance tuning
- Monitoring setup