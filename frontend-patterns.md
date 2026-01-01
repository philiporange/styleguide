# Frontend Patterns

Simple static HTML/JS/CSS frontends. No build tools, no npm, no frameworks.

## Design Philosophy

**Modernism and minimalism are paramount.** Interfaces should be clean, focused, and uncluttered.

- **Negative space**: Use generous padding and margins. Let elements breathe. White space is not wasted space—it creates visual hierarchy and reduces cognitive load.
- **Minimal UI elements**: Only show what's necessary. Remove decorative elements that don't serve a function.
- **Restrained color palette**: Use color sparingly and purposefully. Rely on the slate palette for most UI, with accent colors only for actions and status.
- **Typography over decoration**: Let clean typography and spacing do the work instead of borders, shadows, and ornaments.
- **Flat design**: Avoid gradients, heavy shadows, and 3D effects. Use subtle shadows only where depth aids understanding.

## Responsive Design

All interfaces must be mobile-first and fully responsive. Use a **hamburger menu with slide-out navigation** for mobile viewports.

### Hamburger Menu Pattern

```html
<!-- Header with hamburger -->
<header class="fixed top-0 left-0 right-0 h-14 bg-white border-b border-slate-200 z-50 flex items-center px-4">
    <button id="menuToggle" class="p-2 -ml-2 text-slate-600 hover:bg-slate-100 rounded-lg lg:hidden">
        <i class="fas fa-bars text-xl"></i>
    </button>
    <h1 class="ml-2 font-semibold text-slate-900 lg:ml-0">App Name</h1>
</header>

<!-- Overlay -->
<div id="menuOverlay" class="menu-overlay"></div>

<!-- Slide-out menu -->
<nav id="menuDrawer" class="menu-drawer">
    <div class="flex flex-col h-full">
        <div class="flex items-center justify-between mb-6">
            <span class="font-semibold text-slate-900">Menu</span>
            <button id="menuClose" class="p-2 text-slate-400 hover:text-slate-600">
                <i class="fas fa-times"></i>
            </button>
        </div>

        <nav class="flex-1 space-y-1">
            <a href="/" class="flex items-center gap-3 px-3 py-2 rounded-lg text-slate-700 hover:bg-slate-100">
                <i class="fas fa-home w-5"></i>
                <span>Home</span>
            </a>
            <a href="/settings" class="flex items-center gap-3 px-3 py-2 rounded-lg text-slate-700 hover:bg-slate-100">
                <i class="fas fa-cog w-5"></i>
                <span>Settings</span>
            </a>
        </nav>

        <button onclick="logout()" class="flex items-center gap-3 px-3 py-2 rounded-lg text-slate-700 hover:bg-slate-100 mt-auto">
            <i class="fas fa-sign-out-alt w-5"></i>
            <span>Logout</span>
        </button>
    </div>
</nav>

<!-- Main content with top padding for fixed header -->
<main class="pt-14 lg:pl-64">
    <!-- Page content -->
</main>
```

```javascript
// Hamburger menu toggle
const menuToggle = document.getElementById('menuToggle');
const menuClose = document.getElementById('menuClose');
const menuDrawer = document.getElementById('menuDrawer');
const menuOverlay = document.getElementById('menuOverlay');

function openMenu() {
    menuDrawer.classList.add('open');
    menuOverlay.classList.add('active');
    document.body.style.overflow = 'hidden';
}

function closeMenu() {
    menuDrawer.classList.remove('open');
    menuOverlay.classList.remove('active');
    document.body.style.overflow = '';
}

menuToggle?.addEventListener('click', openMenu);
menuClose?.addEventListener('click', closeMenu);
menuOverlay?.addEventListener('click', closeMenu);
```

### Responsive Breakpoints

Use Tailwind's standard breakpoints:

| Prefix | Min-width | Usage |
|--------|-----------|-------|
| (none) | 0px | Mobile-first base styles |
| `sm:` | 640px | Large phones |
| `md:` | 768px | Tablets |
| `lg:` | 1024px | Desktop—show persistent sidebar |
| `xl:` | 1280px | Large desktop |

### Desktop Sidebar

On large screens (`lg:` and up), show a persistent sidebar instead of the hamburger menu:

```html
<!-- Sidebar (hidden on mobile, visible on desktop) -->
<aside class="hidden lg:flex lg:flex-col lg:fixed lg:left-0 lg:top-0 lg:bottom-0 lg:w-64 bg-white border-r border-slate-200 p-4">
    <!-- Same nav content as mobile drawer -->
</aside>
```

```css
/* Hide hamburger on desktop */
@media (min-width: 1024px) {
    #menuToggle { display: none; }
    .menu-drawer { display: none; }
    .menu-overlay { display: none; }
}
```

## Stack

- **Tailwind CSS** via CDN (`tailwind.js` or `cdn.tailwindcss.com`)
- **Vanilla JavaScript** (no React, Vue, etc.)
- **Handlebars** for HTML templates (cleaner, reusable markup)
- **Font Awesome** for icons via CDN
- **Inter font** from Google Fonts
- **FastAPI** for serving static files

## Directory Structure

Split HTML, CSS, and JS into separate files. Use Handlebars templates for reusable components. Static files live inside the package directory.

```
project_name/
└── project_name/
    ├── server.py           # FastAPI server with static file serving
    ├── static/
    │   ├── css/
    │   │   └── style.css   # All custom styles
    │   ├── js/
    │   │   ├── common.js   # Shared utilities, API client
    │   │   ├── menu.js     # Hamburger menu logic
    │   │   ├── auth.js     # Auth-specific code
    │   │   └── dashboard.js
    │   ├── index.html      # Main page
    │   ├── login.html      # Login page
    │   └── admin.html      # Admin page
    └── templates/          # Handlebars templates (optional)
        ├── nav.hbs         # Navigation partial
        ├── modal.hbs       # Modal partial
        └── toast.hbs       # Toast notification partial
```

## FastAPI Static File Server

In `server.py`, mount the static directory and serve HTML pages. Since static/ is inside the package, use `Path(__file__).parent` to locate it:

```python
"""
FastAPI server with static file serving.

Serves the frontend from /static and provides API endpoints.
"""

from pathlib import Path

from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles
from fastapi.responses import FileResponse


app = FastAPI()

# Static files directory (sibling to this file)
STATIC_DIR = Path(__file__).parent / "static"


# Mount static files
app.mount("/static", StaticFiles(directory=STATIC_DIR), name="static")


# Serve HTML pages
@app.get("/")
async def index():
    return FileResponse(STATIC_DIR / "index.html")


@app.get("/login")
async def login():
    return FileResponse(STATIC_DIR / "login.html")


@app.get("/admin")
async def admin():
    return FileResponse(STATIC_DIR / "admin.html")


# API routes
@app.get("/api/items")
async def get_items():
    # ... API logic
    pass


def main():
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)


if __name__ == "__main__":
    main()
```

## Handlebars Templates

Use Handlebars for reusable HTML components. Load templates client-side and compile them.

### Base HTML Template (templates/base.html)

```html
<!DOCTYPE html>
<html lang="en" class="h-full bg-slate-50">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{title}} - Project Name</title>

    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>

    <!-- Font Awesome -->
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.2/css/all.min.css" rel="stylesheet">

    <!-- Inter Font -->
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap" rel="stylesheet">

    <!-- Custom styles -->
    <link href="/static/css/style.css" rel="stylesheet">

    <!-- Handlebars -->
    <script src="https://cdn.jsdelivr.net/npm/handlebars@4.7.8/dist/handlebars.min.js"></script>

    <style>body { font-family: 'Inter', sans-serif; }</style>
</head>
<body class="h-full">
    {{> content}}

    <!-- Scripts -->
    <script src="/static/js/common.js"></script>
    {{#each scripts}}
    <script src="/static/js/{{this}}"></script>
    {{/each}}
</body>
</html>
```

### Navigation Partial (templates/nav.hbs)

```handlebars
<script id="nav-template" type="text/x-handlebars-template">
<!-- Header with hamburger -->
<header class="fixed top-0 left-0 right-0 h-14 bg-white border-b border-slate-200 z-50 flex items-center px-4">
    <button id="menuToggle" class="p-2 -ml-2 text-slate-600 hover:bg-slate-100 rounded-lg lg:hidden">
        <i class="fas fa-bars text-xl"></i>
    </button>
    <h1 class="ml-2 font-semibold text-slate-900 lg:ml-0">{{appName}}</h1>
</header>

<!-- Overlay -->
<div id="menuOverlay" class="menu-overlay"></div>

<!-- Slide-out menu -->
<nav id="menuDrawer" class="menu-drawer">
    <div class="flex flex-col h-full">
        <div class="flex items-center justify-between mb-6">
            <span class="font-semibold text-slate-900">Menu</span>
            <button id="menuClose" class="p-2 text-slate-400 hover:text-slate-600">
                <i class="fas fa-times"></i>
            </button>
        </div>

        <nav class="flex-1 space-y-1">
            {{#each navItems}}
            <a href="{{this.href}}" class="flex items-center gap-3 px-3 py-2 rounded-lg text-slate-700 hover:bg-slate-100 {{#if this.active}}bg-slate-100{{/if}}">
                <i class="fas fa-{{this.icon}} w-5"></i>
                <span>{{this.label}}</span>
            </a>
            {{/each}}
        </nav>

        <button onclick="logout()" class="flex items-center gap-3 px-3 py-2 rounded-lg text-slate-700 hover:bg-slate-100 mt-auto">
            <i class="fas fa-sign-out-alt w-5"></i>
            <span>Logout</span>
        </button>
    </div>
</nav>
</script>
```

### Item List Partial (templates/item-list.hbs)

```handlebars
<script id="item-list-template" type="text/x-handlebars-template">
{{#if items.length}}
    {{#each items}}
    <div class="p-4 bg-white rounded-lg shadow-sm border border-slate-200 hover:shadow-md transition-shadow" data-id="{{this.id}}">
        <div class="flex justify-between items-start">
            <div>
                <h3 class="font-semibold text-slate-900">{{this.name}}</h3>
                <p class="text-sm text-slate-500 mt-1">{{this.description}}</p>
            </div>
            <div class="flex gap-2">
                <button onclick="editItem('{{this.id}}')" class="p-2 text-slate-400 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors">
                    <i class="fas fa-edit"></i>
                </button>
                <button onclick="deleteItem('{{this.id}}')" class="p-2 text-slate-400 hover:text-rose-600 hover:bg-rose-50 rounded-lg transition-colors">
                    <i class="fas fa-trash"></i>
                </button>
            </div>
        </div>
    </div>
    {{/each}}
{{else}}
    <div class="text-center py-8">
        <i class="fas fa-inbox text-slate-400 text-3xl mb-3"></i>
        <p class="text-slate-500">No items found</p>
    </div>
{{/if}}
</script>
```

### Page Template (index.html)

```html
<!DOCTYPE html>
<html lang="en" class="h-full bg-slate-50">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Dashboard - Project Name</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.2/css/all.min.css" rel="stylesheet">
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap" rel="stylesheet">
    <link href="/static/css/style.css" rel="stylesheet">
    <script src="https://cdn.jsdelivr.net/npm/handlebars@4.7.8/dist/handlebars.min.js"></script>
    <style>body { font-family: 'Inter', sans-serif; }</style>
</head>
<body class="h-full">

    <!-- Loading Spinner -->
    <div id="loadingSpinner" class="flex-1 flex justify-center items-center h-full">
        <i class="fas fa-circle-notch fa-spin text-indigo-500 text-3xl"></i>
    </div>

    <!-- Main Content -->
    <div id="mainContent" class="hidden">
        <div id="nav"></div>

        <main class="pt-14 lg:pl-64 p-6">
            <div id="itemsList" class="space-y-4"></div>
        </main>
    </div>

    <!-- Handlebars Templates -->
    {{> nav}}
    {{> item-list}}

    <!-- Scripts -->
    <script src="/static/js/common.js"></script>
    <script src="/static/js/menu.js"></script>
    <script src="/static/js/dashboard.js"></script>
</body>
</html>
```

### Using Templates in JavaScript

```javascript
// Compile templates once on load
const templates = {};

function initTemplates() {
    document.querySelectorAll('script[type="text/x-handlebars-template"]').forEach(script => {
        const name = script.id.replace('-template', '');
        templates[name] = Handlebars.compile(script.innerHTML);
    });
}

// Render a template
function render(templateName, data) {
    if (!templates[templateName]) {
        console.error(`Template "${templateName}" not found`);
        return '';
    }
    return templates[templateName](data);
}

// Example: Render navigation
function renderNav() {
    const html = render('nav', {
        appName: 'My App',
        navItems: [
            { href: '/', icon: 'home', label: 'Home', active: true },
            { href: '/settings', icon: 'cog', label: 'Settings' }
        ]
    });
    document.getElementById('nav').innerHTML = html;
}

// Example: Render item list
function renderItems(items) {
    const html = render('item-list', { items });
    document.getElementById('itemsList').innerHTML = html;
}

// Initialize on page load
document.addEventListener('DOMContentLoaded', () => {
    initTemplates();
    renderNav();
});
```

## Common JavaScript (js/common.js)

```javascript
// =============================================================================
// Configuration
// =============================================================================

const API_BASE = window.API_CONFIG?.API_BASE_URL || '';


// =============================================================================
// Toast Notifications
// =============================================================================

function showToast(msg, type = 'success') {
    let container = document.getElementById('toast-container');
    if (!container) {
        container = document.createElement('div');
        container.id = 'toast-container';
        container.className = 'fixed bottom-4 right-4 z-50 flex flex-col gap-3 pointer-events-none';
        document.body.appendChild(container);
    }

    const toast = document.createElement('div');
    const bgClass = type === 'success' ? 'bg-emerald-500'
                  : type === 'danger' ? 'bg-rose-500'
                  : 'bg-blue-500';

    toast.className = `${bgClass} text-white px-6 py-4 rounded-lg shadow-xl transform transition-all duration-300 translate-y-10 opacity-0 flex items-center gap-4 pointer-events-auto min-w-[300px]`;

    const icon = type === 'success' ? 'check-circle'
               : type === 'danger' ? 'exclamation-circle'
               : 'info-circle';

    toast.innerHTML = `
        <i class="fas fa-${icon} text-xl"></i>
        <span class="font-medium text-sm">${msg}</span>
        <button onclick="this.parentElement.remove()" class="ml-auto hover:bg-white/20 rounded-full p-1 w-6 h-6 flex items-center justify-center transition-colors">
            <i class="fas fa-times text-xs"></i>
        </button>
    `;

    container.appendChild(toast);

    // Animate in
    requestAnimationFrame(() => {
        toast.classList.remove('translate-y-10', 'opacity-0');
    });

    // Auto-remove after 4 seconds
    setTimeout(() => {
        if (toast.parentElement) {
            toast.classList.add('translate-y-4', 'opacity-0');
            setTimeout(() => toast.remove(), 300);
        }
    }, 4000);
}


// =============================================================================
// API Client
// =============================================================================

async function apiRequest(endpoint, options = {}) {
    try {
        const response = await fetch(`${API_BASE}${endpoint}`, {
            ...options,
            credentials: 'include',  // Include cookies for session auth
            headers: {
                'Content-Type': 'application/json',
                ...options.headers
            }
        });

        // Handle 401 - redirect to login
        if (response.status === 401) {
            if (endpoint !== '/auth/login' && endpoint !== '/auth/me') {
                window.location.href = '/login';
                throw new Error('Session expired');
            }
        }

        if (!response.ok) {
            const error = await response.json();
            throw new Error(error.detail || 'Request failed');
        }

        return await response.json();
    } catch (error) {
        if (endpoint !== '/auth/me') {
            showToast(error.message, 'danger');
        }
        throw error;
    }
}


// =============================================================================
// Auth Helpers
// =============================================================================

async function checkAuth() {
    try {
        const user = await apiRequest('/auth/me');
        return user;
    } catch {
        window.location.href = '/login';
        return null;
    }
}

async function logout() {
    try {
        await apiRequest('/auth/logout', { method: 'POST' });
        window.location.href = '/login';
    } catch (error) {
        console.error(error);
    }
}


// =============================================================================
// Utility Functions
// =============================================================================

function formatBytes(bytes) {
    if (bytes === 0) return '0 B';
    const k = 1024;
    const sizes = ['B', 'KB', 'MB', 'GB', 'TB'];
    const i = Math.floor(Math.log(bytes) / Math.log(k));
    return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i];
}

function formatSpeed(bps) {
    return formatBytes(bps) + '/s';
}

function formatDate(dateStr) {
    const date = new Date(dateStr);
    return date.toLocaleDateString() + ' ' + date.toLocaleTimeString();
}

function debounce(func, wait) {
    let timeout;
    return function executedFunction(...args) {
        const later = () => {
            clearTimeout(timeout);
            func(...args);
        };
        clearTimeout(timeout);
        timeout = setTimeout(later, wait);
    };
}


// =============================================================================
// DOM Helpers
// =============================================================================

const $ = (sel) => document.querySelector(sel);
const $$ = (sel) => document.querySelectorAll(sel);

function show(el) {
    if (typeof el === 'string') el = $(el);
    el?.classList.remove('hidden');
}

function hide(el) {
    if (typeof el === 'string') el = $(el);
    el?.classList.add('hidden');
}
```

## Page-Specific JavaScript (js/dashboard.js)

```javascript
// dashboard.js - Example page script using Handlebars templates
document.addEventListener('DOMContentLoaded', init);

let pollInterval = null;

async function init() {
    initTemplates();  // Initialize Handlebars templates
    renderNav();      // Render navigation

    const user = await checkAuth();
    if (!user) return;

    hide('#loadingSpinner');
    show('#mainContent');

    setupMenuListeners();  // From menu.js
    await loadData();
    setupEventListeners();
    startPolling();
}

function setupEventListeners() {
    $('#searchForm')?.addEventListener('submit', handleSearch);
    $('#refreshBtn')?.addEventListener('click', loadData);
}

async function loadData() {
    try {
        const items = await apiRequest('/api/items');
        // Use Handlebars template instead of inline HTML
        const html = render('item-list', { items });
        $('#itemsList').innerHTML = html;
    } catch (error) {
        console.error('Failed to load data:', error);
    }
}

async function handleSearch(e) {
    e.preventDefault();
    const query = $('#searchInput').value;
    const items = await apiRequest(`/api/items?q=${encodeURIComponent(query)}`);
    const html = render('item-list', { items });
    $('#itemsList').innerHTML = html;
}

async function deleteItem(id) {
    if (!confirm('Delete this item?')) return;

    try {
        await apiRequest(`/api/items/${id}`, { method: 'DELETE' });
        showToast('Item deleted');
        await loadData();
    } catch (error) {
        console.error('Delete failed:', error);
    }
}

// Polling for live updates
function startPolling(intervalMs = 5000) {
    stopPolling();
    pollInterval = setInterval(loadData, intervalMs);
}

function stopPolling() {
    if (pollInterval) {
        clearInterval(pollInterval);
        pollInterval = null;
    }
}

// Stop polling when page hidden
document.addEventListener('visibilitychange', () => {
    document.hidden ? stopPolling() : startPolling();
});
```

## Menu JavaScript (js/menu.js)

```javascript
// menu.js - Hamburger menu logic (separate file for reuse)

function setupMenuListeners() {
    const menuToggle = document.getElementById('menuToggle');
    const menuClose = document.getElementById('menuClose');
    const menuOverlay = document.getElementById('menuOverlay');

    menuToggle?.addEventListener('click', openMenu);
    menuClose?.addEventListener('click', closeMenu);
    menuOverlay?.addEventListener('click', closeMenu);
}

function openMenu() {
    document.getElementById('menuDrawer')?.classList.add('open');
    document.getElementById('menuOverlay')?.classList.add('active');
    document.body.style.overflow = 'hidden';
}

function closeMenu() {
    document.getElementById('menuDrawer')?.classList.remove('open');
    document.getElementById('menuOverlay')?.classList.remove('active');
    document.body.style.overflow = '';
}
```

## Modal Pattern (templates/modal.hbs)

```handlebars
<script id="modal-template" type="text/x-handlebars-template">
<div id="{{id}}" class="fixed inset-0 z-50 overflow-y-auto hidden">
    <div class="flex items-center justify-center min-h-screen px-4">
        <!-- Backdrop -->
        <div class="fixed inset-0 bg-slate-500 bg-opacity-75" onclick="closeModal('{{id}}')"></div>

        <!-- Modal Content -->
        <div class="relative bg-white rounded-lg shadow-xl max-w-lg w-full p-6">
            <div class="absolute top-4 right-4">
                <button onclick="closeModal('{{id}}')" class="text-slate-400 hover:text-slate-600">
                    <i class="fas fa-times"></i>
                </button>
            </div>

            <h3 class="text-lg font-semibold text-slate-900 mb-4">{{title}}</h3>

            <form id="{{id}}Form" onsubmit="{{onSubmit}}(event)">
                <div class="space-y-4">
                    {{#each fields}}
                    <div>
                        <label class="block text-sm font-medium text-slate-700 mb-1">{{this.label}}</label>
                        {{#if this.textarea}}
                        <textarea id="{{this.id}}" rows="3" class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-transparent">{{this.value}}</textarea>
                        {{else}}
                        <input type="{{this.type}}" id="{{this.id}}" value="{{this.value}}" class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-transparent">
                        {{/if}}
                    </div>
                    {{/each}}
                </div>
                <div class="mt-6 flex justify-end gap-3">
                    <button type="button" onclick="closeModal('{{id}}')" class="px-4 py-2 text-slate-700 hover:bg-slate-100 rounded-lg transition-colors">
                        Cancel
                    </button>
                    <button type="submit" class="px-4 py-2 bg-indigo-600 text-white rounded-lg hover:bg-indigo-700 transition-colors">
                        {{submitLabel}}
                    </button>
                </div>
            </form>
        </div>
    </div>
</div>
</script>
```

```javascript
// Modal helpers (in common.js)
function openModal(id) {
    document.getElementById(id)?.classList.remove('hidden');
}

function closeModal(id) {
    document.getElementById(id)?.classList.add('hidden');
}

// Render a modal dynamically
function renderModal(options) {
    const html = render('modal', {
        id: options.id,
        title: options.title,
        onSubmit: options.onSubmit,
        submitLabel: options.submitLabel || 'Save',
        fields: options.fields
    });
    document.getElementById('modals').innerHTML = html;
}

// Example usage:
// renderModal({
//     id: 'editModal',
//     title: 'Edit Item',
//     onSubmit: 'handleEditSubmit',
//     fields: [
//         { id: 'editName', label: 'Name', type: 'text', value: item.name },
//         { id: 'editDescription', label: 'Description', textarea: true, value: item.description }
//     ]
// });
```

## Custom CSS (css/style.css)

```css
/* Custom styles beyond Tailwind */

/* Menu drawer */
.menu-drawer {
    position: fixed;
    bottom: 0;
    left: 0;
    width: 280px;
    height: calc(100vh - 60px);
    background: white;
    box-shadow: 4px 0 20px rgba(0, 0, 0, 0.1);
    transform: translateX(-100%);
    transition: transform 0.3s ease;
    z-index: 100;
    border-top-right-radius: 16px;
    padding: 16px;
}

.menu-drawer.open {
    transform: translateX(0);
}

.menu-overlay {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.5);
    opacity: 0;
    pointer-events: none;
    transition: opacity 0.3s ease;
    z-index: 99;
}

.menu-overlay.active {
    opacity: 1;
    pointer-events: auto;
}

/* Bottom sheet drawer */
.management-drawer {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    background: white;
    border-top-left-radius: 16px;
    border-top-right-radius: 16px;
    box-shadow: 0 -4px 20px rgba(0, 0, 0, 0.15);
    transform: translateY(100%);
    transition: transform 0.3s ease;
    z-index: 100;
    max-height: 85vh;
    overflow-y: auto;
}

.management-drawer.open {
    transform: translateY(0);
}

/* Loading skeleton */
.skeleton {
    background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
    background-size: 200% 100%;
    animation: skeleton-loading 1.5s infinite;
}

@keyframes skeleton-loading {
    0% { background-position: 200% 0; }
    100% { background-position: -200% 0; }
}

/* Custom scrollbar */
::-webkit-scrollbar {
    width: 8px;
    height: 8px;
}

::-webkit-scrollbar-track {
    background: #f1f5f9;
}

::-webkit-scrollbar-thumb {
    background: #cbd5e1;
    border-radius: 4px;
}

::-webkit-scrollbar-thumb:hover {
    background: #94a3b8;
}
```

## File Upload with Drag and Drop

```html
<div id="dropZone" class="border-2 border-dashed border-slate-300 rounded-lg p-8 text-center hover:border-indigo-500 hover:bg-indigo-50 transition-colors cursor-pointer">
    <i class="fas fa-cloud-upload-alt text-slate-400 text-3xl mb-3"></i>
    <p class="text-slate-600">
        <span class="font-medium text-indigo-600">Upload a file</span> or drag and drop
    </p>
    <p class="text-sm text-slate-500 mt-1">Supported formats: PDF, DOCX, TXT</p>
    <input type="file" id="fileInput" class="hidden" accept=".pdf,.docx,.txt">
</div>
```

```javascript
const dropZone = document.getElementById('dropZone');
const fileInput = document.getElementById('fileInput');

dropZone.addEventListener('click', () => fileInput.click());

dropZone.addEventListener('dragover', (e) => {
    e.preventDefault();
    dropZone.classList.add('border-indigo-500', 'bg-indigo-50');
});

dropZone.addEventListener('dragleave', () => {
    dropZone.classList.remove('border-indigo-500', 'bg-indigo-50');
});

dropZone.addEventListener('drop', (e) => {
    e.preventDefault();
    dropZone.classList.remove('border-indigo-500', 'bg-indigo-50');
    const files = e.dataTransfer.files;
    if (files.length) handleFileUpload(files[0]);
});

fileInput.addEventListener('change', (e) => {
    if (e.target.files.length) handleFileUpload(e.target.files[0]);
});

async function handleFileUpload(file) {
    const formData = new FormData();
    formData.append('file', file);

    try {
        const response = await fetch(`${API_BASE}/api/upload`, {
            method: 'POST',
            body: formData,
            credentials: 'include'
        });

        if (!response.ok) throw new Error('Upload failed');

        const result = await response.json();
        showToast(`Uploaded: ${result.filename}`);
    } catch (error) {
        showToast(error.message, 'danger');
    }
}
```

## Tailwind Color Palette

Standard colors used:

| Color | Usage |
|-------|-------|
| `slate-50` | Page background |
| `slate-100` | Card backgrounds |
| `slate-200` | Borders |
| `slate-400` | Muted text, icons |
| `slate-600` | Secondary text |
| `slate-900` | Primary text |
| `indigo-500/600` | Primary actions |
| `emerald-500` | Success |
| `rose-500` | Danger/error |
| `amber-500` | Warning |

## No Build Tools

- Load Tailwind directly from CDN
- Load Handlebars from CDN
- Write vanilla JS (ES6+, no transpilation needed)
- No npm, webpack, vite, etc.
- Serve static files from FastAPI using `StaticFiles` mount
- Keep static files inside the package: `project_name/static/`
- Keep templates inside the package: `project_name/templates/`
