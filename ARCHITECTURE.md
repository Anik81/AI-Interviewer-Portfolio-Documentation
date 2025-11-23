# System Architecture Documentation

## Overview

This document provides an in-depth look at the AI-Powered Interview Platform's architecture, design decisions, and technical implementation details.

---

## 1. High-Level Architecture

### 1.1 Three-Tier Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Presentation Layer                        │
│  Next.js 15 App Router │ React 18 │ TypeScript │ Tailwind   │
└──────────────────┬──────────────────────────────────────────┘
                   │ REST API (JSON)
┌──────────────────▼──────────────────────────────────────────┐
│                    Application Layer                         │
│  FastAPI Microservices │ Business Logic │ API Gateway        │
└──────────────────┬──────────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────────┐
│                      Data Layer                              │
│  PostgreSQL │ Redis │ File Storage │ Vector DB (ChromaDB)   │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Microservices Architecture

The platform is decomposed into distinct microservices, each responsible for a specific domain:

```
┌──────────────────────────────────────────────────────────────────┐
│                         API Gateway                               │
│                 (Request Routing & Load Balancing)                │
└────┬─────────┬──────────┬──────────┬──────────┬─────────┬────────┘
     │         │          │          │          │         │
┌────▼────┐ ┌─▼──────┐ ┌─▼──────┐ ┌─▼──────┐ ┌─▼─────┐ ┌─▼──────┐
│  Auth   │ │Interview│ │   CV   │ │  Job   │ │ Media │ │ Notify │
│ Service │ │ Service │ │Screening│ │Matching│ │Service│ │Service │
└─────────┘ └─────────┘ └─────────┘ └────────┘ └───────┘ └────────┘
     │         │          │          │          │         │
     └─────────┴──────────┴──────────┴──────────┴─────────┘
                              │
                    ┌─────────▼─────────┐
                    │ Shared Data Layer │
                    └───────────────────┘
```

---

## 2. Frontend Architecture (Next.js)

### 2.1 Application Structure

```
ai-interviewer-frontend/
├── app/                          # App Router pages
│   ├── (auth)/                   # Auth route group (no layout)
│   │   ├── login/
│   │   └── register/
│   ├── dashboard/                # Protected routes
│   ├── interview/
│   └── layout.tsx                # Root layout
│
├── components/                   # React components
│   ├── ui/                       # Reusable UI components
│   ├── forms/                    # Form components
│   ├── layouts/                  # Layout components
│   └── shared/                   # Shared business components
│
├── lib/                          # Business logic & utilities
│   ├── api/                      # API client & endpoints
│   ├── hooks/                    # Custom React hooks
│   ├── utils/                    # Helper functions
│   └── types/                    # TypeScript definitions
│
├── stores/                       # State management (Zustand)
│   ├── auth.ts
│   ├── interview.ts
│   └── ui.ts
│
├── contexts/                     # React Context providers
│   ├── AuthContext.tsx
│   └── InterviewContext.tsx
│
└── public/                       # Static assets
    ├── images/
    └── workers/                  # Web Workers for heavy tasks
```

### 2.2 Routing Strategy

**App Router (Next.js 15):**
- **File-system based routing**: Each folder in `app/` becomes a route
- **Route Groups**: Organize routes without affecting URL structure
- **Layouts**: Shared UI that persists across routes
- **Loading & Error States**: Automatic handling with loading.tsx/error.tsx

**Route Protection:**
```typescript
// middleware.ts
export function middleware(request: NextRequest) {
  const token = request.cookies.get('auth-token')
  const isAuthRoute = request.nextUrl.pathname.startsWith('/login')

  if (!token && !isAuthRoute) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  if (token && isAuthRoute) {
    return NextResponse.redirect(new URL('/dashboard', request.url))
  }
}
```

### 2.3 State Management Strategy

**Multi-layered State Management:**

1. **URL State** (Search Params)
   - Filters, pagination, sort order
   - Shareable state via URLs

2. **Server State** (React Query)
   - API data caching
   - Background refetching
   - Optimistic updates

3. **Global Client State** (Zustand)
   - Authentication state
   - User preferences
   - UI state (modals, theme)

4. **Local Component State** (useState)
   - Form inputs
   - UI toggles
   - Temporary data

**Example Zustand Store:**
```typescript
interface AuthStore {
  user: User | null
  token: string | null
  login: (credentials: LoginData) => Promise<void>
  logout: () => void
  refreshToken: () => Promise<void>
}

const useAuthStore = create<AuthStore>((set) => ({
  user: null,
  token: null,
  login: async (credentials) => {
    const response = await apiClient.post('/auth/login', credentials)
    set({ user: response.data.user, token: response.data.token })
  },
  logout: () => set({ user: null, token: null }),
  refreshToken: async () => { /* ... */ }
}))
```

### 2.4 Data Fetching Patterns

**React Query for Server State:**
```typescript
// Custom hook for fetching interview data
function useInterview(id: string) {
  return useQuery({
    queryKey: ['interview', id],
    queryFn: () => apiClient.get(`/interview/${id}`),
    staleTime: 5 * 60 * 1000, // 5 minutes
    cacheTime: 10 * 60 * 1000, // 10 minutes
    refetchOnWindowFocus: false
  })
}

// Optimistic update for submitting answer
function useSubmitAnswer() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (answer: AnswerData) =>
      apiClient.post('/interview/answer', answer),
    onMutate: async (answer) => {
      // Optimistically update UI
      await queryClient.cancelQueries(['interview', answer.interviewId])
      const previous = queryClient.getQueryData(['interview', answer.interviewId])

      queryClient.setQueryData(['interview', answer.interviewId], (old: any) => ({
        ...old,
        answers: [...old.answers, answer]
      }))

      return { previous }
    },
    onError: (err, answer, context) => {
      // Rollback on error
      queryClient.setQueryData(
        ['interview', answer.interviewId],
        context?.previous
      )
    }
  })
}
```

---

## 3. Backend Architecture (FastAPI)

### 3.1 Service Structure

**Domain-Driven Design:**

```
backend/
├── services/
│   ├── auth/
│   │   ├── domain/              # Business entities
│   │   │   ├── user.py
│   │   │   └── token.py
│   │   ├── repository/          # Data access
│   │   │   └── user_repository.py
│   │   ├── use_cases/           # Business logic
│   │   │   ├── register_user.py
│   │   │   └── authenticate_user.py
│   │   └── api/                 # API routes
│   │       └── routes.py
│   │
│   └── interview/
│       ├── domain/
│       │   ├── interview.py
│       │   ├── question.py
│       │   └── response.py
│       ├── repository/
│       │   └── interview_repository.py
│       ├── use_cases/
│       │   ├── create_interview.py
│       │   ├── generate_questions.py
│       │   └── evaluate_response.py
│       └── api/
│           └── routes.py
│
├── core/                        # Shared infrastructure
│   ├── database.py              # DB connection
│   ├── config.py                # Configuration
│   ├── dependencies.py          # DI container
│   └── security.py              # Auth utilities
│
├── integrations/                # External services
│   ├── openai_client.py
│   ├── gemini_client.py
│   └── redis_client.py
│
└── main.py                      # Application entry point
```

### 3.2 Dependency Injection

**FastAPI's DI System:**
```python
# Dependency providers
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        yield session

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db)
) -> User:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials"
    )

    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    user_id = payload.get("sub")

    if user_id is None:
        raise credentials_exception

    user = await user_repository.get_by_id(db, user_id)
    if user is None:
        raise credentials_exception

    return user

# Usage in route
@router.get("/profile")
async def get_profile(
    current_user: User = Depends(get_current_user)
):
    return current_user
```

### 3.3 Repository Pattern

**Abstract Repository:**
```python
class BaseRepository(Generic[T]):
    def __init__(self, model: Type[T]):
        self.model = model

    async def get_by_id(
        self, db: AsyncSession, id: int
    ) -> Optional[T]:
        result = await db.execute(
            select(self.model).where(self.model.id == id)
        )
        return result.scalar_one_or_none()

    async def create(
        self, db: AsyncSession, obj_in: BaseModel
    ) -> T:
        obj_data = obj_in.dict()
        db_obj = self.model(**obj_data)
        db.add(db_obj)
        await db.commit()
        await db.refresh(db_obj)
        return db_obj

    async def update(
        self, db: AsyncSession, db_obj: T, obj_in: BaseModel
    ) -> T:
        obj_data = obj_in.dict(exclude_unset=True)
        for field, value in obj_data.items():
            setattr(db_obj, field, value)
        await db.commit()
        await db.refresh(db_obj)
        return db_obj

# Concrete implementation
class InterviewRepository(BaseRepository[Interview]):
    async def get_by_candidate(
        self, db: AsyncSession, candidate_id: int
    ) -> List[Interview]:
        result = await db.execute(
            select(Interview)
            .where(Interview.candidate_id == candidate_id)
            .order_by(Interview.created_at.desc())
        )
        return result.scalars().all()
```

### 3.4 Use Case Pattern

**Business Logic Encapsulation:**
```python
class CreateInterviewUseCase:
    def __init__(
        self,
        interview_repo: InterviewRepository,
        question_generator: QuestionGenerator
    ):
        self.interview_repo = interview_repo
        self.question_generator = question_generator

    async def execute(
        self,
        db: AsyncSession,
        candidate_id: int,
        job_id: int,
        cv_text: str
    ) -> Interview:
        # Generate questions based on CV and job requirements
        questions = await self.question_generator.generate(
            cv_text=cv_text,
            job_id=job_id
        )

        # Create interview entity
        interview = Interview(
            candidate_id=candidate_id,
            job_id=job_id,
            status=InterviewStatus.INITIATED,
            questions=questions
        )

        # Persist to database
        db_interview = await self.interview_repo.create(db, interview)

        # Emit domain event
        await event_bus.publish(
            InterviewCreatedEvent(interview_id=db_interview.id)
        )

        return db_interview
```

---

## 4. Data Layer Architecture

### 4.1 PostgreSQL Database Schema

**Core Tables:**

```sql
-- Users table (multi-tenant)
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    hashed_password VARCHAR(255) NOT NULL,
    full_name VARCHAR(255),
    role VARCHAR(50) NOT NULL, -- 'candidate', 'hr', 'admin'
    organization_id INTEGER REFERENCES organizations(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE
);

-- Organizations table (multi-tenancy)
CREATE TABLE organizations (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    domain VARCHAR(255),
    settings JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Jobs table
CREATE TABLE jobs (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    requirements JSONB,
    organization_id INTEGER REFERENCES organizations(id),
    created_by INTEGER REFERENCES users(id),
    status VARCHAR(50) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Interviews table
CREATE TABLE interviews (
    id SERIAL PRIMARY KEY,
    candidate_id INTEGER REFERENCES users(id),
    job_id INTEGER REFERENCES jobs(id),
    status VARCHAR(50) NOT NULL,
    cv_text TEXT,
    cv_file_path VARCHAR(500),
    scheduled_at TIMESTAMP,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Questions table
CREATE TABLE questions (
    id SERIAL PRIMARY KEY,
    interview_id INTEGER REFERENCES interviews(id),
    question_text TEXT NOT NULL,
    question_type VARCHAR(50), -- 'technical', 'behavioral', 'situational'
    expected_keywords JSONB,
    tts_audio_path VARCHAR(500),
    order_index INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Responses table
CREATE TABLE responses (
    id SERIAL PRIMARY KEY,
    question_id INTEGER REFERENCES questions(id),
    response_text TEXT,
    video_path VARCHAR(500),
    audio_path VARCHAR(500),
    transcription TEXT,
    duration_seconds INTEGER,
    ai_score FLOAT,
    ai_feedback JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Guest access tokens
CREATE TABLE guest_tokens (
    id SERIAL PRIMARY KEY,
    token VARCHAR(500) UNIQUE NOT NULL,
    interview_id INTEGER REFERENCES interviews(id),
    expires_at TIMESTAMP NOT NULL,
    used_at TIMESTAMP,
    created_by INTEGER REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Indexes for Performance:**
```sql
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_interviews_candidate ON interviews(candidate_id);
CREATE INDEX idx_interviews_status ON interviews(status);
CREATE INDEX idx_questions_interview ON questions(interview_id);
CREATE INDEX idx_responses_question ON responses(question_id);
CREATE INDEX idx_guest_tokens_token ON guest_tokens(token);
CREATE INDEX idx_guest_tokens_expires ON guest_tokens(expires_at);
```

### 4.2 Redis Caching Strategy

**Cache Layers:**

1. **Session Cache**
   ```
   Key: session:{user_id}
   TTL: 15 minutes
   Value: { token, user_data, permissions }
   ```

2. **API Response Cache**
   ```
   Key: api:{endpoint}:{params_hash}
   TTL: 5 minutes
   Value: Serialized response JSON
   ```

3. **Rate Limiting**
   ```
   Key: ratelimit:{user_id}:{endpoint}
   TTL: 1 minute
   Value: Request count
   ```

4. **Job Match Cache**
   ```
   Key: jobmatch:{candidate_id}:{job_id}
   TTL: 1 hour
   Value: Match score and details
   ```

**Cache Invalidation Strategy:**
```python
class CacheManager:
    def __init__(self, redis_client: Redis):
        self.redis = redis_client

    async def invalidate_pattern(self, pattern: str):
        """Invalidate all keys matching pattern"""
        keys = await self.redis.keys(pattern)
        if keys:
            await self.redis.delete(*keys)

    async def invalidate_user_cache(self, user_id: int):
        """Invalidate all user-related cache"""
        patterns = [
            f"session:{user_id}",
            f"api:*:user:{user_id}:*",
            f"jobmatch:{user_id}:*"
        ]
        for pattern in patterns:
            await self.invalidate_pattern(pattern)
```

### 4.3 Vector Database (ChromaDB)

**Embedding Storage for Semantic Search:**

```python
from chromadb import Client
from sentence_transformers import SentenceTransformer

class EmbeddingService:
    def __init__(self):
        self.chroma_client = Client()
        self.collection = self.chroma_client.create_collection(
            name="cv_embeddings"
        )
        self.model = SentenceTransformer('all-MiniLM-L6-v2')

    def add_cv_embedding(
        self, cv_id: str, cv_text: str, metadata: dict
    ):
        """Store CV embedding for semantic search"""
        embedding = self.model.encode(cv_text).tolist()

        self.collection.add(
            ids=[cv_id],
            embeddings=[embedding],
            documents=[cv_text],
            metadatas=[metadata]
        )

    def search_similar_cvs(
        self, query_text: str, top_k: int = 10
    ) -> List[dict]:
        """Find semantically similar CVs"""
        query_embedding = self.model.encode(query_text).tolist()

        results = self.collection.query(
            query_embeddings=[query_embedding],
            n_results=top_k
        )

        return results
```

---

## 5. AI Integration Architecture

### 5.1 Multi-Provider AI Strategy

**Provider Abstraction:**
```python
class AIProvider(ABC):
    @abstractmethod
    async def generate_text(
        self, prompt: str, **kwargs
    ) -> str:
        pass

    @abstractmethod
    async def generate_embeddings(
        self, text: str
    ) -> List[float]:
        pass

class OpenAIProvider(AIProvider):
    def __init__(self, api_key: str):
        self.client = OpenAI(api_key=api_key)

    async def generate_text(
        self, prompt: str, **kwargs
    ) -> str:
        response = await self.client.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}],
            **kwargs
        )
        return response.choices[0].message.content

class GeminiProvider(AIProvider):
    def __init__(self, api_key: str):
        self.client = genai.configure(api_key=api_key)

    async def generate_text(
        self, prompt: str, **kwargs
    ) -> str:
        model = genai.GenerativeModel('gemini-pro')
        response = await model.generate_content_async(prompt)
        return response.text

# Fallback mechanism
class AIServiceWithFallback:
    def __init__(
        self,
        primary: AIProvider,
        fallback: AIProvider
    ):
        self.primary = primary
        self.fallback = fallback

    async def generate(self, prompt: str) -> str:
        try:
            return await self.primary.generate_text(prompt)
        except Exception as e:
            logger.warning(f"Primary AI failed: {e}")
            return await self.fallback.generate_text(prompt)
```

### 5.2 Question Generation Pipeline

**AI-Powered Question Creation:**
```python
class QuestionGenerator:
    def __init__(self, ai_service: AIServiceWithFallback):
        self.ai_service = ai_service

    async def generate_questions(
        self,
        cv_text: str,
        job_description: str,
        num_questions: int = 10
    ) -> List[Question]:

        prompt = f"""
        You are an expert technical interviewer. Generate {num_questions}
        interview questions based on the following:

        Candidate CV:
        {cv_text}

        Job Description:
        {job_description}

        Requirements:
        - Mix of technical and behavioral questions
        - Tailored to candidate's experience level
        - Relevant to job requirements
        - Include expected key points for evaluation

        Return JSON array of questions.
        """

        response = await self.ai_service.generate(prompt)
        questions_data = json.loads(response)

        return [
            Question(
                text=q['text'],
                type=q['type'],
                expected_keywords=q['keywords']
            )
            for q in questions_data
        ]
```

---

## 6. Security Architecture

### 6.1 Authentication Flow

```
┌────────────┐                  ┌────────────┐                 ┌────────────┐
│   Client   │                  │   Backend  │                 │  Database  │
└─────┬──────┘                  └─────┬──────┘                 └─────┬──────┘
      │                               │                              │
      │  1. POST /auth/login          │                              │
      │  { email, password }          │                              │
      ├──────────────────────────────>│                              │
      │                               │  2. Verify credentials       │
      │                               ├─────────────────────────────>│
      │                               │                              │
      │                               │  3. User data                │
      │                               │<─────────────────────────────┤
      │                               │                              │
      │                               │  4. Generate JWT tokens      │
      │                               │     - Access token (15min)   │
      │                               │     - Refresh token (7days)  │
      │                               │                              │
      │  5. { access_token,           │                              │
      │      refresh_token, user }    │                              │
      │<──────────────────────────────┤                              │
      │                               │                              │
      │  6. Store tokens securely     │                              │
      │     (httpOnly cookies)        │                              │
      │                               │                              │
      │  7. Authenticated requests    │                              │
      │  Authorization: Bearer <token>│                              │
      ├──────────────────────────────>│                              │
      │                               │  8. Validate JWT             │
      │                               │                              │
      │  9. Protected resource        │                              │
      │<──────────────────────────────┤                              │
```

### 6.2 Authorization Patterns

**Role-Based Access Control (RBAC):**
```python
from enum import Enum
from functools import wraps

class Role(str, Enum):
    CANDIDATE = "candidate"
    HR = "hr"
    ADMIN = "admin"

class Permission(str, Enum):
    VIEW_INTERVIEWS = "view_interviews"
    CREATE_INTERVIEW = "create_interview"
    MANAGE_JOBS = "manage_jobs"
    MANAGE_USERS = "manage_users"

# Permission matrix
ROLE_PERMISSIONS = {
    Role.CANDIDATE: [
        Permission.VIEW_INTERVIEWS,
        Permission.CREATE_INTERVIEW
    ],
    Role.HR: [
        Permission.VIEW_INTERVIEWS,
        Permission.MANAGE_JOBS
    ],
    Role.ADMIN: [
        Permission.VIEW_INTERVIEWS,
        Permission.MANAGE_JOBS,
        Permission.MANAGE_USERS
    ]
}

def require_permission(permission: Permission):
    """Decorator to enforce permissions"""
    def decorator(func):
        @wraps(func)
        async def wrapper(
            *args,
            current_user: User = Depends(get_current_user),
            **kwargs
        ):
            user_permissions = ROLE_PERMISSIONS.get(current_user.role, [])

            if permission not in user_permissions:
                raise HTTPException(
                    status_code=403,
                    detail="Insufficient permissions"
                )

            return await func(*args, current_user=current_user, **kwargs)

        return wrapper
    return decorator

# Usage
@router.post("/jobs")
@require_permission(Permission.MANAGE_JOBS)
async def create_job(
    job_data: JobCreate,
    current_user: User
):
    return await job_service.create(job_data, current_user.id)
```

---

## 7. Performance Optimization Strategies

### 7.1 Database Query Optimization

**Connection Pooling:**
```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

engine = create_async_engine(
    DATABASE_URL,
    pool_size=20,              # Max connections in pool
    max_overflow=10,           # Additional connections if pool is full
    pool_pre_ping=True,        # Verify connection before use
    pool_recycle=3600,         # Recycle connections after 1 hour
    echo=False                 # Set True for query logging
)

async_session_maker = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)
```

**Query Optimization Techniques:**
```python
# Bad: N+1 query problem
async def get_interviews_with_questions_bad(db: AsyncSession):
    interviews = await db.execute(select(Interview))
    for interview in interviews:
        # This triggers a new query for each interview!
        questions = await db.execute(
            select(Question).where(Question.interview_id == interview.id)
        )

# Good: Eager loading with joinedload
from sqlalchemy.orm import joinedload

async def get_interviews_with_questions_good(db: AsyncSession):
    result = await db.execute(
        select(Interview)
        .options(joinedload(Interview.questions))
        .options(joinedload(Interview.candidate))
    )
    return result.unique().scalars().all()
```

### 7.2 API Response Caching

**Cache Decorator:**
```python
from functools import wraps
import hashlib
import json

def cache_response(ttl: int = 300):
    """Cache API responses in Redis"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Generate cache key from function name and arguments
            cache_key = f"api:{func.__name__}:{hashlib.md5(
                json.dumps(kwargs, sort_keys=True).encode()
            ).hexdigest()}"

            # Try to get from cache
            cached = await redis_client.get(cache_key)
            if cached:
                return json.loads(cached)

            # Execute function and cache result
            result = await func(*args, **kwargs)
            await redis_client.setex(
                cache_key,
                ttl,
                json.dumps(result)
            )

            return result

        return wrapper
    return decorator

# Usage
@router.get("/jobs/{job_id}/candidates")
@cache_response(ttl=300)  # Cache for 5 minutes
async def get_job_candidates(
    job_id: int,
    db: AsyncSession = Depends(get_db)
):
    return await job_service.get_candidates(db, job_id)
```

### 7.3 Frontend Performance

**Code Splitting:**
```typescript
// Dynamic imports for heavy components
import dynamic from 'next/dynamic'

const VideoRecorder = dynamic(
  () => import('@/components/VideoRecorder'),
  {
    loading: () => <LoadingSpinner />,
    ssr: false  // Disable SSR for client-only component
  }
)

const ChartComponent = dynamic(
  () => import('@/components/Charts'),
  { ssr: false }
)
```

**Image Optimization:**
```typescript
import Image from 'next/image'

// Automatic optimization with Next.js Image component
<Image
  src="/candidate-avatar.jpg"
  alt="Candidate"
  width={100}
  height={100}
  quality={80}
  placeholder="blur"
  loading="lazy"
/>
```

---

## 8. Monitoring & Observability

### 8.1 Logging Strategy

**Structured Logging:**
```python
import logging
import json
from contextvars import ContextVar

# Request ID context for distributed tracing
request_id_var: ContextVar[str] = ContextVar('request_id', default='')

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            'timestamp': self.formatTime(record),
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
            'request_id': request_id_var.get(),
            'module': record.module,
            'function': record.funcName,
            'line': record.lineno
        }

        if record.exc_info:
            log_data['exception'] = self.formatException(record.exc_info)

        return json.dumps(log_data)

# Configure logger
logger = logging.getLogger(__name__)
handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
logger.addHandler(handler)
logger.setLevel(logging.INFO)
```

### 8.2 Performance Metrics

**Custom Middleware for Metrics:**
```python
import time
from starlette.middleware.base import BaseHTTPMiddleware

class MetricsMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        start_time = time.time()

        # Process request
        response = await call_next(request)

        # Calculate metrics
        duration = time.time() - start_time

        # Log metrics
        logger.info(
            "Request completed",
            extra={
                'method': request.method,
                'path': request.url.path,
                'status_code': response.status_code,
                'duration_ms': duration * 1000
            }
        )

        # Add custom header
        response.headers['X-Response-Time'] = str(duration)

        return response

# Add to FastAPI app
app.add_middleware(MetricsMiddleware)
```

---

## 9. Deployment Architecture

### 9.1 Container Strategy

**Docker Compose Setup:**
```yaml
version: '3.8'

services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=http://backend:8000
    depends_on:
      - backend

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/aiinterview
      - REDIS_URL=redis://redis:6379
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=aiinterview
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### 9.2 CI/CD Pipeline

**GitHub Actions Workflow:**
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov

      - name: Run tests
        run: pytest --cov=./ --cov-report=xml

      - name: Upload coverage
        uses: codecov/codecov-action@v3

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3

      - name: Build Docker images
        run: |
          docker build -t ai-interview-frontend ./frontend
          docker build -t ai-interview-backend ./backend

      - name: Push to registry
        run: |
          # Push to container registry
          # (credentials configured in GitHub secrets)

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Deploy to production
        run: |
          # Deployment commands
          # (SSH into server, pull images, restart containers)
```

---

## 10. Scalability Considerations

### 10.1 Horizontal Scaling Strategy

**Load Balancing:**
```
                    ┌─────────────────┐
                    │  Load Balancer  │
                    │     (Nginx)     │
                    └────────┬────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
     ┌────▼─────┐      ┌────▼─────┐      ┌────▼─────┐
     │ Backend  │      │ Backend  │      │ Backend  │
     │Instance 1│      │Instance 2│      │Instance 3│
     └────┬─────┘      └────┬─────┘      └────┬─────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
                    ┌────────▼────────┐
                    │   PostgreSQL    │
                    │  (Primary/Read  │
                    │    Replicas)    │
                    └─────────────────┘
```

### 10.2 Caching Layers

**Multi-level Caching:**
1. **Browser Cache**: Static assets (images, JS, CSS)
2. **CDN Cache**: Global content delivery
3. **Application Cache** (Redis): API responses, session data
4. **Database Query Cache**: Frequently accessed queries

---

**Document Version:** 1.0
**Last Updated:** November 2025
**Maintained By:** Tanvir Rahman Anik
