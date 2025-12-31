# Frontend Patterns

Simple static HTML/JS/CSS frontends. No build tools, no npm, no frameworks.

## Stack

- **Tailwind CSS** via CDN (`tailwind.js` or `cdn.tailwindcss.com`)
- **Vanilla JavaScript** (no React, Vue, etc.)
- **Font Awesome** for icons via CDN
- **Inter font** from Google Fonts

## Directory Structure

```
project_name/
└── static/
    ├── index.html          # Main page
    ├── login.html          # Login page
    ├── admin.html          # Admin page
    ├── tailwind.js         # Local Tailwind (optional)
    ├── style.css           # Custom styles
    ├── js/
    │   ├── common.js       # Shared utilities, API client
    │   ├── auth.js         # Auth-specific code
    │   ├── dashboard.js    # Page-specific code
    │   └── admin.js        # Admin page code
    └── vendor/             # Third-party libs (optional)
```

## HTML Template

```html
<!DOCTYPE html>
<html lang="en" class="h-full bg-slate-50">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Page Title - Project Name</title>

    <!-- Tailwind CSS -->
    <script src="/static/tailwind.js"></script>
    <!-- Or CDN: <script src="https://cdn.tailwindcss.com"></script> -->

    <!-- Font Awesome -->
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.2/css/all.min.css" rel="stylesheet">

    <!-- Inter Font -->
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap" rel="stylesheet">

    <!-- Custom styles -->
    <link href="/static/style.css" rel="stylesheet">

    <style>body { font-family: 'Inter', sans-serif; }</style>
</head>
<body class="h-full">

    <!-- Loading Spinner -->
    <div id="loadingSpinner" class="flex-1 flex justify-center items-center">
        <i class="fas fa-circle-notch fa-spin text-indigo-500 text-3xl"></i>
    </div>

    <!-- Main Content -->
    <div id="mainContent" class="hidden">
        <!-- Page content here -->
    </div>

    <!-- Scripts -->
    <script src="/static/js/common.js"></script>
    <script src="/static/js/dashboard.js"></script>
</body>
</html>
```

## Common JavaScript (common.js)

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

## Page-Specific JavaScript

```javascript
// dashboard.js - Example page script
document.addEventListener('DOMContentLoaded', init);

let pollInterval = null;

async function init() {
    const user = await checkAuth();
    if (!user) return;

    hide('#loadingSpinner');
    show('#mainContent');

    await loadData();
    setupEventListeners();
    startPolling();
}

function setupEventListeners() {
    // Form submission
    $('#searchForm')?.addEventListener('submit', handleSearch);

    // Button clicks
    $('#refreshBtn')?.addEventListener('click', loadData);

    // Delegated events for dynamic content
    $('#itemsList')?.addEventListener('click', handleItemClick);
}

async function loadData() {
    try {
        const items = await apiRequest('/api/items');
        renderItems(items);
    } catch (error) {
        console.error('Failed to load data:', error);
    }
}

function renderItems(items) {
    const container = $('#itemsList');
    if (!container) return;

    if (items.length === 0) {
        container.innerHTML = `
            <div class="text-center py-8">
                <i class="fas fa-inbox text-slate-400 text-3xl mb-3"></i>
                <p class="text-slate-500">No items found</p>
            </div>
        `;
        return;
    }

    container.innerHTML = items.map(item => `
        <div class="p-4 bg-white rounded-lg shadow-sm border border-slate-200 hover:shadow-md transition-shadow" data-id="${item.id}">
            <div class="flex justify-between items-start">
                <div>
                    <h3 class="font-semibold text-slate-900">${item.name}</h3>
                    <p class="text-sm text-slate-500 mt-1">${item.description || ''}</p>
                </div>
                <div class="flex gap-2">
                    <button onclick="editItem('${item.id}')" class="p-2 text-slate-400 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors">
                        <i class="fas fa-edit"></i>
                    </button>
                    <button onclick="deleteItem('${item.id}')" class="p-2 text-slate-400 hover:text-rose-600 hover:bg-rose-50 rounded-lg transition-colors">
                        <i class="fas fa-trash"></i>
                    </button>
                </div>
            </div>
        </div>
    `).join('');
}

async function handleSearch(e) {
    e.preventDefault();
    const query = $('#searchInput').value;
    const items = await apiRequest(`/api/items?q=${encodeURIComponent(query)}`);
    renderItems(items);
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
    if (document.hidden) {
        stopPolling();
    } else {
        startPolling();
    }
});
```

## Modal Pattern

```html
<!-- Modal -->
<div id="editModal" class="fixed inset-0 z-50 overflow-y-auto hidden">
    <div class="flex items-center justify-center min-h-screen px-4">
        <!-- Backdrop -->
        <div class="fixed inset-0 bg-slate-500 bg-opacity-75" onclick="closeModal('editModal')"></div>

        <!-- Modal Content -->
        <div class="relative bg-white rounded-lg shadow-xl max-w-lg w-full p-6">
            <div class="absolute top-4 right-4">
                <button onclick="closeModal('editModal')" class="text-slate-400 hover:text-slate-600">
                    <i class="fas fa-times"></i>
                </button>
            </div>

            <h3 class="text-lg font-semibold text-slate-900 mb-4">Edit Item</h3>

            <form id="editForm" onsubmit="handleEditSubmit(event)">
                <div class="space-y-4">
                    <div>
                        <label class="block text-sm font-medium text-slate-700 mb-1">Name</label>
                        <input type="text" id="editName" class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-transparent">
                    </div>
                    <div>
                        <label class="block text-sm font-medium text-slate-700 mb-1">Description</label>
                        <textarea id="editDescription" rows="3" class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-transparent"></textarea>
                    </div>
                </div>
                <div class="mt-6 flex justify-end gap-3">
                    <button type="button" onclick="closeModal('editModal')" class="px-4 py-2 text-slate-700 hover:bg-slate-100 rounded-lg transition-colors">
                        Cancel
                    </button>
                    <button type="submit" class="px-4 py-2 bg-indigo-600 text-white rounded-lg hover:bg-indigo-700 transition-colors">
                        Save
                    </button>
                </div>
            </form>
        </div>
    </div>
</div>
```

```javascript
// Modal helpers
function openModal(id) {
    document.getElementById(id)?.classList.remove('hidden');
}

function closeModal(id) {
    document.getElementById(id)?.classList.add('hidden');
}
```

## Custom CSS (style.css)

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

- Load Tailwind directly (`/static/tailwind.js` or CDN)
- Write vanilla JS (ES6+, no transpilation needed)
- No npm, webpack, vite, etc.
- Serve static files from FastAPI
