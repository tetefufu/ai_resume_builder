# ADR-004: Data Storage and Security Architecture

**Date:** 2025-08-13  
**Status:** Accepted  
**Deciders:** Development Team  

## Context

We need to design a secure and scalable data storage architecture for handling sensitive user information including CVs, personal data, and job descriptions. The system must comply with data protection regulations while providing fast access and real-time updates.

## Decision

We will use Firebase Firestore as the primary database with Firebase Storage for file management, implementing a multi-layered security approach with strict access controls and data encryption.

### Database Architecture

#### Firestore Collections Structure

```
/users/{userId}
  ├── email: string
  ├── displayName: string
  ├── createdAt: timestamp
  ├── lastLoginAt: timestamp
  ├── preferences: object
  └── subscription: object

/original_cvs/{cvId}
  ├── userId: string
  ├── fileName: string
  ├── fileUrl: string
  ├── fileSize: number
  ├── mimeType: string
  ├── parsedContent: object
  ├── uploadedAt: timestamp
  ├── isActive: boolean
  └── metadata: object

/job_descriptions/{jobId}
  ├── userId: string
  ├── title: string
  ├── company: string
  ├── description: string
  ├── requirements: array
  ├── keywords: array
  ├── uploadedAt: timestamp
  └── metadata: object

/enhancement_requests/{requestId}
  ├── userId: string
  ├── originalCVId: string
  ├── jobDescriptionId: string
  ├── status: string ('pending' | 'processing' | 'completed' | 'error')
  ├── createdAt: timestamp
  ├── startedAt: timestamp
  ├── completedAt: timestamp
  └── error: object

/tailored_resumes/{resumeId}
  ├── userId: string
  ├── originalCVId: string
  ├── jobDescriptionId: string
  ├── enhancedContent: object
  ├── aiSuggestions: array
  ├── matchScore: number
  ├── status: string
  ├── generatedAt: timestamp
  ├── downloadUrl: string
  └── metadata: object

/user_analytics/{userId}/sessions/{sessionId}
  ├── startTime: timestamp
  ├── endTime: timestamp
  ├── actions: array
  ├── enhancementsGenerated: number
  └── metadata: object
```

#### Firebase Storage Structure

```
/users/{userId}/
  ├── cvs/
  │   ├── original/{fileName}
  │   └── enhanced/{resumeId}.pdf
  ├── uploads/
  │   └── {fileId}
  └── exports/
      └── {exportId}.pdf
```

### Security Implementation

#### Firebase Security Rules

##### Firestore Rules
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Users can only access their own data
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
    
    // CV access control
    match /original_cvs/{cvId} {
      allow read, write: if request.auth != null && 
        request.auth.uid == resource.data.userId;
      allow create: if request.auth != null && 
        request.auth.uid == request.resource.data.userId;
    }
    
    // Job descriptions access control
    match /job_descriptions/{jobId} {
      allow read, write: if request.auth != null && 
        request.auth.uid == resource.data.userId;
      allow create: if request.auth != null && 
        request.auth.uid == request.resource.data.userId;
    }
    
    // Enhancement requests
    match /enhancement_requests/{requestId} {
      allow create: if request.auth != null && 
        request.auth.uid == request.resource.data.userId;
      allow read: if request.auth != null && 
        request.auth.uid == resource.data.userId;
      // Only allow status updates from Cloud Functions
      allow update: if request.auth.token.admin == true;
    }
    
    // Tailored resumes
    match /tailored_resumes/{resumeId} {
      allow read: if request.auth != null && 
        request.auth.uid == resource.data.userId;
      // Only allow creation/updates from Cloud Functions
      allow create, update: if request.auth.token.admin == true;
    }
    
    // Analytics (read-only for users)
    match /user_analytics/{userId}/{document=**} {
      allow read: if request.auth != null && request.auth.uid == userId;
      allow write: if request.auth.token.admin == true;
    }
  }
}
```

##### Storage Rules
```javascript
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    // Users can only access their own files
    match /users/{userId}/{allPaths=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
    
    // File size and type restrictions
    match /users/{userId}/cvs/original/{fileName} {
      allow write: if request.auth != null && 
        request.auth.uid == userId &&
        resource.size < 10 * 1024 * 1024 && // 10MB limit
        resource.contentType.matches('application/pdf');
    }
  }
}
```

#### Data Encryption Strategy

##### Client-Side Encryption (Optional)
```typescript
// For highly sensitive data, encrypt before storing
interface EncryptedField {
  value: string; // AES encrypted value
  iv: string;    // Initialization vector
  tag: string;   // Authentication tag
}

const encryptSensitiveData = (data: string, key: string): EncryptedField => {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipher('aes-256-gcm', key);
  
  let encrypted = cipher.update(data, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  
  return {
    value: encrypted,
    iv: iv.toString('hex'),
    tag: cipher.getAuthTag().toString('hex')
  };
};
```

##### Server-Side Encryption
- Firebase automatically encrypts data at rest
- All data transmitted over HTTPS/TLS
- Firebase Storage uses Google Cloud's encryption

#### Access Control Matrix

| Resource | User (Owner) | User (Other) | Cloud Function | n8n Webhook |
|----------|-------------|--------------|----------------|-------------|
| User Profile | Read/Write | None | Read | None |
| Original CV | Read/Write | None | Read | Read* |
| Job Descriptions | Read/Write | None | Read | Read* |
| Enhancement Requests | Create/Read | None | Read/Write | None |
| Tailored Resumes | Read | None | Create/Write | None |
| Storage Files | Read/Write | None | Read/Write | None |

*Only through authenticated Cloud Function calls

### Data Lifecycle Management

#### Retention Policies
```typescript
interface DataRetentionPolicy {
  originalCVs: '2 years after last access';
  jobDescriptions: '1 year after creation';
  tailoredResumes: '1 year after generation';
  enhancementRequests: '6 months after completion';
  userAnalytics: '2 years after user deletion';
  errorLogs: '90 days';
}
```

#### Automated Cleanup
```typescript
// Cloud Function for data cleanup
export const cleanupExpiredData = functions.pubsub
  .schedule('0 2 * * *') // Daily at 2 AM
  .onRun(async (context) => {
    const cutoffDate = new Date();
    cutoffDate.setMonth(cutoffDate.getMonth() - 12);
    
    // Clean up old enhancement requests
    const expiredRequests = await admin.firestore()
      .collection('enhancement_requests')
      .where('completedAt', '<', cutoffDate)
      .get();
    
    const batch = admin.firestore().batch();
    expiredRequests.docs.forEach(doc => {
      batch.delete(doc.ref);
    });
    
    await batch.commit();
  });
```

#### Data Export and Portability
```typescript
// User data export functionality
export const exportUserData = functions.https
  .onCall(async (data, context) => {
    if (!context.auth) {
      throw new functions.https.HttpsError('unauthenticated', 'User must be authenticated');
    }
    
    const userId = context.auth.uid;
    const userData = {
      profile: await getUserProfile(userId),
      cvs: await getUserCVs(userId),
      jobDescriptions: await getUserJobDescriptions(userId),
      tailoredResumes: await getUserTailoredResumes(userId)
    };
    
    // Create export file and return download URL
    return await createExportFile(userData, userId);
  });
```

### Performance Optimization

#### Indexing Strategy
```typescript
// Composite indexes for common queries
const indexes = [
  {
    collection: 'original_cvs',
    fields: [
      { field: 'userId', order: 'ASCENDING' },
      { field: 'isActive', order: 'ASCENDING' },
      { field: 'uploadedAt', order: 'DESCENDING' }
    ]
  },
  {
    collection: 'tailored_resumes',
    fields: [
      { field: 'userId', order: 'ASCENDING' },
      { field: 'generatedAt', order: 'DESCENDING' }
    ]
  },
  {
    collection: 'enhancement_requests',
    fields: [
      { field: 'userId', order: 'ASCENDING' },
      { field: 'status', order: 'ASCENDING' },
      { field: 'createdAt', order: 'DESCENDING' }
    ]
  }
];
```

#### Caching Strategy
```typescript
// Redis caching for frequently accessed data
interface CacheStrategy {
  userProfiles: '1 hour';
  activeCVs: '30 minutes';
  recentJobDescriptions: '15 minutes';
  enhancementResults: '24 hours';
}
```

#### Real-time Listeners Optimization
```typescript
// Efficient real-time listeners
const useEnhancementStatus = (requestId: string) => {
  const [status, setStatus] = useState<string>('pending');
  
  useEffect(() => {
    if (!requestId) return;
    
    const unsubscribe = onSnapshot(
      doc(db, 'enhancement_requests', requestId),
      (doc) => {
        if (doc.exists()) {
          setStatus(doc.data().status);
        }
      },
      { includeMetadataChanges: false } // Optimize for actual data changes only
    );
    
    return unsubscribe;
  }, [requestId]);
  
  return status;
};
```

## Rationale

### Why Firestore?
- **Real-time Updates**: Native real-time listeners for live UI updates
- **Scalability**: Automatic scaling with multi-region support
- **Security**: Comprehensive security rules and authentication integration
- **Offline Support**: Built-in offline capabilities for mobile apps
- **Complex Queries**: Support for compound queries and filtering

### Why Firebase Storage?
- **Integration**: Seamless integration with Firestore and Authentication
- **CDN**: Global CDN for fast file access
- **Security**: Integration with Firebase security rules
- **Scalability**: Automatic scaling for file storage needs

### Why This Security Model?
- **Principle of Least Privilege**: Users only access their own data
- **Defense in Depth**: Multiple layers of security (rules, encryption, authentication)
- **Compliance Ready**: Supports GDPR, CCPA, and other regulations
- **Audit Trail**: Comprehensive logging for security monitoring

## Consequences

### Positive
- **Strong Security**: Multiple layers of protection for sensitive data
- **Real-time Performance**: Fast updates and synchronization
- **Scalable Architecture**: Handles growth in users and data
- **Compliance Ready**: Meets data protection requirements
- **Developer Experience**: Easy to work with and debug

### Negative
- **Vendor Lock-in**: Tight coupling with Firebase ecosystem
- **Cost Scaling**: Firestore costs can grow with usage
- **Query Limitations**: Some complex queries not supported
- **Cold Start**: Initial connection can be slow

### Risks
- **Data Breaches**: Centralized data storage creates target
- **Service Outages**: Dependency on Firebase availability
- **Compliance Changes**: Regulations may require architecture changes

## Implementation Checklist

### Security Implementation
- [ ] Implement Firebase Security Rules
- [ ] Set up authentication flows
- [ ] Configure secure file upload
- [ ] Implement data encryption for sensitive fields
- [ ] Set up audit logging

### Performance Optimization
- [ ] Create database indexes
- [ ] Implement caching strategy
- [ ] Optimize real-time listeners
- [ ] Set up monitoring and alerting

### Compliance
- [ ] Implement data retention policies
- [ ] Create data export functionality
- [ ] Set up automated cleanup
- [ ] Document data flows for compliance

### Monitoring
- [ ] Set up Firebase Performance Monitoring
- [ ] Configure error tracking
- [ ] Implement usage analytics
- [ ] Create security monitoring dashboard

## Review Date

This architecture should be reviewed after reaching 1,000 users or 6 months from implementation, whichever comes first.
