# ADR-003: n8n Integration Architecture

**Date:** 2025-08-13  
**Status:** Accepted  
**Deciders:** Development Team  

## Context

We need to design how n8n will integrate with our Firebase backend to handle AI-powered resume enhancement. The integration should be secure, scalable, and maintainable while providing clear error handling and monitoring capabilities.

## Decision

We will implement n8n as an external workflow orchestrator that communicates with Firebase through webhooks and REST APIs, triggered by Firebase Cloud Functions.

### Architecture Overview

```
Firebase Functions → n8n Webhook → AI Processing → Firebase REST API → Firestore Update → Real-time UI
```

### Integration Components

#### 1. Firebase Cloud Function Trigger
```typescript
// Trigger n8n workflow when enhancement request is created
export const triggerResumeEnhancement = functions.firestore
  .document('enhancement_requests/{requestId}')
  .onCreate(async (snap, context) => {
    const request = snap.data();
    
    // Call n8n webhook
    const n8nResponse = await fetch(process.env.N8N_WEBHOOK_URL, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${process.env.N8N_WEBHOOK_TOKEN}`
      },
      body: JSON.stringify({
        requestId: context.params.requestId,
        userId: request.userId,
        originalCVId: request.originalCVId,
        jobDescriptionId: request.jobDescriptionId,
        webhookSecret: process.env.WEBHOOK_SECRET
      })
    });
    
    return n8nResponse;
  });
```

#### 2. n8n Workflow Structure

##### Main Enhancement Workflow Nodes:

1. **Webhook Trigger Node**
   - Receives requests from Firebase Functions
   - Validates webhook secret
   - Extracts request parameters

2. **Firebase Data Retrieval Node**
   - Fetches CV content from Firestore
   - Fetches job description from Firestore
   - Validates data integrity

3. **Content Preprocessing Node**
   - Cleans and structures CV content
   - Extracts key information from job description
   - Prepares data for AI processing

4. **AI Enhancement Node (OpenAI)**
   - Sends structured prompt to GPT-4
   - Processes AI response
   - Validates output format

5. **Post-processing Node**
   - Formats enhanced content
   - Generates improvement suggestions
   - Calculates match score

6. **Firebase Update Node**
   - Updates Firestore with enhanced content
   - Triggers PDF generation if needed
   - Updates request status

7. **Error Handling Node**
   - Logs errors to Firebase
   - Sends error notifications
   - Updates request status to 'error'

#### 3. AI Processing Configuration

##### OpenAI Node Settings
```json
{
  "model": "gpt-4-turbo",
  "maxTokens": 3000,
  "temperature": 0.3,
  "systemMessage": "You are an expert resume writer and career coach specializing in ATS optimization and industry-specific resume tailoring.",
  "userMessage": "{{$json.enhancementPrompt}}"
}
```

##### Enhancement Prompt Template
```
TASK: Enhance the provided resume to better match the job description while maintaining accuracy.

ORIGINAL RESUME:
{{$json.originalCV}}

JOB DESCRIPTION:
{{$json.jobDescription}}

REQUIREMENTS:
1. Keep all factual information accurate (dates, companies, education)
2. Emphasize relevant skills and experiences
3. Add industry-specific keywords from the job description
4. Optimize for ATS systems
5. Improve action verbs and quantify achievements where possible
6. Maintain professional tone and formatting

RESPONSE FORMAT:
Return a JSON object with:
{
  "enhancedCV": { /* enhanced resume content */ },
  "suggestions": [/* array of improvement suggestions */],
  "keywordsAdded": [/* array of keywords added */],
  "matchScore": /* percentage match score */
}
```

#### 4. Data Security and Validation

##### Input Validation
- Verify webhook secret matches
- Validate user ownership of CV and job description
- Check file size limits and content type
- Sanitize input data

##### Output Validation
- Verify AI response format
- Check for inappropriate content
- Validate enhanced content structure
- Ensure no sensitive data leakage

#### 5. Error Handling Strategy

##### Retry Logic
```javascript
// n8n Error Handling Node
if ({{$json.error}}) {
  const retryCount = {{$json.retryCount}} || 0;
  
  if (retryCount < 3) {
    // Retry with exponential backoff
    return {
      "retry": true,
      "retryCount": retryCount + 1,
      "delay": Math.pow(2, retryCount) * 1000
    };
  } else {
    // Mark as failed and notify
    return {
      "status": "failed",
      "error": {{$json.error}},
      "finalAttempt": true
    };
  }
}
```

##### Error Categories
1. **Validation Errors**: Invalid input data
2. **AI Provider Errors**: OpenAI API failures
3. **Firebase Errors**: Database write failures
4. **Timeout Errors**: Processing takes too long
5. **Authentication Errors**: Invalid tokens or permissions

## Rationale

### Why External n8n Instance?
- **Workflow Visibility**: Visual workflow management and debugging
- **AI Provider Flexibility**: Easy to switch between AI providers
- **Complex Logic**: Better suited for multi-step AI processing
- **Monitoring**: Built-in execution logs and metrics
- **Scalability**: Can handle multiple concurrent workflows

### Why Firebase Functions as Trigger?
- **Security**: Proper authentication and authorization
- **Integration**: Native Firebase ecosystem integration
- **Scalability**: Automatic scaling with demand
- **Monitoring**: Integrated with Firebase monitoring

### Why Webhook Communication?
- **Decoupling**: n8n and Firebase remain independent
- **Reliability**: HTTP-based communication with retry logic
- **Flexibility**: Easy to modify either system independently

## Consequences

### Positive
- **Clear Separation**: AI logic separated from application logic
- **Visual Debugging**: Easy to troubleshoot workflow issues
- **Flexible AI Integration**: Can easily add new AI providers
- **Scalable Processing**: n8n handles concurrent requests efficiently
- **Monitoring**: Built-in execution tracking and error logging

### Negative
- **Additional Complexity**: External service adds operational overhead
- **Network Dependency**: Requires reliable network connectivity
- **Latency**: Additional network hops increase processing time
- **Cost**: n8n Cloud subscription or self-hosting costs

### Security Considerations
- **Webhook Security**: Shared secrets for authentication
- **Data in Transit**: HTTPS encryption required
- **Rate Limiting**: Prevent abuse of AI endpoints
- **Input Sanitization**: Prevent injection attacks

## Implementation Plan

### Phase 1: Basic Integration
- [ ] Set up n8n Cloud instance
- [ ] Create basic enhancement workflow
- [ ] Implement Firebase Function trigger
- [ ] Add basic error handling

### Phase 2: Enhanced Features
- [ ] Add retry logic and error recovery
- [ ] Implement comprehensive logging
- [ ] Add workflow monitoring dashboard
- [ ] Optimize AI prompts based on results

### Phase 3: Production Readiness
- [ ] Implement rate limiting
- [ ] Add security scanning
- [ ] Set up alerting and monitoring
- [ ] Performance optimization

## Configuration

### Environment Variables
```bash
# Firebase Functions
N8N_WEBHOOK_URL=https://your-n8n-instance.app/webhook/enhance-resume
N8N_WEBHOOK_TOKEN=your-webhook-token
WEBHOOK_SECRET=your-webhook-secret

# n8n Instance
OPENAI_API_KEY=your-openai-key
FIREBASE_PROJECT_ID=your-project-id
FIREBASE_ADMIN_KEY=your-admin-key
WEBHOOK_SECRET=your-webhook-secret
```

### Firebase Security Rules
```javascript
// Firestore rules for enhancement requests
match /enhancement_requests/{requestId} {
  allow create: if request.auth != null && 
    request.auth.uid == resource.data.userId;
  allow read: if request.auth != null && 
    request.auth.uid == resource.data.userId;
}

match /tailored_resumes/{resumeId} {
  allow read: if request.auth != null && 
    request.auth.uid == resource.data.userId;
}
```

## Monitoring and Metrics

### Key Metrics to Track
- **Processing Time**: Time from trigger to completion
- **Success Rate**: Percentage of successful enhancements
- **Error Rate**: Categorized by error type
- **AI Provider Costs**: Track OpenAI API usage
- **User Satisfaction**: Quality of enhanced content

### Alerting Rules
- Error rate > 5% in 5-minute window
- Average processing time > 60 seconds
- AI provider API errors > 10 in 1 hour
- Webhook endpoint downtime

## Review Date

This integration architecture should be reviewed after processing 1000 enhancement requests or by 2025-11-30, whichever comes first.
