# Frontend Structure

## Overview

PrimeFlow has two Angular applications:
1. **ts-webapp-frontend**: Multi-tenant web application (Freya template)
2. **ts-website-frontend**: Marketing landing page

Both use Angular 19 with standalone components and signals.

---

## WebApp Frontend (ts-webapp-frontend)

### Technology Stack
- **Framework**: Angular 19
- **Template**: Freya (PrimeNG)
- **UI Library**: PrimeNG 19
- **State Management**: Angular Signals
- **Routing**: Angular Router with guards
- **HTTP**: HttpClient with interceptors
- **Forms**: Reactive Forms

### Project Structure

```
ts-webapp-frontend/
├── src/
│   ├── app/
│   │   ├── core/                    # Core functionality
│   │   │   ├── guards/              # Route guards
│   │   │   │   ├── auth.guard.ts    # Protects authenticated routes
│   │   │   │   └── login.guard.ts   # Prevents access to login when authenticated
│   │   │   ├── interceptors/        # HTTP interceptors
│   │   │   │   └── auth.interceptor.ts  # Adds JWT token to requests
│   │   │   ├── models/              # Data models
│   │   │   │   ├── auth.model.ts    # Auth DTOs
│   │   │   │   └── user.interface.ts # User interfaces
│   │   │   └── services/            # Core services
│   │   │       ├── auth.service.ts  # Authentication
│   │   │       ├── cookie.service.ts # Cookie management
│   │   │       ├── user.service.ts  # User CRUD
│   │   │       └── api.service.ts   # Base API service
│   │   │
│   │   ├── features/                # Feature modules
│   │   │   ├── auth/                # Authentication feature
│   │   │   │   ├── components/
│   │   │   │   │   ├── login.ts     # Login page
│   │   │   │   │   ├── signup.ts    # Signup page
│   │   │   │   │   └── forgotpassword.ts
│   │   │   │   └── auth.routes.ts   # Auth routes
│   │   │   │
│   │   │   ├── dashboard/           # Dashboard feature
│   │   │   │   ├── components/
│   │   │   │   │   └── dashboard.ts
│   │   │   │   └── dashboard.routes.ts
│   │   │   │
│   │   │   └── settings/            # Settings feature
│   │   │       ├── components/
│   │   │       │   └── users.ts     # User management
│   │   │       └── settings.routes.ts
│   │   │
│   │   ├── layout/                  # Layout components
│   │   │   ├── app.layout.component.ts
│   │   │   ├── app.menu.component.ts
│   │   │   ├── app.topbar.component.ts
│   │   │   └── app.sidebar.component.ts
│   │   │
│   │   ├── shared/                  # Shared components
│   │   │   ├── smart-grid/          # Reusable grid component
│   │   │   └── smart-form/          # Reusable form component
│   │   │
│   │   ├── app.component.ts         # Root component
│   │   ├── app.config.ts            # App configuration
│   │   └── app.routes.ts            # Root routes
│   │
│   ├── environments/                # Environment configs
│   │   ├── environment.ts           # Production
│   │   └── environment.dev.ts       # Development
│   │
│   ├── assets/                      # Static assets
│   ├── styles/                      # Global styles
│   └── index.html                   # HTML entry point
│
├── angular.json                     # Angular configuration
├── package.json                     # Dependencies
└── tsconfig.json                    # TypeScript configuration
```

---

## Core Services

### AuthService

Handles authentication and user session management.

**Location:** `src/app/core/services/auth.service.ts`

**Key Methods:**
```typescript
// Login user
login(email: string, password: string): Observable<User>

// Register new tenant
signup(signupData: SignupRequest): Observable<User>

// Logout user
logout(): void

// Check authentication status
isAuthenticated(): boolean

// Get current user
getUser(): User | null

// Get backend user data
getBackendUser(): UserInfo | null

// Get tenant data
getTenant(): TenantInfo | null

// Get JWT token
getToken(): string | null

// Refresh auth state from cookies
refreshAuthState(): void
```

**State Management:**
```typescript
private authState = signal<AuthState>({
    isAuthenticated: false,
    user: null,
    token: null,
    loading: false,
    error: null
});
```

**Cookie Storage:**
- `auth_token`: JWT token (7 days)
- `user_data`: App user data
- `backend_user`: Backend user data
- `tenant_data`: Tenant information

---

### CookieService

Manages browser cookies with proper domain handling.

**Location:** `src/app/core/services/cookie.service.ts`

**Key Methods:**
```typescript
// Set cookie
setCookie(name: string, value: string, options?: CookieOptions): Observable<{message: string}>

// Get cookie
getCookie(name: string): string | null

// Remove cookie
removeCookie(name: string): Observable<{message: string}>

// Set auth token
setAuthCookie(authToken: string): Observable<{message: string}>

// Set user data
setUserData(userData: any): Observable<{message: string}>

// Clear all auth cookies
clearAuthCookies(): Observable<any>

// Sync with browser
syncWithBrowser(): void
```

**Cookie Options:**
```typescript
interface CookieOptions {
  expires?: Date | string;
  path?: string;
  domain?: string;
  secure?: boolean;
  sameSite?: 'strict' | 'lax' | 'none';
  maxAge?: number;
}
```

---

### UserService

Handles user CRUD operations.

**Location:** `src/app/core/services/user.service.ts`

**Key Methods:**
```typescript
// List all users
list(): Observable<UserResponse[]>

// Create user
create(user: CreateUserRequest): Observable<UserResponse>

// Update user
update(id: number, user: UpdateUserRequest): Observable<UserResponse>

// Delete user
delete(id: number): Observable<void>
```

**API Integration:**
```typescript
private apiUrl = `${environment.apiUrl}/users/v1/users`;
```

---

## Route Guards

### AuthGuard

Protects routes that require authentication.

**Location:** `src/app/core/guards/auth.guard.ts`

**Usage:**
```typescript
{
  path: 'dashboard',
  canActivate: [authGuard],
  loadComponent: () => import('./features/dashboard/components/dashboard')
}
```

**Behavior:**
- Checks if user is authenticated
- Redirects to `/auth/login` if not authenticated
- Allows access if authenticated

---

### LoginGuard

Prevents authenticated users from accessing login/signup pages.

**Location:** `src/app/core/guards/login.guard.ts`

**Usage:**
```typescript
{
  path: 'auth',
  canActivate: [loginGuard],
  children: [...]
}
```

**Behavior:**
- Checks if user is authenticated
- Redirects to `/dashboard` if authenticated
- Allows access if not authenticated

---

## HTTP Interceptors

### AuthInterceptor

Automatically adds JWT token to API requests.

**Location:** `src/app/core/interceptors/auth.interceptor.ts`

**Behavior:**
```typescript
intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
  const token = this.authService.getToken();

  // Skip for login/signup
  if (req.url.includes('/login') || req.url.includes('/signup')) {
    return next.handle(req);
  }

  // Add Authorization header
  if (token) {
    const clonedReq = req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`
      }
    });
    return next.handle(clonedReq);
  }

  return next.handle(req);
}
```

**Note:** The gateway extracts `tenant_slug` from JWT and injects `X-Tenant-Slug` header automatically.

---

## Routing Structure

### Root Routes

**Location:** `src/app/app.routes.ts`

```typescript
export const routes: Routes = [
  {
    path: '',
    component: AppLayoutComponent,
    canActivate: [authGuard],
    children: [
      {
        path: '',
        redirectTo: 'dashboard',
        pathMatch: 'full'
      },
      {
        path: 'dashboard',
        loadChildren: () => import('./features/dashboard/dashboard.routes')
      },
      {
        path: 'settings',
        loadChildren: () => import('./features/settings/settings.routes')
      }
    ]
  },
  {
    path: 'auth',
    canActivate: [loginGuard],
    loadChildren: () => import('./features/auth/auth.routes')
  },
  {
    path: '**',
    redirectTo: 'dashboard'
  }
];
```

### Auth Routes

**Location:** `src/app/features/auth/auth.routes.ts`

```typescript
export const AUTH_ROUTES: Routes = [
  {
    path: 'login',
    loadComponent: () => import('./components/login')
  },
  {
    path: 'signup',
    loadComponent: () => import('./components/signup')
  },
  {
    path: 'forgotpassword',
    loadComponent: () => import('./components/forgotpassword')
  }
];
```

### Settings Routes

**Location:** `src/app/features/settings/settings.routes.ts`

```typescript
export const SETTINGS_ROUTES: Routes = [
  {
    path: 'users',
    loadComponent: () => import('./components/users')
  }
];
```

---

## Component Patterns

### Standalone Components

All components use Angular 19 standalone pattern:

```typescript
@Component({
  selector: 'app-login',
  standalone: true,
  imports: [
    CommonModule,
    ReactiveFormsModule,
    ButtonModule,
    InputTextModule
  ],
  templateUrl: './login.html',
  styleUrls: ['./login.scss']
})
export class LoginComponent {
  // Component logic
}
```

### Signal-Based State

Using Angular signals for reactive state:

```typescript
export class UsersComponent {
  users = signal<UserResponse[]>([]);
  loading = signal<boolean>(false);
  error = signal<string | null>(null);

  loadUsers() {
    this.loading.set(true);
    this.userService.list().subscribe({
      next: (users) => {
        this.users.set(users);
        this.loading.set(false);
      },
      error: (err) => {
        this.error.set(err.message);
        this.loading.set(false);
      }
    });
  }
}
```

---

## Shared Components

### SmartGrid

Reusable data grid component with CRUD operations.

**Location:** `src/app/shared/smart-grid/`

**Features:**
- Pagination
- Sorting
- Filtering
- Row actions (view, edit, delete)
- Custom actions
- Loading states

**Usage:**
```typescript
<app-smart-grid
  [data]="users()"
  [columns]="columns"
  [actions]="actions"
  (onAction)="handleAction($event)"
/>
```

### SmartForm

Reusable form component with validation.

**Location:** `src/app/shared/smart-form/`

**Features:**
- Dynamic field generation
- Validation
- Error messages
- Loading states
- Submit handling

**Usage:**
```typescript
<app-smart-form
  [fields]="formFields"
  [loading]="loading()"
  (onSubmit)="handleSubmit($event)"
/>
```

---

## Environment Configuration

### Development

**Location:** `src/environments/environment.dev.ts`

```typescript
export const environment = {
  production: false,
  apiUrl: 'http://api.localhost',
  cookieDomain: '.localhost'
};
```

### Production

**Location:** `src/environments/environment.ts`

```typescript
export const environment = {
  production: true,
  apiUrl: 'https://api.primeflow.solutions',
  cookieDomain: '.primeflow.solutions'
};
```

---

## Styling

### PrimeNG Theme

Using Freya theme from PrimeNG:

```scss
@import 'primeng/resources/themes/lara-light-blue/theme.css';
@import 'primeng/resources/primeng.css';
@import 'primeicons/primeicons.css';
```

### Custom Styles

**Location:** `src/styles/`

- `styles.scss`: Global styles
- `layout.scss`: Layout-specific styles
- `variables.scss`: SCSS variables

---

## Website Frontend (ts-website-frontend)

### Purpose
Marketing landing page with:
- Hero section
- Features section
- Pricing section
- Contact section

### Structure

```
ts-website-frontend/
├── src/
│   ├── app/
│   │   ├── landing-page/
│   │   │   ├── navbar/
│   │   │   ├── hero-section/
│   │   │   ├── features-section/
│   │   │   ├── pricing-section/
│   │   │   └── footer/
│   │   └── app.component.ts
│   └── environments/
```

### Signup Redirect

All CTA buttons redirect to webapp signup:

```typescript
redirectToSignup() {
  const isLocalhost = window.location.hostname === 'localhost';
  const signupUrl = isLocalhost 
    ? 'http://app.localhost/auth/signup'
    : 'https://app.primeflow.solutions/auth/signup';
  window.location.href = signupUrl;
}
```

---

## Build & Deployment

### Development Build

```bash
cd ts-webapp-frontend
npm run start
# Access at http://localhost:4200
```

### Production Build

```bash
npm run build
# Output: dist/ts-webapp-frontend/
```

### Docker Build

```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist/ts-webapp-frontend /usr/share/nginx/html
EXPOSE 80
```

---

## Testing

### Unit Tests

```bash
npm run test
```

### E2E Tests

```bash
npm run e2e
```

---

## Best Practices

### Component Design
- Use standalone components
- Keep components small and focused
- Use signals for reactive state
- Implement OnPush change detection

### Service Design
- Inject services via `inject()` function
- Use observables for async operations
- Handle errors gracefully
- Provide loading states

### Routing
- Use lazy loading for features
- Implement route guards
- Use route resolvers for data fetching

### State Management
- Use signals for local state
- Use services for shared state
- Avoid global state when possible

### Performance
- Lazy load routes
- Use OnPush change detection
- Optimize bundle size
- Use trackBy in *ngFor
