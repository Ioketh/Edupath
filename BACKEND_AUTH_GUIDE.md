# EduPath Uganda — Backend Authentication & AI Training Guide

## Overview of Changes Made in V4

### What was removed
- All references to the document author (name, email, "Kaziba Stephen" branding)
- Universities page → replaced with **School Advertising** page

### What was improved
- AI Advisor system prompt massively upgraded with deep Uganda education context
- School advertising system with 3 tiers (Free, Featured UGX 150k, Premium UGX 400k)
- PUJAB subsidiary rules clarified throughout

---

## PART 1: AUTHENTICATION OPTIONS

You have two solid choices. Here is an honest comparison:

### Option A: Django + JWT (Recommended for production)

**Best for:** Full-featured app, real database, multiple users, school admin accounts

**Tech stack:**
- Django 5.x (Python web framework)
- djangorestframework (REST API)
- djangorestframework-simplejwt (JSON Web Tokens)
- PostgreSQL (database)
- django-cors-headers (allow frontend to call API)

**How JWT works:**
1. School admin logs in with email + password
2. Server verifies credentials → returns `access_token` (expires in 15-60 min) + `refresh_token` (expires in 7 days)
3. Frontend stores tokens in memory (NOT localStorage — security risk)
4. Every API request includes `Authorization: Bearer <access_token>` header
5. When access token expires, frontend silently refreshes using refresh token

---

## PART 2: DJANGO SETUP — STEP BY STEP

### Step 1: Install dependencies
```bash
pip install django djangorestframework djangorestframework-simplejwt \
            django-cors-headers psycopg2-binary python-dotenv \
            gunicorn whitenoise pillow
```

### Step 2: Project structure
```
edupath/
├── manage.py
├── requirements.txt
├── .env
├── edupath/
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── accounts/          ← handles school auth
│   ├── models.py
│   ├── serializers.py
│   ├── views.py
│   └── urls.py
├── students/          ← student records
│   ├── models.py
│   ├── serializers.py
│   └── views.py
└── advertising/       ← school ad listings
    ├── models.py
    └── views.py
```

### Step 3: settings.py (key parts)
```python
import os
from pathlib import Path
from datetime import timedelta
from dotenv import load_dotenv

load_dotenv()

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    # Third party
    'rest_framework',
    'rest_framework_simplejwt',
    'rest_framework_simplejwt.token_blacklist',
    'corsheaders',
    # Our apps
    'accounts',
    'students',
    'advertising',
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',  # Must be FIRST
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',
    # ... rest of Django middleware
]

# Database
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DB_NAME', 'edupath_db'),
        'USER': os.getenv('DB_USER', 'postgres'),
        'PASSWORD': os.getenv('DB_PASSWORD'),
        'HOST': os.getenv('DB_HOST', 'localhost'),
        'PORT': os.getenv('DB_PORT', '5432'),
    }
}

# Django REST Framework
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ),
    'DEFAULT_PERMISSION_CLASSES': (
        'rest_framework.permissions.IsAuthenticated',
    ),
}

# JWT Configuration
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=60),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'AUTH_HEADER_TYPES': ('Bearer',),
}

# CORS — allow your frontend domain
CORS_ALLOWED_ORIGINS = [
    "http://localhost:3000",
    "https://yourdomain.com",
    "https://your-github-pages-url.github.io",
]

# Allow credentials (needed for JWT refresh)
CORS_ALLOW_CREDENTIALS = True
```

### Step 4: Custom School User Model (accounts/models.py)
```python
from django.contrib.auth.models import AbstractBaseUser, BaseUserManager, PermissionsMixin
from django.db import models

class SchoolUserManager(BaseUserManager):
    def create_user(self, email, school_name, password=None, **extra_fields):
        if not email:
            raise ValueError('Email is required')
        email = self.normalize_email(email)
        user = self.model(email=email, school_name=school_name, **extra_fields)
        user.set_password(password)
        user.save(using=self._db)
        return user

    def create_superuser(self, email, school_name, password, **extra_fields):
        extra_fields.setdefault('is_staff', True)
        extra_fields.setdefault('is_superuser', True)
        return self.create_user(email, school_name, password, **extra_fields)


class SchoolUser(AbstractBaseUser, PermissionsMixin):
    SCHOOL_TYPES = [
        ('government', 'Government'),
        ('private', 'Private'),
        ('catholic', 'Religious (Catholic)'),
        ('protestant', 'Religious (Protestant)'),
        ('islamic', 'Religious (Islamic)'),
    ]

    email = models.EmailField(unique=True)
    school_name = models.CharField(max_length=200)
    school_type = models.CharField(max_length=20, choices=SCHOOL_TYPES)
    district = models.CharField(max_length=100)
    phone = models.CharField(max_length=20)
    is_verified = models.BooleanField(default=False)  # Admin verifies school
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    objects = SchoolUserManager()

    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['school_name']

    def __str__(self):
        return f"{self.school_name} ({self.email})"
```

### Step 5: Auth Views (accounts/views.py)
```python
from rest_framework import status
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import AllowAny, IsAuthenticated
from rest_framework.response import Response
from rest_framework_simplejwt.tokens import RefreshToken
from rest_framework_simplejwt.exceptions import TokenError
from django.contrib.auth import authenticate
from .models import SchoolUser
from .serializers import SchoolUserSerializer, RegisterSerializer

@api_view(['POST'])
@permission_classes([AllowAny])
def register_school(request):
    """Register a new school account"""
    serializer = RegisterSerializer(data=request.data)
    if serializer.is_valid():
        user = serializer.save()
        refresh = RefreshToken.for_user(user)
        return Response({
            'message': 'School registered. Pending admin verification.',
            'access': str(refresh.access_token),
            'refresh': str(refresh),
            'school': SchoolUserSerializer(user).data
        }, status=status.HTTP_201_CREATED)
    return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


@api_view(['POST'])
@permission_classes([AllowAny])
def login_school(request):
    """Login with email + password → returns JWT tokens"""
    email = request.data.get('email')
    password = request.data.get('password')
    
    user = authenticate(request, email=email, password=password)
    if user is None:
        return Response(
            {'error': 'Invalid email or password'}, 
            status=status.HTTP_401_UNAUTHORIZED
        )
    if not user.is_verified:
        return Response(
            {'error': 'Your school account is pending verification by EduPath admin.'},
            status=status.HTTP_403_FORBIDDEN
        )
    
    refresh = RefreshToken.for_user(user)
    return Response({
        'access': str(refresh.access_token),
        'refresh': str(refresh),
        'school': SchoolUserSerializer(user).data
    })


@api_view(['POST'])
@permission_classes([IsAuthenticated])
def logout_school(request):
    """Blacklist the refresh token on logout"""
    try:
        refresh_token = request.data.get('refresh')
        token = RefreshToken(refresh_token)
        token.blacklist()
        return Response({'message': 'Logged out successfully'})
    except TokenError:
        return Response({'error': 'Invalid token'}, status=status.HTTP_400_BAD_REQUEST)


@api_view(['GET'])
@permission_classes([IsAuthenticated])
def get_profile(request):
    """Get current school's profile"""
    return Response(SchoolUserSerializer(request.user).data)
```

### Step 6: Student Model (students/models.py)
```python
from django.db import models
from accounts.models import SchoolUser

class Student(models.Model):
    GRADES = [('A','A'),('B','B'),('C','C'),('D','D'),('E','E'),('F','F')]
    
    school = models.ForeignKey(SchoolUser, on_delete=models.CASCADE, related_name='students')
    name = models.CharField(max_length=200)
    mathematics = models.CharField(max_length=1, choices=GRADES, blank=True)
    physics = models.CharField(max_length=1, choices=GRADES, blank=True)
    chemistry = models.CharField(max_length=1, choices=GRADES, blank=True)
    biology = models.CharField(max_length=1, choices=GRADES, blank=True)
    english_language = models.CharField(max_length=1, choices=GRADES, blank=True)
    geography = models.CharField(max_length=1, choices=GRADES, blank=True)
    history = models.CharField(max_length=1, choices=GRADES, blank=True)
    economics = models.CharField(max_length=1, choices=GRADES, blank=True)
    literature = models.CharField(max_length=1, choices=GRADES, blank=True)
    career_interest = models.CharField(max_length=200, blank=True)
    best_combination = models.CharField(max_length=20, blank=True)
    match_percentage = models.IntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f"{self.name} — {self.school.school_name}"
```

### Step 7: URL Configuration (edupath/urls.py)
```python
from django.contrib import admin
from django.urls import path, include
from rest_framework_simplejwt.views import TokenRefreshView

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/auth/', include('accounts.urls')),
    path('api/auth/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/students/', include('students.urls')),
    path('api/advertising/', include('advertising.urls')),
]
```

### Step 8: accounts/urls.py
```python
from django.urls import path
from . import views

urlpatterns = [
    path('register/', views.register_school, name='register'),
    path('login/', views.login_school, name='login'),
    path('logout/', views.logout_school, name='logout'),
    path('profile/', views.get_profile, name='profile'),
]
```

---

## PART 3: CONNECTING FRONTEND TO DJANGO

### In your HTML file, replace the school login function:
```javascript
// Store tokens in memory (NOT localStorage — XSS risk)
let AUTH = { access: null, refresh: null, school: null };

async function schoolLogin() {
    const email = document.getElementById('s-email').value;
    const password = document.getElementById('s-pass').value;
    const schoolName = document.getElementById('s-school-name').value;
    
    try {
        const res = await fetch('https://your-api.onrender.com/api/auth/login/', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ email, password })
        });
        const data = await res.json();
        
        if (!res.ok) {
            alert(data.error || 'Login failed. Check your credentials.');
            return;
        }
        
        // Store in memory
        AUTH.access = data.access;
        AUTH.refresh = data.refresh;
        AUTH.school = data.school;
        
        // Go to dashboard
        document.getElementById('schName').textContent = data.school.school_name;
        go('school');
        
    } catch (err) {
        alert('Cannot connect to server. Check your internet connection.');
    }
}

// Helper: make authenticated API requests
async function apiCall(url, method='GET', body=null) {
    const headers = {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${AUTH.access}`
    };
    const opts = { method, headers };
    if (body) opts.body = JSON.stringify(body);
    
    let res = await fetch(url, opts);
    
    // If token expired, refresh it
    if (res.status === 401 && AUTH.refresh) {
        const refreshRes = await fetch('/api/auth/token/refresh/', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ refresh: AUTH.refresh })
        });
        if (refreshRes.ok) {
            const tokens = await refreshRes.json();
            AUTH.access = tokens.access;
            // Retry original request
            headers['Authorization'] = `Bearer ${AUTH.access}`;
            res = await fetch(url, { method, headers, body: body ? JSON.stringify(body) : null });
        } else {
            // Refresh also expired — force logout
            AUTH = { access: null, refresh: null, school: null };
            go('school-auth');
            return null;
        }
    }
    return res.json();
}
```

---

## PART 4: OPTION B — SUPABASE (Simpler Alternative to Django)

If Django feels complex, **Supabase** is a much faster alternative:

**What Supabase gives you:**
- PostgreSQL database (free tier: 500MB)
- Built-in authentication (email/password, Google, etc.)
- Auto-generated REST API
- Real-time subscriptions
- Free hosting

**Setup:**
```javascript
// 1. Install: npm install @supabase/supabase-js
// 2. In your HTML:
import { createClient } from 'https://cdn.jsdelivr.net/npm/@supabase/supabase-js/+esm'

const supabase = createClient(
    'https://YOUR_PROJECT.supabase.co',
    'YOUR_ANON_KEY'
);

// School login
async function schoolLogin() {
    const { data, error } = await supabase.auth.signInWithPassword({
        email: document.getElementById('s-email').value,
        password: document.getElementById('s-pass').value
    });
    if (error) { alert(error.message); return; }
    go('school');
}

// Save student to database
async function saveStudent(studentData) {
    const { error } = await supabase.from('students').insert(studentData);
    if (error) console.error(error);
}

// Get school's students
async function getStudents() {
    const { data } = await supabase
        .from('students')
        .select('*')
        .order('created_at', { ascending: false });
    return data;
}
```

**Recommendation:** Start with Supabase → move to Django when you need more control.

---

## PART 5: HOW TO TRAIN / CUSTOMIZE THE AI

The AI (Claude) is guided by a **system prompt** — a set of instructions that shape every response. You have already have a detailed prompt in the code. Here are ways to improve it further:

### Method 1: Enhance the System Prompt (Free, in-code)
The system prompt in the HTML file is the primary way to train the AI behavior. Add:

```javascript
// Add to SYSTEM_PROMPT:
`
## SCHOOL-SPECIFIC CONTEXT
This system is used by schools in Arua City, West Nile region. 
Always mention schools in West Nile when students ask about local options.
Key local schools: Arua Public S.S.S, St. Joseph's Ombaci, Mvara S.S.S, 
                   Sacred Heart Girls, Ediofe Girls, Muni Girls High School.

## WHEN TO RECOMMEND VOCATIONAL ROUTES
If a student has Division III or IV, always mention:
1. BTVET programmes (Uganda Business and Technical Examinations Board)
2. UBTEB certificates in: Electrical, Building, Plumbing, Business, ICT
3. These can lead to good careers and even university later via mature-age entry
`
```

### Method 2: Add Student Context to Prompts
Before the AI call, include the student's grades:
```javascript
async function sendChat() {
    const grades = Object.keys(GRADES).length > 0 
        ? `\n\n[STUDENT CONTEXT: This student has entered these O-Level grades: ${JSON.stringify(GRADES)}. Their top combination is ${rankCombos()[0]?.code || 'not yet calculated'}.]`
        : '';
    
    // Prepend grades to user message so AI has context
    const messageWithContext = msg + grades;
    messages.push({ role: 'user', content: messageWithContext });
    // ... rest of API call
}
```

### Method 3: Fine-Tuning (Advanced, costs money)
For a fully custom AI trained on Uganda education data:
1. Collect 1,000+ Uganda-specific Q&A pairs (student questions + expert answers)
2. Format as JSONL: `{"prompt": "...", "completion": "..."}`
3. Use Anthropic's fine-tuning API (contact Anthropic for access) OR
4. Use OpenAI fine-tuning (cheaper): `openai.fine_tuning.jobs.create()`
5. This creates a model that "knows" Uganda education by default

**Realistic approach for now:** The system prompt method is excellent and costs nothing extra. The prompt in the code is already very detailed.

---

## PART 6: DEPLOYMENT GUIDE

### Frontend (GitHub Pages — Free)
```bash
git init
git add .
git commit -m "EduPath v4 launch"
git branch -M main
git remote add origin https://github.com/YOURNAME/edupath.git
git push -u origin main
# Then: GitHub repo → Settings → Pages → Deploy from main branch
```

### Backend Django (Render.com — Free tier available)
```bash
# requirements.txt
django>=5.0
djangorestframework>=3.15
djangorestframework-simplejwt>=5.3
django-cors-headers>=4.3
psycopg2-binary>=2.9
gunicorn>=21.0
whitenoise>=6.6
python-dotenv>=1.0

# Procfile
web: gunicorn edupath.wsgi --log-file -

# render.yaml
services:
  - type: web
    name: edupath-api
    env: python
    buildCommand: pip install -r requirements.txt && python manage.py migrate
    startCommand: gunicorn edupath.wsgi
    envVars:
      - key: SECRET_KEY
        generateValue: true
      - key: DATABASE_URL
        fromDatabase:
          name: edupath-db
          property: connectionString
      - key: ALLOWED_HOSTS
        value: edupath-api.onrender.com
      - key: DEBUG
        value: false
databases:
  - name: edupath-db
    plan: free
```

### Backend Supabase (Instant — No server needed)
1. Go to supabase.com → New Project
2. Create tables in SQL editor (use the models above as reference)
3. Get `Project URL` and `anon public` key
4. Paste into your HTML file — done!

---

## PART 7: SECURITY CHECKLIST

- [ ] Never put API keys in the HTML frontend (Claude API key should be on your server)
- [ ] Use HTTPS everywhere (Render.com and GitHub Pages do this automatically)
- [ ] Store JWT tokens in memory, not localStorage
- [ ] Set `DEBUG = False` in Django production
- [ ] Add `ALLOWED_HOSTS` to settings.py
- [ ] Use `django-environ` or `.env` files for secrets
- [ ] Enable CORS only for your specific frontend domain
- [ ] Verify school accounts manually before granting dashboard access

---

## PART 8: RECOMMENDED DEVELOPMENT SEQUENCE

1. **Week 1:** Deploy the HTML file on GitHub Pages — works as a complete standalone app
2. **Week 2:** Set up Supabase (free) → add school registration + login
3. **Week 3:** Connect student upload/storage to Supabase database
4. **Week 4:** Set up a simple Django API on Render.com for PDF generation
5. **Month 2:** Move all auth to Django + JWT for full control
6. **Month 3:** Add school advertising payment integration (MTN MoMo API or Flutterwave)

---

*EduPath Uganda v4.0 — Backend Guide*
*For technical support, review this document and the in-code comments carefully.*
