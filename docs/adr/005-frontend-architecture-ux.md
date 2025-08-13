# ADR-005: Frontend Architecture and User Experience

**Date:** 2025-08-13  
**Status:** Accepted  
**Deciders:** Development Team  

## Context

We need to design a frontend architecture that provides an intuitive user experience for the AI resume builder while being maintainable, scalable, and fast to develop. The interface should handle file uploads, real-time updates, and provide clear feedback during AI processing.

## Decision

We will build a React 18 TypeScript application using Vite as the build tool, with a component-based architecture utilizing shadcn/ui components and Tailwind CSS for rapid development and consistent design.

### Frontend Architecture Overview

```
src/
├── components/          # Reusable UI components
├── pages/              # Route-based page components
├── hooks/              # Custom React hooks
├── services/           # External service integrations
├── types/              # TypeScript type definitions
├── utils/              # Utility functions
├── contexts/           # React context providers
└── assets/             # Static assets
```

### Core User Interface Components

#### 1. Authentication Flow
```typescript
// Components: LoginPage, RegisterPage, AuthGuard
interface AuthState {
  user: User | null;
  loading: boolean;
  error: string | null;
}

const AuthContext = createContext<AuthState | undefined>(undefined);

// Custom hook for authentication
const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};
```

#### 2. CV Upload Interface
```typescript
// Components: CVUploader, FileDropzone, UploadProgress
interface CVUploadState {
  uploading: boolean;
  progress: number;
  error: string | null;
  parsedContent: CVContent | null;
}

const CVUploader: React.FC = () => {
  const [uploadState, setUploadState] = useState<CVUploadState>({
    uploading: false,
    progress: 0,
    error: null,
    parsedContent: null
  });

  const handleFileUpload = async (file: File) => {
    // Upload logic with progress tracking
  };

  return (
    <div className="space-y-4">
      <FileDropzone onFileSelect={handleFileUpload} />
      {uploadState.uploading && (
        <UploadProgress progress={uploadState.progress} />
      )}
      {uploadState.parsedContent && (
        <CVPreview content={uploadState.parsedContent} />
      )}
    </div>
  );
};
```

#### 3. Job Description Management
```typescript
// Components: JobDescriptionForm, JobList, JobCard
interface JobDescription {
  id: string;
  title: string;
  company: string;
  description: string;
  requirements: string[];
  createdAt: Date;
}

const JobDescriptionForm: React.FC = () => {
  const [formData, setFormData] = useState<Partial<JobDescription>>({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setIsSubmitting(true);
    
    try {
      await jobService.createJobDescription(formData);
      // Handle success
    } catch (error) {
      // Handle error
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <Input 
        label="Job Title"
        value={formData.title}
        onChange={(value) => setFormData({...formData, title: value})}
        required
      />
      <Input 
        label="Company"
        value={formData.company}
        onChange={(value) => setFormData({...formData, company: value})}
        required
      />
      <Textarea 
        label="Job Description"
        value={formData.description}
        onChange={(value) => setFormData({...formData, description: value})}
        rows={6}
        required
      />
      <Button type="submit" loading={isSubmitting}>
        Save Job Description
      </Button>
    </form>
  );
};
```

#### 4. Resume Enhancement Interface
```typescript
// Components: EnhancementPanel, ProgressTracker, ResultsComparison
interface EnhancementState {
  status: 'idle' | 'processing' | 'completed' | 'error';
  progress: number;
  result: TailoredResume | null;
  error: string | null;
}

const EnhancementPanel: React.FC<{
  cvId: string;
  jobId: string;
}> = ({ cvId, jobId }) => {
  const [enhancementState, setEnhancementState] = useState<EnhancementState>({
    status: 'idle',
    progress: 0,
    result: null,
    error: null
  });

  const startEnhancement = async () => {
    setEnhancementState(prev => ({ ...prev, status: 'processing' }));
    
    try {
      const requestId = await enhancementService.createRequest(cvId, jobId);
      
      // Listen for real-time updates
      const unsubscribe = enhancementService.onStatusUpdate(requestId, (update) => {
        setEnhancementState(prev => ({
          ...prev,
          progress: update.progress,
          status: update.status,
          result: update.result,
          error: update.error
        }));
      });

      return () => unsubscribe();
    } catch (error) {
      setEnhancementState(prev => ({
        ...prev,
        status: 'error',
        error: error.message
      }));
    }
  };

  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <h2 className="text-2xl font-bold">Enhance Resume</h2>
        <Button 
          onClick={startEnhancement}
          disabled={enhancementState.status === 'processing'}
          loading={enhancementState.status === 'processing'}
        >
          Start Enhancement
        </Button>
      </div>

      {enhancementState.status === 'processing' && (
        <ProgressTracker progress={enhancementState.progress} />
      )}

      {enhancementState.status === 'completed' && enhancementState.result && (
        <ResultsComparison 
          original={cvContent}
          enhanced={enhancementState.result}
        />
      )}

      {enhancementState.status === 'error' && (
        <ErrorDisplay error={enhancementState.error} />
      )}
    </div>
  );
};
```

### State Management Strategy

#### Context-Based State Management
```typescript
// Global application state using React Context
interface AppState {
  user: User | null;
  activeCV: CVDocument | null;
  jobDescriptions: JobDescription[];
  enhancementHistory: TailoredResume[];
}

const AppStateContext = createContext<{
  state: AppState;
  dispatch: React.Dispatch<AppAction>;
} | undefined>(undefined);

type AppAction = 
  | { type: 'SET_USER'; payload: User | null }
  | { type: 'SET_ACTIVE_CV'; payload: CVDocument | null }
  | { type: 'ADD_JOB_DESCRIPTION'; payload: JobDescription }
  | { type: 'ADD_ENHANCEMENT_RESULT'; payload: TailoredResume };

const appReducer = (state: AppState, action: AppAction): AppState => {
  switch (action.type) {
    case 'SET_USER':
      return { ...state, user: action.payload };
    case 'SET_ACTIVE_CV':
      return { ...state, activeCV: action.payload };
    case 'ADD_JOB_DESCRIPTION':
      return { 
        ...state, 
        jobDescriptions: [...state.jobDescriptions, action.payload] 
      };
    case 'ADD_ENHANCEMENT_RESULT':
      return { 
        ...state, 
        enhancementHistory: [...state.enhancementHistory, action.payload] 
      };
    default:
      return state;
  }
};
```

#### Real-time Data Hooks
```typescript
// Custom hooks for real-time Firebase data
const useFirestoreCollection = <T>(
  collectionPath: string,
  queryConstraints?: QueryConstraint[]
) => {
  const [data, setData] = useState<T[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const q = query(
      collection(db, collectionPath),
      ...(queryConstraints || [])
    );

    const unsubscribe = onSnapshot(
      q,
      (snapshot) => {
        const items = snapshot.docs.map(doc => ({
          id: doc.id,
          ...doc.data()
        })) as T[];
        setData(items);
        setLoading(false);
      },
      (err) => {
        setError(err.message);
        setLoading(false);
      }
    );

    return unsubscribe;
  }, [collectionPath, queryConstraints]);

  return { data, loading, error };
};

// Usage example
const useUserJobDescriptions = (userId: string) => {
  return useFirestoreCollection<JobDescription>(
    'job_descriptions',
    [where('userId', '==', userId), orderBy('createdAt', 'desc')]
  );
};
```

### User Experience Design

#### 1. Onboarding Flow
```typescript
const OnboardingFlow: React.FC = () => {
  const [currentStep, setCurrentStep] = useState(0);
  
  const steps = [
    { title: 'Welcome', component: WelcomeStep },
    { title: 'Upload CV', component: CVUploadStep },
    { title: 'Add Job', component: JobDescriptionStep },
    { title: 'Generate Resume', component: EnhancementStep }
  ];

  return (
    <div className="max-w-2xl mx-auto p-6">
      <ProgressIndicator 
        currentStep={currentStep} 
        totalSteps={steps.length} 
      />
      <div className="mt-8">
        {React.createElement(steps[currentStep].component, {
          onNext: () => setCurrentStep(prev => prev + 1),
          onPrev: () => setCurrentStep(prev => prev - 1)
        })}
      </div>
    </div>
  );
};
```

#### 2. Loading States and Feedback
```typescript
// Loading states for different operations
interface LoadingStates {
  uploadingCV: boolean;
  processingAI: boolean;
  generatingPDF: boolean;
  savingJobDescription: boolean;
}

const LoadingIndicator: React.FC<{
  type: keyof LoadingStates;
  message?: string;
}> = ({ type, message }) => {
  const messages = {
    uploadingCV: 'Uploading and parsing your CV...',
    processingAI: 'AI is enhancing your resume...',
    generatingPDF: 'Generating your tailored resume...',
    savingJobDescription: 'Saving job description...'
  };

  return (
    <div className="flex items-center space-x-3 p-4 bg-blue-50 rounded-lg">
      <Spinner className="w-5 h-5 text-blue-600" />
      <span className="text-blue-800">
        {message || messages[type]}
      </span>
    </div>
  );
};
```

#### 3. Error Handling and Recovery
```typescript
// Error boundary for React error handling
class ErrorBoundary extends React.Component<
  { children: React.ReactNode },
  { hasError: boolean; error: Error | null }
> {
  constructor(props: { children: React.ReactNode }) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('Error caught by boundary:', error, errorInfo);
    // Log to error tracking service
  }

  render() {
    if (this.state.hasError) {
      return (
        <ErrorFallback 
          error={this.state.error}
          resetError={() => this.setState({ hasError: false, error: null })}
        />
      );
    }

    return this.props.children;
  }
}

// User-friendly error display
const ErrorFallback: React.FC<{
  error: Error | null;
  resetError: () => void;
}> = ({ error, resetError }) => {
  return (
    <div className="min-h-screen flex items-center justify-center p-4">
      <div className="max-w-md w-full bg-white rounded-lg shadow-lg p-6">
        <div className="flex items-center mb-4">
          <AlertCircle className="w-8 h-8 text-red-500 mr-3" />
          <h2 className="text-xl font-semibold text-gray-900">
            Something went wrong
          </h2>
        </div>
        <p className="text-gray-600 mb-6">
          We encountered an unexpected error. Please try again or contact support if the problem persists.
        </p>
        <div className="flex space-x-3">
          <Button onClick={resetError} variant="outline">
            Try Again
          </Button>
          <Button onClick={() => window.location.reload()}>
            Refresh Page
          </Button>
        </div>
      </div>
    </div>
  );
};
```

### Performance Optimization

#### 1. Code Splitting and Lazy Loading
```typescript
// Route-based code splitting
const Dashboard = lazy(() => import('./pages/Dashboard'));
const CVUpload = lazy(() => import('./pages/CVUpload'));
const JobDescriptions = lazy(() => import('./pages/JobDescriptions'));
const Enhancement = lazy(() => import('./pages/Enhancement'));

const App: React.FC = () => {
  return (
    <Router>
      <Suspense fallback={<PageLoader />}>
        <Routes>
          <Route path="/" element={<Dashboard />} />
          <Route path="/upload" element={<CVUpload />} />
          <Route path="/jobs" element={<JobDescriptions />} />
          <Route path="/enhance" element={<Enhancement />} />
        </Routes>
      </Suspense>
    </Router>
  );
};
```

#### 2. Memoization and Optimization
```typescript
// Memoized components for performance
const CVPreview = React.memo<{
  content: CVContent;
  onEdit?: (section: string) => void;
}>(({ content, onEdit }) => {
  const sections = useMemo(() => {
    return processCVSections(content);
  }, [content]);

  return (
    <div className="space-y-6">
      {sections.map((section, index) => (
        <CVSection 
          key={section.id} 
          section={section} 
          onEdit={onEdit}
        />
      ))}
    </div>
  );
});

// Debounced input for search and filtering
const useDebounced = <T>(value: T, delay: number): T => {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => {
      clearTimeout(handler);
    };
  }, [value, delay]);

  return debouncedValue;
};
```

### PDF Generation and Alternatives

We will primarily use **React PDF** for client-side PDF generation, but remain open to alternatives based on performance and feature requirements:

#### React PDF
- **Pros**: Client-side generation, React-like syntax, good TypeScript support
- **Cons**: Limited styling options, larger bundle size
- **Use Case**: Simple resume layouts with consistent formatting

#### Alternative Options
- **Puppeteer**: Server-side PDF generation with full HTML/CSS support
- **jsPDF**: Lightweight, programmatic PDF creation
- **PDFKit**: Low-level PDF generation with fine control
- **HTML-to-PDF Services**: External services like Browserless or PDF Shift

#### Decision Criteria
- **Performance**: Client vs server-side generation trade-offs
- **Styling Flexibility**: Need for complex layouts and designs
- **Bundle Size**: Impact on application load time
- **Maintenance**: Complexity of implementation and updates

```typescript
// PDF Generation Service Interface
interface PDFGenerationService {
  generateResume(content: ResumeContent, template: Template): Promise<Blob>;
  previewResume(content: ResumeContent, template: Template): Promise<string>; // Base64 or URL
  getSupportedTemplates(): Template[];
}

// React PDF Implementation
class ReactPDFService implements PDFGenerationService {
  async generateResume(content: ResumeContent, template: Template): Promise<Blob> {
    const doc = (
      <Document>
        <Page size="A4" style={template.styles}>
          <ResumeTemplate content={content} template={template} />
        </Page>
      </Document>
    );
    
    return await pdf(doc).toBlob();
  }
}

// Alternative: Server-side PDF generation
class ServerPDFService implements PDFGenerationService {
  async generateResume(content: ResumeContent, template: Template): Promise<Blob> {
    const response = await fetch('/api/generate-pdf', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ content, template })
    });
    
    return await response.blob();
  }
}
```

## Coding Practices and Standards

### SOLID Principles Implementation

#### 1. Single Responsibility Principle (SRP)
Each component and function should have a single, well-defined responsibility.

```typescript
// ❌ Bad: Component doing too many things
const CVUploadPage: React.FC = () => {
  // File upload logic
  // PDF parsing logic
  // Form validation
  // API calls
  // UI rendering
  // Error handling
  return <div>...</div>;
};

// ✅ Good: Separated responsibilities
const CVUploadPage: React.FC = () => {
  return (
    <div className="cv-upload-page">
      <PageHeader title="Upload CV" />
      <CVUploadForm />
      <UploadProgress />
      <CVPreview />
    </div>
  );
};

const CVUploadForm: React.FC = () => {
  const { upload, progress, error } = useFileUpload();
  // Only handles file upload UI and state
};

const CVPreview: React.FC = () => {
  const { parsedContent } = useParsedCV();
  // Only handles CV preview display
};
```

#### 2. Open/Closed Principle (OCP)
Components should be open for extension but closed for modification.

```typescript
// ✅ Extensible component design
interface BaseInputProps {
  label: string;
  value: string;
  onChange: (value: string) => void;
  error?: string;
  required?: boolean;
}

// Base input component
const BaseInput: React.FC<BaseInputProps> = ({ label, error, ...props }) => {
  return (
    <div className="input-group">
      <label className="input-label">{label}</label>
      <input {...props} className="base-input" />
      {error && <span className="input-error">{error}</span>}
    </div>
  );
};

// Extended components without modifying base
const EmailInput: React.FC<Omit<BaseInputProps, 'type'>> = (props) => (
  <BaseInput {...props} type="email" />
);

const PasswordInput: React.FC<Omit<BaseInputProps, 'type'>> = (props) => (
  <BaseInput {...props} type="password" />
);

// Plugin-style extensions
interface InputPlugin {
  validate?: (value: string) => string | null;
  transform?: (value: string) => string;
  icon?: React.ComponentType;
}

const ExtendedInput: React.FC<BaseInputProps & { plugins?: InputPlugin[] }> = 
  ({ plugins = [], ...props }) => {
    // Apply plugins without modifying base component
  };
```

#### 3. Liskov Substitution Principle (LSP)
Derived components should be substitutable for their base components.

```typescript
// ✅ Proper substitution hierarchy
interface ButtonProps {
  children: React.ReactNode;
  onClick: () => void;
  disabled?: boolean;
}

const Button: React.FC<ButtonProps> = ({ children, onClick, disabled }) => (
  <button onClick={onClick} disabled={disabled} className="btn">
    {children}
  </button>
);

// Specialized buttons that can substitute the base Button
const PrimaryButton: React.FC<ButtonProps> = (props) => (
  <Button {...props} className="btn btn-primary" />
);

const LoadingButton: React.FC<ButtonProps & { loading?: boolean }> = 
  ({ loading, children, ...props }) => (
    <Button {...props} disabled={props.disabled || loading}>
      {loading ? <Spinner /> : children}
    </Button>
  );

// Can be used interchangeably
const actions = [
  <Button onClick={handleSave}>Save</Button>,
  <PrimaryButton onClick={handleSubmit}>Submit</PrimaryButton>,
  <LoadingButton onClick={handleProcess} loading={isProcessing}>Process</LoadingButton>
];
```

#### 4. Interface Segregation Principle (ISP)
Create specific interfaces rather than large, monolithic ones.

```typescript
// ❌ Bad: Fat interface
interface CVOperations {
  upload: (file: File) => Promise<void>;
  parse: (content: string) => CVContent;
  enhance: (content: CVContent, job: JobDescription) => Promise<CVContent>;
  generatePDF: (content: CVContent) => Promise<Blob>;
  save: (content: CVContent) => Promise<void>;
  delete: (id: string) => Promise<void>;
}

// ✅ Good: Segregated interfaces
interface CVUploadOperations {
  upload: (file: File) => Promise<void>;
  parse: (content: string) => CVContent;
}

interface CVEnhancementOperations {
  enhance: (content: CVContent, job: JobDescription) => Promise<CVContent>;
}

interface CVStorageOperations {
  save: (content: CVContent) => Promise<void>;
  delete: (id: string) => Promise<void>;
}

interface CVExportOperations {
  generatePDF: (content: CVContent) => Promise<Blob>;
}

// Components only depend on what they need
const CVUploader: React.FC = () => {
  const { upload, parse } = useService<CVUploadOperations>('cvUpload');
  // Only has access to upload operations
};

const CVEnhancer: React.FC = () => {
  const { enhance } = useService<CVEnhancementOperations>('cvEnhancement');
  // Only has access to enhancement operations
};
```

#### 5. Dependency Inversion Principle (DIP)
Depend on abstractions, not concretions.

```typescript
// ✅ Abstract service interfaces
interface StorageService {
  upload(file: File, path: string): Promise<string>;
  download(url: string): Promise<Blob>;
  delete(url: string): Promise<void>;
}

interface AIService {
  enhanceContent(content: string, context: string): Promise<string>;
  analyzeContent(content: string): Promise<AnalysisResult>;
}

// Concrete implementations
class FirebaseStorageService implements StorageService {
  async upload(file: File, path: string): Promise<string> {
    // Firebase-specific implementation
  }
}

class OpenAIService implements AIService {
  async enhanceContent(content: string, context: string): Promise<string> {
    // OpenAI-specific implementation
  }
}

// Components depend on abstractions
const CVUploader: React.FC = () => {
  const storageService = useService<StorageService>('storage');
  const aiService = useService<AIService>('ai');
  
  // Can work with any implementation of these interfaces
};

// Dependency injection setup
const ServiceProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const services = {
    storage: new FirebaseStorageService(),
    ai: new OpenAIService(),
  };
  
  return (
    <ServiceContext.Provider value={services}>
      {children}
    </ServiceContext.Provider>
  );
};
```

### Component Architecture Guidelines

#### Small, Focused Components
Err on the side of creating many small files rather than large, monolithic components.

```typescript
// ✅ File structure for CV upload feature
src/
├── components/
│   ├── cv-upload/
│   │   ├── index.ts                    // Export barrel
│   │   ├── CVUploadPage.tsx           // Main page component
│   │   ├── CVUploadForm.tsx           // Form wrapper
│   │   ├── FileDropzone.tsx           // Drag & drop area
│   │   ├── FilePreview.tsx            // File preview
│   │   ├── UploadProgress.tsx         // Progress indicator
│   │   ├── ParsedContentPreview.tsx   // Parsed content display
│   │   ├── ValidationErrors.tsx       // Error display
│   │   └── hooks/
│   │       ├── useFileUpload.ts       // Upload logic
│   │       ├── useFileValidation.ts   // Validation logic
│   │       └── useParsedContent.ts    // Content parsing
│   ├── ui/                            // Reusable UI components
│   │   ├── Button/
│   │   │   ├── Button.tsx
│   │   │   ├── Button.types.ts
│   │   │   └── index.ts
│   │   ├── Input/
│   │   │   ├── Input.tsx
│   │   │   ├── Input.types.ts
│   │   │   └── index.ts
│   │   └── Modal/
│   │       ├── Modal.tsx
│   │       ├── Modal.types.ts
│   │       └── index.ts
│   └── common/                        // Shared business components
│       ├── ErrorBoundary/
│       ├── LoadingSpinner/
│       └── PageHeader/

// ✅ Small, focused components
const FileDropzone: React.FC<FileDropzoneProps> = ({ onFileSelect, accept, maxSize }) => {
  // Only handles file drop functionality - ~50 lines
};

const UploadProgress: React.FC<UploadProgressProps> = ({ progress, status }) => {
  // Only displays upload progress - ~30 lines
};

const ValidationErrors: React.FC<ValidationErrorsProps> = ({ errors }) => {
  // Only displays validation errors - ~20 lines
};
```

#### Component Composition Patterns

```typescript
// ✅ Composition over inheritance
const CVUploadPage: React.FC = () => {
  return (
    <PageLayout>
      <PageHeader 
        title="Upload CV" 
        subtitle="Upload your resume to get started"
      />
      
      <Card>
        <CardHeader>
          <CardTitle>Select File</CardTitle>
        </CardHeader>
        <CardContent>
          <CVUploadForm>
            <FileDropzone />
            <FileValidation />
            <UploadProgress />
          </CVUploadForm>
        </CardContent>
      </Card>
      
      <Conditional when={hasContent}>
        <Card>
          <CardHeader>
            <CardTitle>Preview</CardTitle>
          </CardHeader>
          <CardContent>
            <ParsedContentPreview />
          </CardContent>
        </Card>
      </Conditional>
    </PageLayout>
  );
};

// ✅ Compound components pattern
const CVUploadForm = {
  Root: CVUploadFormRoot,
  Dropzone: FileDropzone,
  Progress: UploadProgress,
  Preview: ParsedContentPreview,
  Errors: ValidationErrors,
};

// Usage
<CVUploadForm.Root>
  <CVUploadForm.Dropzone />
  <CVUploadForm.Progress />
  <CVUploadForm.Preview />
  <CVUploadForm.Errors />
</CVUploadForm.Root>
```

#### File Naming Conventions

```typescript
// ✅ Clear, descriptive file names
src/
├── components/
│   ├── CVUploadPage.tsx              // PascalCase for components
│   ├── CVUploadForm.tsx
│   ├── FileDropzone.tsx
│   └── hooks/
│       ├── useFileUpload.ts          // camelCase for hooks
│       ├── useValidation.ts
│       └── useParsedContent.ts
├── services/
│   ├── firebaseService.ts            // camelCase for services
│   ├── pdfParsingService.ts
│   └── aiEnhancementService.ts
├── types/
│   ├── CVTypes.ts                    // PascalCase for type files
│   ├── JobTypes.ts
│   └── index.ts                      // Export barrel
└── utils/
    ├── fileValidation.ts             // camelCase for utilities
    ├── pdfParser.ts
    └── formatters.ts
```

#### Component Testing Strategy

```typescript
// ✅ Component-specific test files
src/
├── components/
│   ├── CVUploadPage/
│   │   ├── CVUploadPage.tsx
│   │   ├── CVUploadPage.test.tsx     // Co-located tests
│   │   ├── CVUploadPage.stories.tsx  // Storybook stories
│   │   └── index.ts
│   └── FileDropzone/
│       ├── FileDropzone.tsx
│       ├── FileDropzone.test.tsx
│       ├── FileDropzone.stories.tsx
│       └── index.ts

// ✅ Test structure following component structure
describe('CVUploadPage', () => {
  describe('FileDropzone', () => {
    it('should accept valid file types', () => {});
    it('should reject invalid file types', () => {});
    it('should handle drag and drop', () => {});
  });
  
  describe('UploadProgress', () => {
    it('should display progress percentage', () => {});
    it('should show completion state', () => {});
  });
  
  describe('ValidationErrors', () => {
    it('should display file size errors', () => {});
    it('should display file type errors', () => {});
  });
});
```

### Code Organization Benefits

#### Maintainability
- **Easy to Locate**: Features organized in dedicated folders
- **Small Surface Area**: Each file has a focused responsibility
- **Clear Dependencies**: Import/export relationships are explicit

#### Testability
- **Isolated Testing**: Small components are easier to test
- **Mocking**: Dependencies can be easily mocked
- **Coverage**: Easier to achieve comprehensive test coverage

#### Reusability
- **Component Library**: Small components become reusable building blocks
- **Composition**: Components can be combined in different ways
- **Extensibility**: Easy to extend without modifying existing code

#### Team Collaboration
- **Reduced Conflicts**: Smaller files reduce merge conflicts
- **Parallel Development**: Team members can work on different components
- **Code Reviews**: Smaller changes are easier to review

### Implementation Guidelines

1. **Start Small**: Begin with the smallest possible component
2. **Extract Early**: When a component reaches ~100 lines, consider extraction
3. **Single Concern**: Each file should address one specific concern
4. **Clear Interfaces**: Define clear props and return types
5. **Consistent Patterns**: Follow established patterns across the codebase
6. **Document Decisions**: Use comments to explain complex logic or decisions

## Rationale

### Why React 18 + TypeScript?
- **Type Safety**: Prevents runtime errors and improves developer experience
- **Component Reusability**: Modular architecture for maintainable code
- **Ecosystem**: Large ecosystem of libraries and tools
- **Performance**: Built-in optimizations and concurrent features
- **Developer Experience**: Excellent tooling and debugging capabilities

### Why Vite?
- **Fast Development**: Hot module replacement and fast builds
- **Modern Tooling**: Native ES modules and optimized bundling
- **TypeScript Support**: First-class TypeScript integration
- **Plugin Ecosystem**: Rich plugin ecosystem for additional features

### Why shadcn/ui + Tailwind?
- **Rapid Development**: Pre-built components with consistent design
- **Customization**: Fully customizable and extendable
- **Performance**: Utility-first CSS with optimized bundle size
- **Accessibility**: Built-in accessibility features
- **Design System**: Consistent design language across the application

### Why Context-Based State Management?
- **Simplicity**: Avoid complexity of external state management libraries
- **React Native**: Built into React, no additional dependencies
- **Type Safety**: Full TypeScript support
- **Server State**: Firebase handles server state synchronization

## Consequences

### Positive
- **Fast Development**: Rapid prototyping with pre-built components
- **Type Safety**: Reduced bugs and better developer experience
- **Performance**: Optimized builds and runtime performance
- **Maintainability**: Clean architecture and reusable components
- **User Experience**: Responsive design and smooth interactions

### Negative
- **Learning Curve**: Team needs to learn TypeScript and new tools
- **Bundle Size**: React application can be larger than vanilla JS
- **Complexity**: Additional abstraction layers
- **Build Dependencies**: Multiple dependencies to maintain

### Technical Debt
- **State Management**: May need external library for complex state
- **Testing**: Need to set up comprehensive testing strategy
- **SEO**: Single-page application may need SSR for better SEO

## Implementation Plan

### Phase 1: Core Infrastructure
- [ ] Set up Vite + React + TypeScript project
- [ ] Configure Tailwind CSS and shadcn/ui
- [ ] Implement authentication flow
- [ ] Set up Firebase integration

### Phase 2: Core Features
- [ ] Build CV upload interface
- [ ] Implement job description management
- [ ] Create enhancement interface
- [ ] Add real-time updates

### Phase 3: User Experience
- [ ] Implement onboarding flow
- [ ] Add comprehensive error handling
- [ ] Optimize performance and loading states
- [ ] Add accessibility features

### Phase 4: Polish and Testing
- [ ] Comprehensive testing suite
- [ ] Performance optimization
- [ ] Cross-browser compatibility
- [ ] Mobile responsiveness

## Review Date

This frontend architecture should be reviewed after the MVP is completed or by 2025-11-30, whichever comes first.
