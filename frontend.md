# Frontend Best Practices

A comprehensive guide to building modern, performant, and accessible frontend applications based on web standards and industry best practices.

## Table of Contents

1. [Project Structure](#project-structure)
2. [Performance](#performance)
3. [Accessibility](#accessibility)
4. [State Management](#state-management)
5. [Routing](#routing)
6. [Forms & Validation](#forms--validation)
7. [API Integration](#api-integration)
8. [Error Handling](#error-handling)
9. [Testing](#testing)
10. [Security](#security)
11. [Build & Deployment](#build--deployment)
12. [Developer Experience](#developer-experience)

---

## Project Structure

Organize your codebase for maintainability and scalability.

### Recommended Structure

```
src/
├── assets/          # Static files (images, fonts, icons)
├── components/      # Reusable UI components
│   ├── common/      # Shared components (Button, Input, etc.)
│   ├── layout/      # Layout components (Header, Footer, Sidebar)
│   └── features/    # Feature-specific components
├── hooks/           # Custom React hooks / composables
├── services/        # API services and external integrations
├── utils/           # Helper functions and utilities
├── types/           # TypeScript types and interfaces
├── styles/          # Global styles and theme
├── pages/           # Page components / route views
├── store/           # State management (Redux, Zustand, etc.)
├── config/          # Configuration files
├── constants/       # Application constants
└── tests/           # Test utilities and setup
```

### Component Organization

```
components/
└── Button/
    ├── Button.tsx          # Component implementation
    ├── Button.test.tsx     # Tests
    ├── Button.stories.tsx  # Storybook stories
    ├── Button.module.css   # Component styles
    └── index.ts            # Public exports
```

### Best Practices

- Use feature-based folders for large applications
- Keep components small and focused (Single Responsibility)
- Separate business logic from UI components
- Use barrel exports (index.ts) for cleaner imports
- Colocate related files (component, styles, tests)
- Avoid deep nesting (max 3-4 levels)

---

## Performance

Optimize for speed and user experience.

### Core Web Vitals

Monitor and optimize these key metrics:

**Largest Contentful Paint (LCP)** - < 2.5s
- Optimize images (WebP, AVIF formats)
- Use lazy loading
- Minimize render-blocking resources
- Use CDN for static assets

**First Input Delay (FID)** - < 100ms
- Minimize JavaScript execution
- Code splitting
- Use web workers for heavy computations
- Optimize event handlers

**Cumulative Layout Shift (CLS)** - < 0.1
- Set explicit dimensions for images/videos
- Avoid inserting content above existing content
- Use CSS transforms instead of layout properties

### Code Splitting

```javascript
// Route-based splitting
const Home = lazy(() => import('./pages/Home'));
const About = lazy(() => import('./pages/About'));
const Dashboard = lazy(() => import('./pages/Dashboard'));

// Component-based splitting
const HeavyChart = lazy(() => import('./components/HeavyChart'));
```

### Image Optimization

```jsx
// Responsive images
<img
  src="/images/hero.jpg"
  srcSet="
    /images/hero-320w.jpg 320w,
    /images/hero-640w.jpg 640w,
    /images/hero-1280w.jpg 1280w
  "
  sizes="(max-width: 320px) 280px,
         (max-width: 640px) 600px,
         1200px"
  alt="Hero image"
  loading="lazy"
  decoding="async"
/>

// Modern formats with fallback
<picture>
  <source srcset="/images/hero.avif" type="image/avif" />
  <source srcset="/images/hero.webp" type="image/webp" />
  <img src="/images/hero.jpg" alt="Hero image" />
</picture>
```

### Lazy Loading

```javascript
// Intersection Observer for lazy loading
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src;
      observer.unobserve(img);
    }
  });
});

document.querySelectorAll('img[data-src]').forEach(img => {
  observer.observe(img);
});
```

### Memoization

```javascript
// React.memo for component memoization
const ExpensiveComponent = React.memo(({ data }) => {
  return <div>{/* expensive render */}</div>;
});

// useMemo for expensive calculations
const expensiveValue = useMemo(() => {
  return computeExpensiveValue(a, b);
}, [a, b]);

// useCallback for function memoization
const handleClick = useCallback(() => {
  doSomething(a, b);
}, [a, b]);
```

### Bundle Optimization

```javascript
// Webpack configuration
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: 10
        }
      }
    }
  }
};
```

### Best Practices

- Compress assets (gzip, brotli)
- Use HTTP/2 or HTTP/3
- Implement service workers for caching
- Minimize third-party scripts
- Use resource hints (preload, prefetch, preconnect)
- Optimize fonts (subset, preload)
- Remove unused CSS/JS
- Use tree shaking
- Monitor bundle size

---

## Accessibility

Build inclusive applications for all users.

### Semantic HTML

```html
<!-- Good: Semantic and accessible -->
<nav>
  <ul>
    <li><a href="/home">Home</a></li>
    <li><a href="/about">About</a></li>
  </ul>
</nav>

<main>
  <article>
    <h1>Article Title</h1>
    <p>Content...</p>
  </article>
</main>

<aside>
  <h2>Related Links</h2>
</aside>

<!-- Bad: Generic divs -->
<div class="nav">
  <div class="link">Home</div>
</div>
```

### ARIA Attributes

```jsx
// Button with accessible name
<button aria-label="Close dialog">
  <CloseIcon />
</button>

// Live region for dynamic content
<div aria-live="polite" aria-atomic="true">
  {statusMessage}
</div>

// Disclosure widget
<button
  aria-expanded={isOpen}
  aria-controls="dropdown-menu"
  onClick={toggle}
>
  Menu
</button>
<div id="dropdown-menu" hidden={!isOpen}>
  {/* menu items */}
</div>

// Form with proper labels
<label htmlFor="email">Email Address</label>
<input
  id="email"
  type="email"
  aria-required="true"
  aria-invalid={hasError}
  aria-describedby={hasError ? "email-error" : undefined}
/>
{hasError && <span id="email-error" role="alert">{errorMessage}</span>}
```

### Keyboard Navigation

```javascript
// Ensure interactive elements are keyboard accessible
const handleKeyDown = (e) => {
  if (e.key === 'Enter' || e.key === ' ') {
    e.preventDefault();
    handleClick();
  }
};

<div
  role="button"
  tabIndex={0}
  onClick={handleClick}
  onKeyDown={handleKeyDown}
>
  Click me
</div>
```

### Focus Management

```javascript
// Trap focus in modal
useEffect(() => {
  if (isOpen) {
    const focusableElements = modal.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    const firstElement = focusableElements[0];
    const lastElement = focusableElements[focusableElements.length - 1];
    
    firstElement?.focus();
    
    const handleTab = (e) => {
      if (e.key === 'Tab') {
        if (e.shiftKey && document.activeElement === firstElement) {
          e.preventDefault();
          lastElement?.focus();
        } else if (!e.shiftKey && document.activeElement === lastElement) {
          e.preventDefault();
          firstElement?.focus();
        }
      }
    };
    
    modal.addEventListener('keydown', handleTab);
    return () => modal.removeEventListener('keydown', handleTab);
  }
}, [isOpen]);
```

### Color Contrast

```css
/* Ensure minimum contrast ratios */
/* WCAG AA: 4.5:1 for normal text, 3:1 for large text */
/* WCAG AAA: 7:1 for normal text, 4.5:1 for large text */

.button {
  background-color: #0066cc;
  color: #ffffff; /* 7.3:1 contrast ratio */
}

.text {
  color: #333333; /* Against white: 12.6:1 */
}
```

### Screen Reader Support

```jsx
// Skip to main content
<a href="#main-content" className="skip-link">
  Skip to main content
</a>

<main id="main-content">
  {/* content */}
</main>

// Announce route changes
useEffect(() => {
  const announcement = document.createElement('div');
  announcement.setAttribute('role', 'status');
  announcement.setAttribute('aria-live', 'polite');
  announcement.textContent = `Navigated to ${pageTitle}`;
  document.body.appendChild(announcement);
  
  setTimeout(() => announcement.remove(), 1000);
}, [location]);
```

### Best Practices

- Use semantic HTML elements
- Provide text alternatives for images
- Ensure keyboard navigation works
- Maintain sufficient color contrast
- Support screen readers with ARIA
- Test with accessibility tools (axe, Lighthouse)
- Don't rely on color alone to convey information
- Provide focus indicators
- Make clickable areas large enough (min 44x44px)
- Test with actual screen readers

---

## State Management

Manage application state effectively.

### Local State

```javascript
// useState for component state
const [count, setCount] = useState(0);
const [user, setUser] = useState(null);

// useReducer for complex state logic
const [state, dispatch] = useReducer(reducer, initialState);

function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 };
    case 'decrement':
      return { count: state.count - 1 };
    default:
      return state;
  }
}
```

### Context API

```javascript
// Theme context
const ThemeContext = createContext();

export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  
  const value = {
    theme,
    toggleTheme: () => setTheme(t => t === 'light' ? 'dark' : 'light')
  };
  
  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
}
```

### Global State (Redux/Zustand)

```javascript
// Zustand store
import create from 'zustand';

const useStore = create((set) => ({
  user: null,
  isAuthenticated: false,
  login: (userData) => set({ user: userData, isAuthenticated: true }),
  logout: () => set({ user: null, isAuthenticated: false })
}));

// Usage
function Profile() {
  const user = useStore(state => state.user);
  const logout = useStore(state => state.logout);
  
  return <div>{user?.name}</div>;
}
```

### Server State (React Query)

```javascript
// Fetching data with React Query
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

function Users() {
  const { data, isLoading, error } = useQuery({
    queryKey: ['users'],
    queryFn: fetchUsers
  });
  
  const queryClient = useQueryClient();
  
  const createUser = useMutation({
    mutationFn: createUserAPI,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    }
  });
  
  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  
  return <div>{/* render users */}</div>;
}
```

### Best Practices

- Keep state as local as possible
- Lift state only when necessary
- Use Context for theme, auth, i18n
- Use specialized libraries for server state
- Avoid prop drilling (use composition or context)
- Normalize nested/relational data
- Use selectors to derive state
- Keep state immutable

---

## Routing

Implement client-side routing effectively.

### React Router Example

```javascript
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <nav>
        <Link to="/">Home</Link>
        <Link to="/about">About</Link>
        <Link to="/users">Users</Link>
      </nav>
      
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/users" element={<Users />} />
        <Route path="/users/:id" element={<UserDetail />} />
        <Route path="*" element={<NotFound />} />
      </Routes>
    </BrowserRouter>
  );
}
```

### Nested Routes

```javascript
<Route path="/dashboard" element={<Dashboard />}>
  <Route path="overview" element={<Overview />} />
  <Route path="analytics" element={<Analytics />} />
  <Route path="settings" element={<Settings />} />
</Route>

// Dashboard component
function Dashboard() {
  return (
    <div>
      <Sidebar />
      <Outlet /> {/* Renders nested routes */}
    </div>
  );
}
```

### Protected Routes

```javascript
function ProtectedRoute({ children }) {
  const { isAuthenticated } = useAuth();
  
  if (!isAuthenticated) {
    return <Navigate to="/login" replace />;
  }
  
  return children;
}

// Usage
<Route
  path="/dashboard"
  element={
    <ProtectedRoute>
      <Dashboard />
    </ProtectedRoute>
  }
/>
```

### Route Parameters & Search Params

```javascript
// URL: /users/123?tab=profile
function UserDetail() {
  const { id } = useParams();
  const [searchParams] = useSearchParams();
  const tab = searchParams.get('tab');
  
  return <div>User {id}, Tab: {tab}</div>;
}
```

### Programmatic Navigation

```javascript
function LoginForm() {
  const navigate = useNavigate();
  
  const handleSubmit = async (data) => {
    await login(data);
    navigate('/dashboard', { replace: true });
  };
  
  return <form onSubmit={handleSubmit}>{/* form */}</form>;
}
```

### Best Practices

- Use code splitting for routes
- Implement loading states
- Handle 404 pages
- Use meaningful URLs
- Implement breadcrumbs for deep navigation
- Preserve scroll position when appropriate
- Use `replace` for redirects after actions
- Implement route-based analytics

---

## Forms & Validation

Build robust forms with proper validation.

### Controlled Components

```javascript
function LoginForm() {
  const [formData, setFormData] = useState({
    email: '',
    password: ''
  });
  
  const handleChange = (e) => {
    const { name, value } = e.target;
    setFormData(prev => ({ ...prev, [name]: value }));
  };
  
  const handleSubmit = (e) => {
    e.preventDefault();
    // Handle submission
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        name="email"
        type="email"
        value={formData.email}
        onChange={handleChange}
        required
      />
      <input
        name="password"
        type="password"
        value={formData.password}
        onChange={handleChange}
        required
      />
      <button type="submit">Login</button>
    </form>
  );
}
```

### Form Libraries (React Hook Form)

```javascript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import * as z from 'zod';

const schema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
  age: z.number().min(18, 'Must be at least 18 years old')
});

function RegisterForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting }
  } = useForm({
    resolver: zodResolver(schema)
  });
  
  const onSubmit = async (data) => {
    await createUser(data);
  };
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <input
          {...register('email')}
          type="email"
          placeholder="Email"
        />
        {errors.email && <span>{errors.email.message}</span>}
      </div>
      
      <div>
        <input
          {...register('password')}
          type="password"
          placeholder="Password"
        />
        {errors.password && <span>{errors.password.message}</span>}
      </div>
      
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Register'}
      </button>
    </form>
  );
}
```

### Custom Validation

```javascript
const validateEmail = (value) => {
  if (!value) return 'Email is required';
  if (!/\S+@\S+\.\S+/.test(value)) return 'Invalid email format';
  return true;
};

const validatePassword = (value) => {
  if (!value) return 'Password is required';
  if (value.length < 8) return 'Password must be at least 8 characters';
  if (!/[A-Z]/.test(value)) return 'Must contain uppercase letter';
  if (!/[a-z]/.test(value)) return 'Must contain lowercase letter';
  if (!/[0-9]/.test(value)) return 'Must contain number';
  return true;
};
```

### File Upload

```javascript
function FileUpload() {
  const [file, setFile] = useState(null);
  const [preview, setPreview] = useState(null);
  
  const handleFileChange = (e) => {
    const selectedFile = e.target.files[0];
    
    if (selectedFile) {
      // Validate file
      if (selectedFile.size > 5 * 1024 * 1024) {
        alert('File too large (max 5MB)');
        return;
      }
      
      if (!['image/jpeg', 'image/png'].includes(selectedFile.type)) {
        alert('Invalid file type');
        return;
      }
      
      setFile(selectedFile);
      setPreview(URL.createObjectURL(selectedFile));
    }
  };
  
  const handleUpload = async () => {
    const formData = new FormData();
    formData.append('file', file);
    
    await fetch('/api/upload', {
      method: 'POST',
      body: formData
    });
  };
  
  return (
    <div>
      <input
        type="file"
        accept="image/jpeg,image/png"
        onChange={handleFileChange}
      />
      {preview && <img src={preview} alt="Preview" />}
      <button onClick={handleUpload} disabled={!file}>
        Upload
      </button>
    </div>
  );
}
```

### Best Practices

- Provide immediate feedback on validation
- Show clear error messages
- Disable submit during processing
- Preserve form data on errors
- Use proper input types (email, tel, number)
- Implement client-side and server-side validation
- Provide helpful placeholder text
- Use autocomplete attributes
- Handle loading and error states
- Consider accessibility in form design

---

## API Integration

Connect frontend to backend services.

### Fetch API

```javascript
async function fetchUsers() {
  try {
    const response = await fetch('/api/users', {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      }
    });
    
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    
    const data = await response.json();
    return data;
  } catch (error) {
    console.error('Failed to fetch users:', error);
    throw error;
  }
}
```

### API Service Layer

```javascript
// services/api.js
const API_BASE_URL = process.env.REACT_APP_API_URL;

class ApiService {
  constructor() {
    this.baseURL = API_BASE_URL;
  }
  
  async request(endpoint, options = {}) {
    const url = `${this.baseURL}${endpoint}`;
    const token = localStorage.getItem('token');
    
    const config = {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...(token && { 'Authorization': `Bearer ${token}` }),
        ...options.headers
      }
    };
    
    try {
      const response = await fetch(url, config);
      
      if (!response.ok) {
        const error = await response.json();
        throw new ApiError(error.message, response.status, error);
      }
      
      return await response.json();
    } catch (error) {
      if (error instanceof ApiError) throw error;
      throw new ApiError('Network error', 0, error);
    }
  }
  
  get(endpoint, options) {
    return this.request(endpoint, { ...options, method: 'GET' });
  }
  
  post(endpoint, data, options) {
    return this.request(endpoint, {
      ...options,
      method: 'POST',
      body: JSON.stringify(data)
    });
  }
  
  put(endpoint, data, options) {
    return this.request(endpoint, {
      ...options,
      method: 'PUT',
      body: JSON.stringify(data)
    });
  }
  
  delete(endpoint, options) {
    return this.request(endpoint, { ...options, method: 'DELETE' });
  }
}

class ApiError extends Error {
  constructor(message, status, details) {
    super(message);
    this.status = status;
    this.details = details;
  }
}

export const api = new ApiService();

// services/users.js
export const userService = {
  getUsers: () => api.get('/users'),
  getUser: (id) => api.get(`/users/${id}`),
  createUser: (data) => api.post('/users', data),
  updateUser: (id, data) => api.put(`/users/${id}`, data),
  deleteUser: (id) => api.delete(`/users/${id}`)
};
```

### Request Interceptors

```javascript
// Add request interceptor for auth token
const originalFetch = window.fetch;
window.fetch = async (...args) => {
  const [url, config = {}] = args;
  
  // Add auth token
  const token = localStorage.getItem('token');
  if (token) {
    config.headers = {
      ...config.headers,
      'Authorization': `Bearer ${token}`
    };
  }
  
  // Add request ID
  config.headers = {
    ...config.headers,
    'X-Request-ID': generateRequestId()
  };
  
  const response = await originalFetch(url, config);
  
  // Handle 401 (unauthorized)
  if (response.status === 401) {
    // Redirect to login
    window.location.href = '/login';
  }
  
  return response;
};
```

### Optimistic Updates

```javascript
function TodoList() {
  const [todos, setTodos] = useState([]);
  
  const toggleTodo = async (id) => {
    // Optimistic update
    setTodos(prev =>
      prev.map(todo =>
        todo.id === id ? { ...todo, completed: !todo.completed } : todo
      )
    );
    
    try {
      await api.put(`/todos/${id}/toggle`);
    } catch (error) {
      // Revert on error
      setTodos(prev =>
        prev.map(todo =>
          todo.id === id ? { ...todo, completed: !todo.completed } : todo
        )
      );
      showError('Failed to update todo');
    }
  };
  
  return <div>{/* render todos */}</div>;
}
```

### Best Practices

- Centralize API calls in service layer
- Handle errors consistently
- Implement request/response interceptors
- Add loading states
- Implement retry logic for failed requests
- Use environment variables for API URLs
- Add request timeouts
- Cache responses when appropriate
- Implement optimistic updates for better UX
- Handle offline scenarios

---

## Error Handling

Gracefully handle errors for better user experience.

### Error Boundaries (React)

```javascript
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }
  
  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }
  
  componentDidCatch(error, errorInfo) {
    // Log to error reporting service
    console.error('Error caught:', error, errorInfo);
    logErrorToService(error, errorInfo);
  }
  
  render() {
    if (this.state.hasError) {
      return (
        <div className="error-fallback">
          <h1>Something went wrong</h1>
          <p>{this.state.error?.message}</p>
          <button onClick={() => this.setState({ hasError: false })}>
            Try again
          </button>
        </div>
      );
    }
    
    return this.props.children;
  }
}

// Usage
<ErrorBoundary>
  <App />
</ErrorBoundary>
```

### Try-Catch Pattern

```javascript
async function loadUserData() {
  try {
    setLoading(true);
    setError(null);
    
    const user = await fetchUser();
    const posts = await fetchUserPosts(user.id);
    
    setUser(user);
    setPosts(posts);
  } catch (error) {
    if (error.status === 404) {
      setError('User not found');
    } else if (error.status >= 500) {
      setError('Server error. Please try again later.');
    } else {
      setError('An unexpected error occurred');
    }
    
    // Log error
    logError(error);
  } finally {
    setLoading(false);
  }
}
```

### Toast Notifications

```javascript
import { toast } from 'react-hot-toast';

// Success
toast.success('Settings saved successfully!');

// Error
toast.error('Failed to save settings');

// Custom
toast.custom((t) => (
  <div className={`toast ${t.visible ? 'animate-enter' : 'animate-leave'}`}>
    <p>Custom notification</p>
    <button onClick={() => toast.dismiss(t.id)}>Dismiss</button>
  </div>
));
```

### Global Error Handler

```javascript
// Catch unhandled promise rejections
window.addEventListener('unhandledrejection', (event) => {
  console.error('Unhandled promise rejection:', event.reason);
  logErrorToService(event.reason);
  
  toast.error('An unexpected error occurred');
  event.preventDefault();
});

// Catch global errors
window.addEventListener('error', (event) => {
  console.error('Global error:', event.error);
  logErrorToService(event.error);
});
```

### Best Practices

- Use error boundaries for component errors
- Provide user-friendly error messages
- Log errors to monitoring service
- Show appropriate fallback UI
- Handle network errors gracefully
- Implement retry mechanisms
- Don't expose sensitive error details to users
- Provide recovery options
- Handle different error types appropriately

---

## Testing

Ensure code quality with comprehensive testing.

### Unit Tests (Jest + React Testing Library)

```javascript
import { render, screen, fireEvent } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Counter } from './Counter';

describe('Counter', () => {
  it('renders initial count', () => {
    render(<Counter initialCount={0} />);
    expect(screen.getByText('Count: 0')).toBeInTheDocument();
  });
  
  it('increments count when button clicked', async () => {
    const user = userEvent.setup();
    render(<Counter initialCount={0} />);
    
    const button = screen.getByRole('button', { name: /increment/i });
    await user.click(button);
    
    expect(screen.getByText('Count: 1')).toBeInTheDocument();
  });
  
  it('calls onCountChange with new value', async () => {
    const user = userEvent.setup();
    const handleCountChange = jest.fn();
    render(<Counter initialCount={0} onCountChange={handleCountChange} />);
    
    const button = screen.getByRole('button', { name: /increment/i });
    await user.click(button);
    
    expect(handleCountChange).toHaveBeenCalledWith(1);
  });
});
```

### Integration Tests

```javascript
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { UserProfile } from './UserProfile';
import { server } from './mocks/server';
import { rest } from 'msw';

describe('UserProfile Integration', () => {
  it('loads and displays user data', async () => {
    render(<UserProfile userId="123" />);
    
    expect(screen.getByText(/loading/i)).toBeInTheDocument();
    
    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument();
    });
  });
  
  it('handles server error', async () => {
    server.use(
      rest.get('/api/users/:id', (req, res, ctx) => {
        return res(ctx.status(500));
      })
    );
    
    render(<UserProfile userId="123" />);
    
    await waitFor(() => {
      expect(screen.getByText(/error/i)).toBeInTheDocument();
    });
  });
});
```

### E2E Tests (Playwright/Cypress)

```javascript
// Playwright
import { test, expect } from '@playwright/test';

test('user can login and view dashboard', async ({ page }) => {
  await page.goto('/login');
  
  await page.fill('[name="email"]', 'user@example.com');
  await page.fill('[name="password"]', 'password123');
  await page.click('button[type="submit"]');
  
  await expect(page).toHaveURL('/dashboard');
  await expect(page.locator('h1')).toContainText('Dashboard');
});

// Cypress
describe('Login Flow', () => {
  it('allows user to login', () => {
    cy.visit('/login');
    cy.get('[name="email"]').type('user@example.com');
    cy.get('[name="password"]').type('password123');
    cy.get('button[type="submit"]').click();
    
    cy.url().should('include', '/dashboard');
    cy.contains('h1', 'Dashboard').should('be.visible');
  });
});
```

### Test Coverage

```json
{
  "jest": {
    "collectCoverageFrom": [
      "src/**/*.{js,jsx,ts,tsx}",
      "!src/**/*.test.{js,jsx,ts,tsx}",
      "!src/index.tsx"
    ],
    "coverageThreshold": {
      "global": {
        "branches": 80,
        "functions": 80,
        "lines": 80,
        "statements": 80
      }
    }
  }
}
```

### Best Practices

- Write tests for critical user paths
- Test behavior, not implementation
- Use data-testid sparingly (prefer accessible queries)
- Mock external dependencies
- Test error states and edge cases
- Maintain test coverage above 80%
- Run tests in CI/CD pipeline
- Use visual regression testing for UI
- Keep tests fast and isolated

---

## Security

Protect your application and users.

### XSS Prevention

```javascript
// Good: React escapes by default
<div>{userInput}</div>

// Dangerous: Avoid dangerouslySetInnerHTML
<div dangerouslySetInnerHTML={{ __html: userInput }} /> // ❌

// If needed, sanitize first
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userInput) }} />
```

### CSRF Protection

```javascript
// Include CSRF token in requests
const csrfToken = document.querySelector('meta[name="csrf-token"]').content;

fetch('/api/data', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-CSRF-Token': csrfToken
  },
  body: JSON.stringify(data)
});
```

### Secure Storage

```javascript
// Never store sensitive data in localStorage
// ❌ Don't do this
localStorage.setItem('creditCard', cardNumber);

// ✅ Store in httpOnly cookies (set by server)
// ✅ Or use sessionStorage for temporary data
sessionStorage.setItem('tempData', data);

// Clear on logout
function logout() {
  localStorage.clear();
  sessionStorage.clear();
  // Redirect to login
}
```

### Content Security Policy

```html
<meta
  http-equiv="Content-Security-Policy"
  content="
    default-src 'self';
    script-src 'self' 'unsafe-inline' https://trusted-cdn.com;
    style-src 'self' 'unsafe-inline';
    img-src 'self' data: https:;
    font-src 'self';
    connect-src 'self' https://api.example.com;
  "
/>
```

### Input Validation

```javascript
function sanitizeInput(input) {
  // Remove HTML tags
  return input.replace(/<[^>]*>/g, '');
}

function validateEmail(email) {
  const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return regex.test(email);
}

function validateURL(url) {
  try {
    new URL(url);
    return true;
  } catch {
    return false;
  }
}
```

### Best Practices

- Sanitize all user input
- Use HTTPS everywhere
- Implement CSP headers
- Avoid inline scripts and styles
- Use httpOnly and secure flags for cookies
- Implement proper CORS policies
- Keep dependencies updated
- Use security headers (X-Frame-Options, etc.)
- Implement rate limiting on client side
- Never expose API keys in client code

---

## Build & Deployment

Optimize for production deployment.

### Environment Variables

```bash
# .env.development
REACT_APP_API_URL=http://localhost:3000/api
REACT_APP_ENV=development

# .env.production
REACT_APP_API_URL=https://api.production.com
REACT_APP_ENV=production
```

```javascript
const apiUrl = process.env.REACT_APP_API_URL;
const isDevelopment = process.env.NODE_ENV === 'development';
```

### Build Optimization

```javascript
// vite.config.js
export default {
  build: {
    minify: 'terser',
    sourcemap: false,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          router: ['react-router-dom']
        }
      }
    }
  }
};
```

### Progressive Web App

```javascript
// service-worker.js
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open('v1').then((cache) => {
      return cache.addAll([
        '/',
        '/index.html',
        '/styles.css',
        '/app.js'
      ]);
    })
  );
});

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((response) => {
      return response || fetch(event.request);
    })
  );
});
```

### CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - run: npm ci
      - run: npm run lint
      - run: npm test
      - run: npm run build
      
      - name: Deploy to Production
        run: npm run deploy
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

### Best Practices

- Use environment variables for configuration
- Minify and compress assets
- Remove console.logs in production
- Enable source maps for debugging (stored separately)
- Implement caching strategies
- Use CDN for static assets
- Monitor bundle size
- Set up automated deployments
- Implement rollback mechanisms
- Use feature flags for gradual rollouts

---

## Developer Experience

Improve productivity and code quality.

### Code Formatting (Prettier)

```json
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 80
}
```

### Linting (ESLint)

```json
{
  "extends": [
    "eslint:recommended",
    "plugin:react/recommended",
    "plugin:@typescript-script/recommended"
  ],
  "rules": {
    "no-console": "warn",
    "no-unused-vars": "error",
    "react/prop-types": "off"
  }
}
```

### Git Hooks (Husky)

```json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged",
      "pre-push": "npm test"
    }
  },
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ]
  }
}
```

### TypeScript

```typescript
// Properly typed components
interface ButtonProps {
  variant?: 'primary' | 'secondary';
  size?: 'small' | 'medium' | 'large';
  onClick?: () => void;
  children: React.ReactNode;
}

const Button: React.FC<ButtonProps> = ({
  variant = 'primary',
  size = 'medium',
  onClick,
  children
}) => {
  return (
    <button className={`btn btn-${variant} btn-${size}`} onClick={onClick}>
      {children}
    </button>
  );
};
```

### Component Documentation (Storybook)

```javascript
// Button.stories.tsx
export default {
  title: 'Components/Button',
  component: Button,
  argTypes: {
    variant: {
      control: 'select',
      options: ['primary', 'secondary']
    }
  }
};

export const Primary = {
  args: {
    variant: 'primary',
    children: 'Click me'
  }
};

export const Secondary = {
  args: {
    variant: 'secondary',
    children: 'Click me'
  }
};
```

### Best Practices

- Use TypeScript for type safety
- Set up automated formatting and linting
- Document components with Storybook
- Use git hooks to enforce quality
- Implement code review process
- Use consistent naming conventions
- Write meaningful commit messages
- Keep dependencies updated
- Use absolute imports for better organization
- Implement hot module replacement for faster development

---

## Summary

Following these frontend best practices will help you build modern, performant, and maintainable applications:

1. **Organize code** with clear structure and separation of concerns
2. **Optimize performance** with code splitting, lazy loading, and caching
3. **Build accessible** applications that work for everyone
4. **Manage state** effectively with appropriate tools
5. **Implement routing** for seamless navigation
6. **Validate forms** with user-friendly error handling
7. **Integrate APIs** through a clean service layer
8. **Handle errors** gracefully with proper fallbacks
9. **Test thoroughly** with unit, integration, and E2E tests
10. **Secure your app** against common vulnerabilities
11. **Deploy efficiently** with optimized builds and CI/CD
12. **Enhance DX** with tooling and automation

These standards create a foundation for frontend applications that are fast, reliable, accessible, and maintainable.
