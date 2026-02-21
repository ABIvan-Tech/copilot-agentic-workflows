# Frontend Architecture & Performance

## Overview
Best practices for building scalable, performant frontend applications with modern frameworks, focusing on architecture patterns, optimization techniques, and user experience.

## When to Use This Skill
- Architecting new frontend applications
- Optimizing performance and bundle size  
- Implementing state management
- Building component libraries
- Improving Core Web Vitals
- Debugging performance issues
- Setting up build optimization

## Component Architecture

### Component Design Principles

**1. Single Responsibility**
```jsx
// ❌ Bad - Component does too much
function UserDashboard() {
  const [user, setUser] = useState(null);
  const [orders, setOrders] = useState([]);
  const [loading, setLoading] = useState(false);
  
  useEffect(() => {
    // Fetch user
    // Fetch orders  
    // Handle loading states
  }, []);
  
  return (
    <div>
      {/* User profile rendering */}
      {/* Orders table rendering */}
      {/* Analytics charts */}
    </div>
  );
}

// ✅ Good - Separated concerns
function UserDashboard() {
  return (
    <div>
      <UserProfile />
      <UserOrders />
      <UserAnalytics />
    </div>
  );
}
```

**2. Composition Over Inheritance**
```jsx
// ✅ Good - Composition pattern
function Card({ children, className }) {
  return <div className={`card ${className}`}>{children}</div>;
}

function UserCard({ user }) {
  return (
    <Card>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </Card>
  );
}

// Use composition
<Card>
  <UserProfile user={user} />
</Card>
```

**3. Props vs Children**
```jsx
// Use children for flexible content
function Modal({ children, onClose }) {
  return (
    <div className="modal">
      <button onClick={onClose}>×</button>
      {children}
    </div>
  );
}

// Use props for specific configuration
function DataTable({ columns, data, onRowClick }) {
  return (
    <table>
      <thead>
        <tr>
          {columns.map(col => <th key={col.key}>{col.label}</th>)}
        </tr>
      </thead>
      <tbody>
        {data.map(row => (
          <tr key={row.id} onClick={() => onRowClick(row)}>
            {columns.map(col => <td key={col.key}>{row[col.key]}</td>)}
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

## State Management

### Choose the Right Tool

**Component State (useState)**
```jsx
// ✅ Good for - Local UI state
function Dropdown() {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <div>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      {isOpen && <Menu />}
    </div>
  );
}
```

**Context API**
```jsx
// ✅ Good for - Shared state across component tree
const ThemeContext = createContext();

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) throw new Error('useTheme must be within ThemeProvider');
  return context;
}
```

**State Management Libraries (Zustand/Redux)**
```javascript
// ✅ Good for - Complex global state, multiple actions
import create from 'zustand';

const useStore = create((set) => ({
  user: null,
  cart: [],
  
  setUser: (user) => set({ user }),
  
  addToCart: (item) => set((state) => ({
    cart: [...state.cart, item]
  })),
  
  removeFromCart: (itemId) => set((state) => ({
    cart: state.cart.filter(item => item.id !== itemId)
  })),
  
  clearCart: () => set({ cart: [] })
}));

// Usage
function Cart() {
  const { cart, removeFromCart } = useStore();
  
  return (
    <div>
      {cart.map(item => (
        <div key={item.id}>
          {item.name}
          <button onClick={() => removeFromCart(item.id)}>Remove</button>
        </div>
      ))}
    </div>
  );
}
```

### Avoid Prop Drilling
```jsx
// ❌ Bad - Prop drilling through multiple levels
function App() {
  const [user, setUser] = useState(null);
  return <Header user={user} setUser={setUser} />;
}

function Header({ user, setUser }) {
  return <Navigation user={user} setUser={setUser} />;
}

function Navigation({ user, setUser }) {
  return <UserMenu user={user} setUser={setUser} />;
}

// ✅ Good - Use Context or state management
const UserContext = createContext();

function App() {
  const [user, setUser] = useState(null);
  return (
    <UserContext.Provider value={{ user, setUser }}>
      <Header />
    </UserContext.Provider>
  );
}

function UserMenu() {
  const { user, setUser } = useContext(UserContext);
  // Direct access, no prop drilling
}
```

## Performance Optimization

### 1. Code Splitting & Lazy Loading
```jsx
import { lazy, Suspense } from 'react';

// ✅ Good - Lazy load routes
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Profile = lazy(() => import('./pages/Profile'));
const Settings = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <Router>
      <Suspense fallback={<LoadingSpinner />}>
        <Routes>
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/profile" element={<Profile />} />
          <Route path="/settings" element={<Settings />} />
        </Routes>
      </Suspense>
    </Router>
  );
}

// ✅ Good - Lazy load heavy components
const HeavyChart = lazy(() => import('./components/HeavyChart'));

function Analytics() {
  return (
    <div>
      <h1>Analytics</h1>
      <Suspense fallback={<div>Loading chart...</div>}>
        <HeavyChart data={data} />
      </Suspense>
    </div>
  );
}
```

### 2. Memoization
```jsx
import { memo, useMemo, useCallback } from 'react';

// ✅ Memo component - Prevents re-renders if props unchanged
const ExpensiveComponent = memo(function ExpensiveComponent({ data }) {
  // Complex rendering logic
  return <div>...</div>;
});

// ✅ useMemo - Cache computed values
function ProductList({ products, filter }) {
  const filteredProducts = useMemo(() => {
    return products.filter(p => p.category === filter);
  }, [products, filter]); // Only recompute when dependencies change
  
  return filteredProducts.map(p => <ProductCard key={p.id} product={p} />);
}

// ✅ useCallback - Cache function references
function TodoList({ todos }) {
  const [filter, setFilter] = useState('all');
  
  // Without useCallback, new function created on every render
  const handleToggle = useCallback((id) => {
    // Toggle todo logic
  }, []); // Empty deps = function never changes
  
  return todos.map(todo => (
    <TodoItem key={todo.id} todo={todo} onToggle={handleToggle} />
  ));
}
```

### 3. Virtual Scrolling (Large Lists)
```jsx
import { FixedSizeList } from 'react-window';

// ✅ Good - Virtual scrolling for large lists
function LargeList({ items }) {
  const Row = ({ index, style }) => (
    <div style={style}>
      {items[index].name}
    </div>
  );
  
  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={50}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}

// ❌ Bad - Rendering 10,000 items
function LargeList({ items }) {
  return items.map(item => <div key={item.id}>{item.name}</div>);
}
```

### 4. Image Optimization
```jsx
// ✅ Good - Lazy load images with native loading
<img 
  src="/images/large-photo.jpg"
  alt="Description"
  loading="lazy"
  width={800}
  height={600}
/>

// ✅ Better - Use Next.js Image component
import Image from 'next/image';

<Image
  src="/images/large-photo.jpg"
  alt="Description"
  width={800}
  height={600}
  placeholder="blur"
  blurDataURL="/images/large-photo-blur.jpg"
/>

// ✅ Responsive images with srcset
<img
  src="/images/photo-800.jpg"
  srcSet="
    /images/photo-400.jpg 400w,
    /images/photo-800.jpg 800w,
    /images/photo-1200.jpg 1200w
  "
  sizes="(max-width: 600px) 400px, (max-width: 1200px) 800px, 1200px"
  alt="Description"
/>
```

### 5. Bundle Size Optimization
```javascript
// vite.config.js / webpack config
export default {
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor': ['react', 'react-dom'],
          'ui': ['@radix-ui/react-dialog', '@radix-ui/react-dropdown-menu'],
          'charts': ['recharts']
        }
      }
    }
  }
};

// ✅ Tree-shaking friendly imports
import { debounce } from 'lodash-es';

// ❌ Imports entire library
import _ from 'lodash';
```

## Data Fetching Patterns

### Server State vs Client State
```jsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// ✅ Good - React Query for server state
function UserProfile({ userId }) {
  const { data: user, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    staleTime: 5 * 60 * 1000, // Consider data fresh for 5 minutes
  });
  
  if (isLoading) return <Spinner />;
  if (error) return <Error message={error.message} />;
  
  return <div>{user.name}</div>;
}

// ✅ Mutations with cache invalidation
function UpdateProfileForm({ userId }) {
  const queryClient = useQueryClient();
  
  const mutation = useMutation({
    mutationFn: (data) => updateUser(userId, data),
    onSuccess: () => {
      // Invalidate and refetch user query
      queryClient.invalidateQueries({ queryKey: ['user', userId] });
    }
  });
  
  const handleSubmit = (data) => {
    mutation.mutate(data);
  };
  
  return <form onSubmit={handleSubmit}>...</form>;
}
```

### Suspense for Data Fetching (Experimental)
```jsx
import { Suspense } from 'react';

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <UserProfile />
    </Suspense>
  );
}

// Using React Query with Suspense
function UserProfile() {
  const { data } = useQuery({
    queryKey: ['user'],
    queryFn: fetchUser,
    suspense: true // Enable Suspense mode
  });
  
  return <div>{data.name}</div>;
}
```

## Routing & Navigation

### Code-Split Routes
```jsx
import { lazy } from 'react';
import { createBrowserRouter } from 'react-router-dom';

const router = createBrowserRouter([
  {
    path: '/',
    element: <Layout />,
    children: [
      {
        index: true,
        element: lazy(() => import('./pages/Home'))
      },
      {
        path: 'products',
        element: lazy(() => import('./pages/Products')),
        loader: async () => {
          // Preload data before showing page
          return fetch('/api/products').then(r => r.json());
        }
      },
      {
        path: 'products/:id',
        element: lazy(() => import('./pages/ProductDetail')),
        loader: async ({ params }) => {
          return fetch(`/api/products/${params.id}`).then(r => r.json());
        }
      }
    ]
  }
]);
```

### Prefetching
```jsx
import { Link } from 'react-router-dom';

// ✅ Prefetch on hover
<Link 
  to="/products" 
  onMouseEnter={() => {
    import('./pages/Products'); // Prefetch module
    queryClient.prefetchQuery(['products'], fetchProducts); // Prefetch data
  }}
>
  Products
</Link>
```

## Core Web Vitals Optimization

### Largest Contentful Paint (LCP)
```jsx
// ✅ Optimize LCP
// 1. Preload critical resources
<link rel="preload" as="image" href="/hero.jpg" />

// 2. Use priority hints
<img src="/hero.jpg" fetchpriority="high" />

// 3. Avoid render-blocking JavaScript
<script src="/analytics.js" defer />

// 4. Server-side render initial content
// Use Next.js, Remix, or similar framework
```

### First Input Delay (FID) / Interaction to Next Paint (INP)
```jsx
// ✅ Debounce expensive operations
import { debounce } from 'lodash-es';

function SearchInput() {
  const [query, setQuery] = useState('');
  
  const debouncedSearch = useMemo(
    () => debounce((value) => {
      // Expensive search operation
      performSearch(value);
    }, 300),
    []
  );
  
  const handleChange = (e) => {
    setQuery(e.target.value);
    debouncedSearch(e.target.value);
  };
  
  return <input value={query} onChange={handleChange} />;
}

// ✅ Break up long tasks
async function processLargeDataset(data) {
  const chunks = chunkArray(data, 100);
  
  for (const chunk of chunks) {
    await processChunk(chunk);
    // Yield to browser
    await new Promise(resolve => setTimeout(resolve, 0));
  }
}
```

### Cumulative Layout Shift (CLS)
```jsx
// ✅ Reserve space for dynamic content
<div style={{ minHeight: '200px' }}>
  {loading ? <Skeleton /> : <Content />}
</div>

// ✅ Specify image dimensions
<img src="/photo.jpg" width={800} height={600} alt="Photo" />

// ✅ Use CSS aspect ratio
<div style={{ aspectRatio: '16/9' }}>
  <img src="/photo.jpg" alt="Photo" />
</div>

// ❌ Bad - No dimensions, causes layout shift
<img src="/photo.jpg" alt="Photo" />
```

## Error Boundaries
```jsx
import { Component } from 'react';

class ErrorBoundary extends Component {
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
        <div>
          <h1>Something went wrong</h1>
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
function App() {
  return (
    <ErrorBoundary>
      <Router>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/dashboard" element={
            <ErrorBoundary>
              <Dashboard />
            </ErrorBoundary>
          } />
        </Routes>
      </Router>
    </ErrorBoundary>
  );
}
```

## Build Optimization

### Vite Configuration
```javascript
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    react(),
    visualizer({ open: true }) // Analyze bundle
  ],
  build: {
    rollupOptions: {
      output: {
        manualChunks(id) {
          if (id.includes('node_modules')) {
            if (id.includes('react') || id.includes('react-dom')) {
              return 'vendor-react';
            }
            return 'vendor';
          }
        }
      }
    },
    chunkSizeWarningLimit: 1000,
    sourcemap: false // Disable in production
  }
});
```

### Environment Variables
```javascript
// .env
VITE_API_URL=https://api.example.com
VITE_ANALYTICS_ID=GA-123456

// Usage
const apiUrl = import.meta.env.VITE_API_URL;
```

## Testing Frontend Applications

### Component Testing
```jsx
import { render, screen, fireEvent } from '@testing-library/react';
import { Counter } from './Counter';

test('increments counter', () => {
  render(<Counter />);
  
  const button = screen.getByRole('button', { name: /increment/i });
  const count = screen.getByText(/count:/i);
  
  expect(count).toHaveTextContent('Count: 0');
  
  fireEvent.click(button);
  expect(count).toHaveTextContent('Count: 1');
});
```

### Integration Testing
```jsx
import { render, screen, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { UserProfile } from './UserProfile';

test('displays user profile', async () => {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } }
  });
  
  render(
    <QueryClientProvider client={queryClient}>
      <UserProfile userId={123} />
    </QueryClientProvider>
  );
  
  expect(screen.getByText(/loading/i)).toBeInTheDocument();
  
  await waitFor(() => {
    expect(screen.getByText(/john doe/i)).toBeInTheDocument();
  });
});
```

## Accessibility

### Semantic HTML
```jsx
// ✅ Good - Semantic HTML
<nav>
  <ul>
    <li><a href="/">Home</a></li>
    <li><a href="/about">About</a></li>
  </ul>
</nav>

<main>
  <article>
    <h1>Article Title</h1>
    <p>Content...</p>
  </article>
</main>

// ❌ Bad - Divs for everything
<div className="nav">
  <div className="nav-item">Home</div>
  <div className="nav-item">About</div>
</div>
```

### ARIA Attributes
```jsx
// ✅ Use ARIA when semantic HTML isn't enough
<button 
  aria-label="Close dialog"
  aria-expanded={isOpen}
  onClick={handleClose}
>
  ×
</button>

<div 
  role="alert" 
  aria-live="assertive"
>
  {errorMessage}
</div>

// Keyboard navigation
function Tabs() {
  const handleKeyDown = (e) => {
    if (e.key === 'ArrowRight') {
      // Focus next tab
    } else if (e.key === 'ArrowLeft') {
      // Focus previous tab
    }
  };
  
  return (
    <div role="tablist" onKeyDown={handleKeyDown}>
      <button role="tab" aria-selected={true}>Tab 1</button>
      <button role="tab" aria-selected={false}>Tab 2</button>
    </div>
  );
}
```

## Frontend Architecture Checklist

- [ ] Component structure follows single responsibility
- [ ] State management appropriate for scale
- [ ] Code splitting and lazy loading implemented
- [ ] Images optimized and lazy loaded
- [ ] Bundle size analyzed and optimized
- [ ] Memoization used for expensive calculations
- [ ] Virtual scrolling for large lists
- [ ] Error boundaries in place
- [ ] Loading and error states handled
- [ ] Accessibility (WCAG compliance)
- [ ] Core Web Vitals optimized (LCP, FID/INP, CLS)
- [ ] Server state managed with React Query or similar
- [ ] API calls properly cached
- [ ] Build configuration optimized
- [ ] Environment variables secured
- [ ] Responsive design (mobile-first)
- [ ] SEO meta tags configured
- [ ] Analytics and monitoring setup
- [ ] E2E tests for critical paths
- [ ] Performance monitoring (Lighthouse)
