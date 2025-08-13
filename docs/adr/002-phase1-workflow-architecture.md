# ADR-002: Phase 1 Workflow Architecture

**Date:** 2025-08-13  
**Status:** Accepted  
**Deciders:** Development Team  

## Context

For the MVP (Phase 1), we need to define a simple but effective workflow that allows users to upload their CV, submit job descriptions, and receive AI-enhanced resumes tailored for specific positions. The workflow should be straightforward to implement while laying the foundation for future enhancements.

## Decision

We will implement the following Phase 1 workflow:

### Core User Journey
1. **User Registration/Login** → Firebase Authentication
2. **Upload Original CV** → Parse and store in Firebase (Storage + Firestore)
3. **Submit Job Description** → Store multiple job descriptions per user
4. **Generate Tailored Resume** → AI enhancement via n8n workflow
5. **Download Enhanced Resume** → PDF generation with optimized content

### Data Flow Architecture
```
User Action → React UI → Firebase → Cloud Function → n8n Workflow → AI Provider → Firebase → Real-time UI Update
```

### Core Entities

#### User Profile
```typescript
interface User {
  id: string;
  email: string;
  displayName?: string;
  createdAt: Date;
  lastLoginAt: Date;
}
```

#### Original CV Storage
```typescript
interface OriginalCV {
  id: string;
  userId: string;
  fileName: string;
  fileUrl: string; // Firebase Storage URL
  parsedContent: {
    personalInfo: PersonalInfo;
    experience: WorkExperience[];
    education: Education[];
    skills: string[];
    summary: string;
  };
  uploadedAt: Date;
  isActive: boolean; // Only one active CV per user
}
```

#### Job Description
```typescript
interface JobDescription {
  id: string;
  userId: string;
  title: string;
  company: string;
  description: string;
  requirements: string[];
  keywords: string[];
  uploadedAt: Date;
}
```

#### Tailored Resume
```typescript
interface TailoredResume {
  id: string;
  userId: string;
  originalCVId: string;
  jobDescriptionId: string;
  enhancedContent: CVContent;
  aiSuggestions: string[];
  matchScore: number;
  status: 'processing' | 'completed' | 'error';
  generatedAt: Date;
  downloadUrl?: string;
}
```

## Rationale

### Why This Simplified Workflow?

#### 1. **Single CV Approach**
- **Simplicity**: Users maintain one master CV that gets tailored
- **Clarity**: Reduces confusion about which CV is the "source of truth"
- **Quick Implementation**: Less complex data relationships

#### 2. **Multiple Job Descriptions**
- **Flexibility**: Users can tailor their CV for different opportunities
- **Reusability**: Job descriptions can be reused and referenced
- **Portfolio Building**: Users can track all positions they've applied for

#### 3. **AI Enhancement Focus**
- **Core Value**: The AI enhancement is the primary differentiator
- **Measurable Impact**: Users can see before/after comparisons
- **Iterative Improvement**: Easy to improve AI prompts and logic

#### 4. **Asynchronous Processing**
- **User Experience**: Users aren't blocked waiting for AI processing
- **Scalability**: Can handle multiple concurrent AI requests
- **Error Handling**: Failed requests don't break the user experience

### Workflow Triggers

#### CV Upload Process
1. User uploads PDF via drag-and-drop interface
2. File stored in Firebase Storage (`/users/{userId}/cvs/{filename}`)
3. Cloud Function triggered to parse PDF content
4. Parsed content stored in Firestore
5. UI updates with parsed information for user verification

#### Resume Tailoring Process
1. User selects active CV and target job description
2. Enhancement request created in Firestore
3. Cloud Function triggers n8n webhook
4. n8n processes AI enhancement
5. Results stored back in Firestore
6. Real-time listener updates UI
7. PDF generation triggered for download

## Consequences

### Positive
- **Fast Development**: Simple entities and relationships
- **Clear User Experience**: Linear workflow easy to understand
- **Scalable Foundation**: Can extend with additional features
- **Real-time Feedback**: Users see progress and results immediately
- **Error Recovery**: Failed operations can be retried

### Negative
- **Limited CV Management**: Only one active CV per user
- **Basic Job Tracking**: No advanced job application management
- **No Versioning**: CV changes overwrite previous versions

### Technical Debt
- **PDF Parsing**: May need more sophisticated parsing for complex CVs
- **AI Prompt Management**: Hardcoded prompts in n8n workflows
- **File Management**: No cleanup strategy for old files

## Implementation Strategy

### Phase 1.1: Core Infrastructure
- [ ] Firebase project setup with security rules
- [ ] React app with authentication flow
- [ ] Basic CV upload and storage
- [ ] n8n instance setup and basic workflow

### Phase 1.2: AI Integration
- [ ] PDF parsing implementation
- [ ] n8n workflow for CV enhancement
- [ ] Real-time UI updates
- [ ] Error handling and retry logic

### Phase 1.3: User Experience
- [ ] PDF preview and download
- [ ] Job description management
- [ ] Enhancement history and comparison
- [ ] Basic analytics and feedback

## Metrics for Success

### Technical Metrics
- CV upload success rate > 95%
- AI processing time < 30 seconds
- System uptime > 99%
- Error rate < 5%

### User Experience Metrics
- Time from upload to first tailored resume < 2 minutes
- User completion rate (upload → tailored resume) > 70%
- User satisfaction score > 4.0/5.0

## Future Considerations

### Phase 2 Enhancements
- Multiple CV versions and templates
- Advanced job application tracking
- Team collaboration features
- Integration with job boards
- A/B testing for AI prompts

### Scaling Considerations
- Database sharding strategy for large user base
- CDN implementation for file storage
- n8n cluster setup for high availability
- Monitoring and alerting system

## Review Date

This workflow should be reviewed after 100 users have completed the full journey or by 2025-10-31, whichever comes first.
