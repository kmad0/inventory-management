<template>
  <div class="app">
    <header class="top-bar">
      <div class="logo">
        <h1>{{ t('nav.companyName') }}</h1>
        <span class="subtitle">{{ t('nav.subtitle') }}</span>
      </div>
      <div class="top-bar-actions">
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </div>
    </header>

    <div class="app-body">
      <aside class="sidebar" :class="{ collapsed: isCollapsed }">
        <nav class="sidebar-nav">
          <router-link
            v-for="item in navItems"
            :key="item.path"
            :to="item.path"
            class="nav-item"
            :class="{ active: $route.path === item.path }"
            :title="isCollapsed ? item.label : ''"
          >
            <svg
              viewBox="0 0 24 24"
              stroke="currentColor"
              stroke-width="2"
              fill="none"
              stroke-linecap="round"
              stroke-linejoin="round"
              v-html="item.icon"
            ></svg>
            <span class="nav-label">{{ item.label }}</span>
          </router-link>
        </nav>

        <button
          class="sidebar-toggle"
          :class="{ collapsed: isCollapsed }"
          @click="toggleSidebar"
          :title="isCollapsed ? 'Expand sidebar' : 'Collapse sidebar'"
        >
          <svg
            viewBox="0 0 24 24"
            stroke="currentColor"
            stroke-width="2"
            fill="none"
            stroke-linecap="round"
            stroke-linejoin="round"
            width="16"
            height="16"
          >
            <polyline v-if="!isCollapsed" points="15 18 9 12 15 6" />
            <polyline v-else points="9 18 15 12 9 6" />
          </svg>
        </button>
      </aside>

      <div class="content-area">
        <FilterBar />
        <main class="main-content">
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
  </div>
</template>

<script>
import { ref, computed, onMounted, onUnmounted } from 'vue'
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
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])

    // Sidebar collapse state
    const isCollapsed = ref(localStorage.getItem('sidebar-collapsed') === 'true')

    const toggleSidebar = () => {
      isCollapsed.value = !isCollapsed.value
      localStorage.setItem('sidebar-collapsed', String(isCollapsed.value))
    }

    const handleResize = () => {
      if (window.innerWidth < 768) {
        isCollapsed.value = true
        localStorage.setItem('sidebar-collapsed', 'true')
      } else if (window.innerWidth > 1024) {
        isCollapsed.value = false
        localStorage.setItem('sidebar-collapsed', 'false')
      }
    }

    // Nav items defined inside setup so computed labels can access t()
    const navItems = [
      {
        path: '/',
        label: computed(() => t('nav.overview')),
        icon: `<path d="M3 9l9-7 9 7v11a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2z"/><polyline points="9 22 9 12 15 12 15 22"/>`
      },
      {
        path: '/inventory',
        label: computed(() => t('nav.inventory')),
        icon: `<line x1="16.5" y1="9.4" x2="7.5" y2="4.21"/><path d="M21 16V8a2 2 0 0 0-1-1.73l-7-4a2 2 0 0 0-2 0l-7 4A2 2 0 0 0 3 8v8a2 2 0 0 0 1 1.73l7 4a2 2 0 0 0 2 0l7-4A2 2 0 0 0 21 16z"/><polyline points="3.27 6.96 12 12.01 20.73 6.96"/><line x1="12" y1="22.08" x2="12" y2="12"/>`
      },
      {
        path: '/orders',
        label: computed(() => t('nav.orders')),
        icon: `<path d="M16 4h2a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2H6a2 2 0 0 1-2-2V6a2 2 0 0 1 2-2h2"/><rect x="8" y="2" width="8" height="4" rx="1" ry="1"/><line x1="9" y1="12" x2="15" y2="12"/><line x1="9" y1="16" x2="13" y2="16"/>`
      },
      {
        path: '/spending',
        label: computed(() => t('nav.finance')),
        icon: `<line x1="12" y1="1" x2="12" y2="23"/><path d="M17 5H9.5a3.5 3.5 0 0 0 0 7h5a3.5 3.5 0 0 1 0 7H6"/>`
      },
      {
        path: '/demand',
        label: computed(() => t('nav.demandForecast')),
        icon: `<polyline points="23 6 13.5 15.5 8.5 10.5 1 18"/><polyline points="17 6 23 6 23 12"/>`
      },
      {
        path: '/reports',
        label: 'Reports',
        icon: `<line x1="18" y1="20" x2="18" y2="10"/><line x1="12" y1="20" x2="12" y2="4"/><line x1="6" y1="20" x2="6" y2="14"/>`
      },
      {
        path: '/restocking',
        label: 'Restocking',
        icon: `<polyline points="23 4 23 10 17 10"/><polyline points="1 20 1 14 7 14"/><path d="M3.51 9a9 9 0 0 1 14.85-3.36L23 10M1 14l4.64 4.36A9 9 0 0 0 20.49 15"/>`
      }
    ]

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
        apiTasks.value.unshift(newTask)
      } catch (err) {
        console.error('Failed to add task:', err)
      }
    }

    const deleteTask = async (taskId) => {
      try {
        const isMockTask = currentUser.value.tasks.some(t => t.id === taskId)

        if (isMockTask) {
          const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
          if (index !== -1) {
            currentUser.value.tasks.splice(index, 1)
          }
        } else {
          await api.deleteTask(taskId)
          apiTasks.value = apiTasks.value.filter(t => t.id !== taskId)
        }
      } catch (err) {
        console.error('Failed to delete task:', err)
      }
    }

    const toggleTask = async (taskId) => {
      try {
        const mockTask = currentUser.value.tasks.find(t => t.id === taskId)

        if (mockTask) {
          mockTask.status = mockTask.status === 'pending' ? 'completed' : 'pending'
        } else {
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

    onMounted(() => {
      loadTasks()
      if (window.innerWidth < 768) {
        isCollapsed.value = true
        localStorage.setItem('sidebar-collapsed', 'true')
      }
      window.addEventListener('resize', handleResize)
    })

    onUnmounted(() => {
      window.removeEventListener('resize', handleResize)
    })

    return {
      t,
      showProfileDetails,
      showTasks,
      tasks,
      addTask,
      deleteTask,
      toggleTask,
      isCollapsed,
      toggleSidebar,
      navItems
    }
  }
}
</script>

<style>
/* =============================================
   Cyberpunk 2050 — CSS Custom Properties
   ============================================= */
:root {
  --bg: #050714;
  --surface: #0d1117;
  --surface-alt: #080d14;
  --border: #1a2744;
  --border-soft: #0d1a2e;
  --border-mid: #243860;
  --text-primary: #e0f0ff;
  --text-secondary: #7090b0;
  --text-body: #a0b8d0;
  --text-muted: #506880;
  --neon-cyan: #00f5ff;
  --neon-pink: #ff2d78;
  --neon-purple: #b44fff;
  --neon-green: #00ff9d;
  --neon-yellow: #ffe600;
  --nav-hover-bg: #0d1e38;
  --nav-active-bg: #001a33;
  --nav-active-color: #00f5ff;
  --shadow-sm: 0 0 10px rgba(0, 245, 255, 0.08);
  --shadow-card: 0 0 20px rgba(0, 245, 255, 0.12);
  --toggle-bg: #0d1e38;
  --toggle-color: #7090b0;
  --toggle-hover-bg: #1a2e50;
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Share Tech Mono', 'Courier New', monospace;
  background: var(--bg);
  color: var(--text-body);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

.app {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

/* Top bar */
.top-bar {
  position: sticky;
  top: 0;
  z-index: 200;
  height: 60px;
  background: var(--surface);
  border-bottom: 1px solid var(--neon-cyan);
  box-shadow: 0 0 20px rgba(0, 245, 255, 0.15);
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 1.5rem;
  flex-shrink: 0;
}

.top-bar-actions {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.logo {
  display: flex;
  align-items: baseline;
  gap: 0.75rem;
}

.logo h1 {
  font-size: 1.375rem;
  font-weight: 700;
  color: var(--neon-cyan);
  letter-spacing: -0.025em;
  text-shadow: 0 0 20px rgba(0, 245, 255, 0.7);
}

.subtitle {
  font-size: 0.813rem;
  color: var(--text-secondary);
  font-weight: 400;
  padding-left: 0.75rem;
  border-left: 1px solid var(--border-mid);
}

/* App body: sidebar + content */
.app-body {
  display: flex;
  flex: 1;
  min-height: 0;
}

/* Sidebar */
.sidebar {
  position: sticky;
  top: 60px;
  height: calc(100vh - 60px);
  overflow: hidden;
  background: var(--surface);
  border-right: 1px solid var(--border);
  width: 220px;
  flex-shrink: 0;
  transition: width 0.25s ease;
  display: flex;
  flex-direction: column;
}

.sidebar.collapsed {
  width: 60px;
}

.sidebar-nav {
  flex: 1;
  padding: 0.5rem 0;
  overflow: hidden;
}

/* Nav items */
.nav-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.625rem 1rem;
  color: var(--text-secondary);
  text-decoration: none;
  font-size: 0.875rem;
  font-weight: 500;
  border-radius: 6px;
  margin: 2px 8px;
  white-space: nowrap;
  overflow: hidden;
  transition: background 0.15s, color 0.15s;
}

.nav-item:hover {
  background: var(--nav-hover-bg);
  color: var(--neon-cyan);
}

.nav-item.active {
  background: var(--nav-active-bg);
  color: var(--neon-cyan);
  text-shadow: 0 0 8px rgba(0, 245, 255, 0.5);
}

.nav-item svg {
  flex-shrink: 0;
  width: 18px;
  height: 18px;
}

.sidebar.collapsed .nav-label {
  display: none;
}

.sidebar.collapsed .nav-item {
  justify-content: center;
  margin: 2px 6px;
  padding: 0.625rem;
}

/* Toggle button */
.sidebar-toggle {
  width: 100%;
  border: none;
  background: none;
  padding: 0.875rem 1rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  color: var(--toggle-color);
  border-top: 1px solid var(--border);
  flex-shrink: 0;
  transition: color 0.15s;
}

.sidebar-toggle:hover {
  color: var(--neon-cyan);
}

.sidebar-toggle.collapsed {
  justify-content: center;
}

/* Content area */
.content-area {
  flex: 1;
  min-width: 0;
  overflow: auto;
}

.main-content {
  max-width: 1600px;
  width: 100%;
  margin: 0 auto;
  padding: 1.5rem 2rem;
}

/* Page header */
.page-header {
  margin-bottom: 1.5rem;
}

.page-header h2 {
  font-size: 1.875rem;
  font-weight: 700;
  color: var(--neon-cyan);
  margin-bottom: 0.375rem;
  letter-spacing: -0.02em;
  text-shadow: 0 0 20px rgba(0, 245, 255, 0.5);
}

.page-header p {
  color: var(--text-secondary);
  font-size: 0.938rem;
}

/* Stats grid */
.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1.25rem;
  margin-bottom: 1.5rem;
}

.stat-card {
  background: var(--surface);
  padding: 1.25rem;
  border-radius: 10px;
  border: 1px solid var(--border);
  border-left: 3px solid var(--neon-cyan);
  transition: all 0.2s ease;
}

.stat-card:hover {
  border-color: var(--border-mid);
  box-shadow: var(--shadow-card);
}

.stat-label {
  color: var(--text-secondary);
  font-size: 0.875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  margin-bottom: 0.625rem;
}

.stat-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: var(--neon-cyan);
  letter-spacing: -0.025em;
  text-shadow: 0 0 10px rgba(0, 245, 255, 0.5);
}

.stat-card.warning .stat-value {
  color: var(--neon-yellow);
  text-shadow: 0 0 10px rgba(255, 230, 0, 0.5);
}

.stat-card.success .stat-value {
  color: var(--neon-green);
  text-shadow: 0 0 10px rgba(0, 255, 157, 0.5);
}

.stat-card.danger .stat-value {
  color: var(--neon-pink);
  text-shadow: 0 0 10px rgba(255, 45, 120, 0.5);
}

.stat-card.info .stat-value {
  color: var(--neon-purple);
  text-shadow: 0 0 10px rgba(180, 79, 255, 0.5);
}

/* Cards */
.card {
  background: var(--surface);
  border-radius: 10px;
  padding: 1.25rem;
  border: 1px solid var(--border);
  border-top: 2px solid var(--neon-cyan);
  margin-bottom: 1.25rem;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  padding-bottom: 0.875rem;
  border-bottom: 1px solid var(--border);
}

.card-title {
  font-size: 1.125rem;
  font-weight: 700;
  color: var(--neon-cyan);
  letter-spacing: -0.025em;
}

/* Tables */
.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: var(--surface-alt);
  border-top: 1px solid var(--border);
  border-bottom: 1px solid var(--border);
}

th {
  text-align: left;
  padding: 0.5rem 0.75rem;
  font-weight: 600;
  color: var(--neon-cyan);
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.08em;
}

td {
  padding: 0.5rem 0.75rem;
  border-top: 1px solid var(--border-soft);
  color: var(--text-body);
  font-size: 0.875rem;
}

tbody tr {
  transition: background-color 0.15s ease;
}

tbody tr:hover {
  background: rgba(0, 245, 255, 0.04);
}

/* Badges */
.badge {
  display: inline-block;
  padding: 0.313rem 0.75rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.badge.success {
  background: #003320;
  color: var(--neon-green);
  border: 1px solid var(--neon-green);
}

.badge.warning {
  background: #332200;
  color: var(--neon-yellow);
  border: 1px solid var(--neon-yellow);
}

.badge.danger {
  background: #330011;
  color: var(--neon-pink);
  border: 1px solid var(--neon-pink);
}

.badge.info {
  background: #0d0033;
  color: var(--neon-purple);
  border: 1px solid var(--neon-purple);
}

.badge.increasing {
  background: #003320;
  color: var(--neon-green);
  border: 1px solid var(--neon-green);
}

.badge.decreasing {
  background: #330011;
  color: var(--neon-pink);
  border: 1px solid var(--neon-pink);
}

.badge.stable {
  background: #001433;
  color: #60a5fa;
  border: 1px solid #60a5fa;
}

.badge.high {
  background: #330011;
  color: var(--neon-pink);
  border: 1px solid var(--neon-pink);
}

.badge.medium {
  background: #332200;
  color: var(--neon-yellow);
  border: 1px solid var(--neon-yellow);
}

.badge.low {
  background: #0d0033;
  color: var(--neon-purple);
  border: 1px solid var(--neon-purple);
}

/* Loading / error */
.loading {
  text-align: center;
  padding: 3rem;
  color: var(--text-secondary);
  font-size: 0.938rem;
}

.error {
  background: #1a0008;
  border: 1px solid var(--neon-pink);
  color: var(--neon-pink);
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
  font-size: 0.938rem;
}
</style>
