<template>
  <!-- Mobile topbar (hidden on desktop) -->
  <div class="mobile-topbar">
    <button class="mobile-menu-btn" @click="mobileOpen = !mobileOpen" aria-label="Toggle menu">
      <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <template v-if="!mobileOpen">
          <line x1="3" y1="6" x2="21" y2="6"/><line x1="3" y1="12" x2="21" y2="12"/><line x1="3" y1="18" x2="21" y2="18"/>
        </template>
        <template v-else>
          <line x1="18" y1="6" x2="6" y2="18"/><line x1="6" y1="6" x2="18" y2="18"/>
        </template>
      </svg>
    </button>
    <div class="mobile-brand">
      <div class="brand-mark">F</div>
      <span class="brand-name">FactoryIMS</span>
    </div>
  </div>

  <!-- Backdrop for mobile drawer -->
  <div v-if="mobileOpen" class="sidebar-backdrop" @click="mobileOpen = false"></div>

  <div class="app-layout">
    <aside class="sidebar" :class="{ collapsed, 'mobile-open': mobileOpen }">
      <!-- Brand -->
      <div class="sidebar-brand">
        <div class="brand-mark">F</div>
        <span v-if="!collapsed" class="brand-name">FactoryIMS</span>
      </div>

      <!-- Nav -->
      <nav class="sidebar-nav">
        <router-link
          v-for="item in navItems"
          :key="item.path"
          :to="item.path"
          :title="collapsed ? item.label : ''"
          :class="['nav-item', { active: isActive(item.path) }]"
        >
          <span class="nav-icon" v-html="item.icon"></span>
          <span v-if="!collapsed" class="nav-label">{{ item.label }}</span>
        </router-link>
      </nav>

      <!-- Bottom -->
      <div class="sidebar-bottom">
        <div class="sidebar-bottom-item" v-if="!collapsed">
          <LanguageSwitcher />
        </div>
        <div class="sidebar-bottom-item">
          <ProfileMenu
            @show-profile-details="showProfileDetails = true"
            @show-tasks="showTasks = true"
          />
        </div>
        <button
          class="collapse-toggle"
          @click="toggleCollapse"
          :title="collapsed ? 'Expand sidebar' : 'Collapse sidebar'"
        >
          <svg v-if="!collapsed" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="15 18 9 12 15 6"/></svg>
          <svg v-else width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="9 18 15 12 9 6"/></svg>
          <span v-if="!collapsed" class="collapse-label">Collapse</span>
        </button>
      </div>
    </aside>

    <div class="main-area">
      <FilterBar />
      <main class="content-wrapper">
        <router-view />
      </main>
    </div>
  </div>

  <ProfileDetailsModal
    :is-open="showProfileDetails"
    @close="showProfileDetails = false"
  />

  <TasksModal
    :is-open="showTasks"
    :tasks="tasks"
    @close="showTasks = false"
    @add-task="addTask"
    @delete-task="deleteTask"
    @toggle-task="toggleTask"
  />
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { useRoute } from 'vue-router'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import FilterBar from './components/FilterBar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const route = useRoute()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])

    // Sidebar collapse state — persisted to localStorage
    const collapsed = ref(localStorage.getItem('sidebar-collapsed') === 'true')
    // Mobile drawer state
    const mobileOpen = ref(false)

    const toggleCollapse = () => {
      collapsed.value = !collapsed.value
      localStorage.setItem('sidebar-collapsed', collapsed.value)
    }

    // Nav items with inline SVG icons (18×18, stroke-based)
    const navItems = [
      {
        path: '/',
        label: 'Overview',
        icon: '<svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" stroke-linejoin="round"><path d="M3 9l9-7 9 7v11a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2z"/><polyline points="9 22 9 12 15 12 15 22"/></svg>'
      },
      {
        path: '/inventory',
        label: 'Inventory',
        icon: '<svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" stroke-linejoin="round"><path d="M21 16V8a2 2 0 0 0-1-1.73l-7-4a2 2 0 0 0-2 0l-7 4A2 2 0 0 0 3 8v8a2 2 0 0 0 1 1.73l7 4a2 2 0 0 0 2 0l7-4A2 2 0 0 0 21 16z"/><polyline points="3.27 6.96 12 12.01 20.73 6.96"/><line x1="12" y1="22.08" x2="12" y2="12"/></svg>'
      },
      {
        path: '/orders',
        label: 'Orders',
        icon: '<svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" stroke-linejoin="round"><circle cx="9" cy="21" r="1"/><circle cx="20" cy="21" r="1"/><path d="M1 1h4l2.68 13.39a2 2 0 0 0 2 1.61h9.72a2 2 0 0 0 2-1.61L23 6H6"/></svg>'
      },
      {
        path: '/spending',
        label: 'Finance',
        icon: '<svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" stroke-linejoin="round"><line x1="12" y1="1" x2="12" y2="23"/><path d="M17 5H9.5a3.5 3.5 0 0 0 0 7h5a3.5 3.5 0 0 1 0 7H6"/></svg>'
      },
      {
        path: '/demand',
        label: 'Demand Forecast',
        icon: '<svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" stroke-linejoin="round"><polyline points="23 6 13.5 15.5 8.5 10.5 1 18"/><polyline points="17 6 23 6 23 12"/></svg>'
      },
      {
        path: '/reports',
        label: 'Reports',
        icon: '<svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" stroke-linejoin="round"><line x1="18" y1="20" x2="18" y2="10"/><line x1="12" y1="20" x2="12" y2="4"/><line x1="6" y1="20" x2="6" y2="14"/><line x1="2" y1="20" x2="22" y2="20"/></svg>'
      },
      {
        path: '/restocking',
        label: 'Restocking',
        icon: '<svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" stroke-linejoin="round"><polyline points="23 4 23 10 17 10"/><polyline points="1 20 1 14 7 14"/><path d="M3.51 9a9 9 0 0 1 14.85-3.36L23 10M1 14l4.64 4.36A9 9 0 0 0 20.49 15"/></svg>'
      }
    ]

    // Active route detection — exact match for root, prefix match for others
    const isActive = (path) => {
      if (path === '/') return route.path === '/'
      return route.path.startsWith(path)
    }

    // Merge mock tasks from currentUser with API tasks
    const tasks = computed(() => {
      return [...currentUser.value.tasks, ...apiTasks.value]
    })

    const loadTasks = async () => {
      try {
        apiTasks.value = await api.getTasks()
      } catch (err) {
        console.error('Failed to load tasks:', err)
      }
    }

    const addTask = async (taskData) => {
      try {
        const newTask = await api.createTask(taskData)
        // Add new task to the beginning of the array
        apiTasks.value.unshift(newTask)
      } catch (err) {
        console.error('Failed to add task:', err)
      }
    }

    const deleteTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const isMockTask = currentUser.value.tasks.some(t => t.id === taskId)

        if (isMockTask) {
          // Remove from mock tasks
          const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
          if (index !== -1) {
            currentUser.value.tasks.splice(index, 1)
          }
        } else {
          // Remove from API tasks
          await api.deleteTask(taskId)
          apiTasks.value = apiTasks.value.filter(t => t.id !== taskId)
        }
      } catch (err) {
        console.error('Failed to delete task:', err)
      }
    }

    const toggleTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const mockTask = currentUser.value.tasks.find(t => t.id === taskId)

        if (mockTask) {
          // Toggle mock task status
          mockTask.status = mockTask.status === 'pending' ? 'completed' : 'pending'
        } else {
          // Toggle API task
          const updatedTask = await api.toggleTask(taskId)
          const index = apiTasks.value.findIndex(t => t.id === taskId)
          if (index !== -1) {
            apiTasks.value[index] = updatedTask
          }
        }
      } catch (err) {
        console.error('Failed to toggle task:', err)
      }
    }

    onMounted(loadTasks)

    return {
      t,
      route,
      showProfileDetails,
      showTasks,
      tasks,
      addTask,
      deleteTask,
      toggleTask,
      collapsed,
      toggleCollapse,
      mobileOpen,
      navItems,
      isActive
    }
  }
}
</script>

<style>
/* ── Reset ─────────────────────────────────────────── */
* { margin: 0; padding: 0; box-sizing: border-box; }

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  background: #f8fafc;
  color: #1e293b;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

/* ── App Shell ──────────────────────────────────────── */
.app-layout {
  display: flex;
  min-height: 100vh;
}

/* ── Sidebar ────────────────────────────────────────── */
.sidebar {
  width: 240px;
  min-height: 100vh;
  background: #0f172a;
  display: flex;
  flex-direction: column;
  flex-shrink: 0;
  transition: width 0.22s ease;
  overflow: hidden;
  position: sticky;
  top: 0;
  height: 100vh;
}

.sidebar.collapsed {
  width: 64px;
}

/* Brand */
.sidebar-brand {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 1.25rem 1rem 1rem;
  border-bottom: 1px solid rgba(255,255,255,0.06);
  min-height: 64px;
  flex-shrink: 0;
}

.brand-mark {
  width: 32px;
  height: 32px;
  background: #2563eb;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  font-size: 1rem;
  color: #fff;
  flex-shrink: 0;
}

.brand-name {
  font-size: 1rem;
  font-weight: 700;
  color: #f8fafc;
  white-space: nowrap;
  letter-spacing: -0.02em;
}

/* Nav */
.sidebar-nav {
  flex: 1;
  padding: 0.75rem 0.5rem;
  display: flex;
  flex-direction: column;
  gap: 2px;
  overflow-y: auto;
  overflow-x: hidden;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.563rem 0.75rem;
  border-radius: 8px;
  color: #94a3b8;
  text-decoration: none;
  font-size: 0.875rem;
  font-weight: 500;
  transition: background 0.15s, color 0.15s;
  white-space: nowrap;
  min-height: 40px;
}

.sidebar.collapsed .nav-item {
  justify-content: center;
  padding: 0.563rem;
}

.nav-item:hover {
  background: rgba(255,255,255,0.06);
  color: #e2e8f0;
}

.nav-item.active {
  background: rgba(255,255,255,0.10);
  color: #f8fafc;
}

.nav-icon {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  width: 18px;
  height: 18px;
}

.nav-label {
  overflow: hidden;
  text-overflow: ellipsis;
}

/* Bottom */
.sidebar-bottom {
  padding: 0.5rem;
  border-top: 1px solid rgba(255,255,255,0.06);
  display: flex;
  flex-direction: column;
  gap: 2px;
  flex-shrink: 0;
}

.sidebar-bottom-item {
  padding: 0 0.25rem;
}

.collapse-toggle {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  width: 100%;
  padding: 0.563rem 0.75rem;
  background: none;
  border: none;
  border-radius: 8px;
  color: #64748b;
  font-size: 0.813rem;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.15s, color 0.15s;
  white-space: nowrap;
}

.sidebar.collapsed .collapse-toggle {
  justify-content: center;
  padding: 0.563rem;
}

.collapse-toggle:hover {
  background: rgba(255,255,255,0.06);
  color: #94a3b8;
}

/* ── Main Area ──────────────────────────────────────── */
.main-area {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 0;
  background: #f8fafc;
}

.content-wrapper {
  flex: 1;
  padding: 1.5rem 2rem;
  max-width: 1600px;
  width: 100%;
  margin: 0 auto;
}

/* ── Shared Page Layout ─────────────────────────────── */
.page-header {
  margin-bottom: 1.5rem;
}

.page-header h2 {
  font-size: 1.875rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.375rem;
  letter-spacing: -0.025em;
}

.page-header p {
  color: #64748b;
  font-size: 0.938rem;
}

/* ── Stats Grid ─────────────────────────────────────── */
.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 1.25rem;
  margin-bottom: 1.5rem;
}

.stat-card {
  background: white;
  padding: 1.25rem;
  border-radius: 10px;
  border: 1px solid #e2e8f0;
  transition: all 0.2s ease;
}

.stat-card:hover {
  border-color: #cbd5e1;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.06);
}

.stat-label {
  color: #64748b;
  font-size: 0.875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 0.625rem;
}

.stat-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.stat-card.warning .stat-value { color: #ea580c; }
.stat-card.success .stat-value { color: #059669; }
.stat-card.danger .stat-value  { color: #dc2626; }
.stat-card.info .stat-value    { color: #2563eb; }

/* ── Cards ──────────────────────────────────────────── */
.card {
  background: white;
  border-radius: 10px;
  padding: 1.25rem;
  border: 1px solid #e2e8f0;
  margin-bottom: 1.25rem;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  padding-bottom: 0.875rem;
  border-bottom: 1px solid #e2e8f0;
}

.card-title {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

/* ── Tables ─────────────────────────────────────────── */
.table-container { overflow-x: auto; }

table { width: 100%; border-collapse: collapse; }

thead {
  background: #f8fafc;
  border-top: 1px solid #e2e8f0;
  border-bottom: 1px solid #e2e8f0;
}

th {
  text-align: left;
  padding: 0.5rem 0.75rem;
  font-weight: 600;
  color: #475569;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

td {
  padding: 0.5rem 0.75rem;
  border-top: 1px solid #f1f5f9;
  color: #334155;
  font-size: 0.875rem;
}

tbody tr { transition: background-color 0.15s ease; }
tbody tr:hover { background: #f8fafc; }

/* ── Badges ─────────────────────────────────────────── */
.badge {
  display: inline-block;
  padding: 0.313rem 0.75rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.badge.success    { background: #d1fae5; color: #065f46; }
.badge.warning    { background: #fed7aa; color: #92400e; }
.badge.danger     { background: #fecaca; color: #991b1b; }
.badge.info       { background: #dbeafe; color: #1e40af; }
.badge.increasing { background: #d1fae5; color: #065f46; }
.badge.decreasing { background: #fecaca; color: #991b1b; }
.badge.stable     { background: #e0e7ff; color: #3730a3; }
.badge.high       { background: #fecaca; color: #991b1b; }
.badge.medium     { background: #fed7aa; color: #92400e; }
.badge.low        { background: #dbeafe; color: #1e40af; }

/* ── States ─────────────────────────────────────────── */
.loading {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}

.error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
  font-size: 0.938rem;
}

/* ── Mobile topbar (hidden on desktop) ─────────────── */
.mobile-topbar {
  display: none;
}

.sidebar-backdrop {
  display: none;
}

/* ── Tablet: auto icon-only at ≤1024px ─────────────── */
@media (max-width: 1024px) {
  .sidebar {
    width: 64px;
  }

  /* Force icon-only layout without needing the JS .collapsed class */
  .sidebar:not(.mobile-open) .nav-item {
    justify-content: center;
    padding: 0.563rem;
  }

  .sidebar:not(.mobile-open) .collapse-toggle {
    justify-content: center;
    padding: 0.563rem;
  }

  .sidebar:not(.mobile-open) .brand-name {
    display: none;
  }
}

/* ── Mobile: drawer overlay at ≤640px ──────────────── */
@media (max-width: 640px) {
  /* Thin topbar */
  .mobile-topbar {
    display: flex;
    align-items: center;
    gap: 0.75rem;
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    height: 52px;
    background: #0f172a;
    padding: 0 1rem;
    z-index: 300;
    border-bottom: 1px solid rgba(255,255,255,0.06);
  }

  .mobile-menu-btn {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 36px;
    height: 36px;
    background: none;
    border: none;
    border-radius: 8px;
    color: #94a3b8;
    cursor: pointer;
    transition: background 0.15s, color 0.15s;
    flex-shrink: 0;
  }

  .mobile-menu-btn:hover {
    background: rgba(255,255,255,0.08);
    color: #f8fafc;
  }

  .mobile-brand {
    display: flex;
    align-items: center;
    gap: 0.625rem;
  }

  .mobile-brand .brand-mark {
    width: 28px;
    height: 28px;
    font-size: 0.875rem;
  }

  .mobile-brand .brand-name {
    font-size: 0.938rem;
    font-weight: 700;
    color: #f8fafc;
    letter-spacing: -0.02em;
  }

  /* Sidebar becomes a fixed drawer */
  .sidebar {
    position: fixed;
    top: 52px; /* below mobile topbar */
    left: 0;
    height: calc(100vh - 52px);
    width: 240px !important; /* always full width when drawer opens */
    transform: translateX(-100%);
    transition: transform 0.22s ease;
    z-index: 250;
    min-height: unset;
  }

  .sidebar.mobile-open {
    transform: translateX(0);
  }

  /* Restore full nav-item layout inside drawer */
  .sidebar .nav-item {
    justify-content: flex-start !important;
    padding: 0.563rem 0.75rem !important;
  }

  .sidebar .collapse-toggle {
    justify-content: flex-start !important;
    padding: 0.563rem 0.75rem !important;
  }

  .sidebar .brand-name {
    display: block !important;
  }

  /* Semi-transparent backdrop */
  .sidebar-backdrop {
    display: block;
    position: fixed;
    inset: 52px 0 0 0;
    background: rgba(0,0,0,0.45);
    z-index: 200;
  }

  /* Push content below mobile topbar */
  .app-layout {
    padding-top: 52px;
  }
}
</style>
