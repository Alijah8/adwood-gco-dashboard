<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { supabase } from './supabase.js'

// ─── State ───
const jobs = ref([])
const packages = ref([])
const loading = ref(true)
const error = ref(null)

// Filters
const filterStatus = ref('all')
const filterMinFit = ref(0)
const filterMinSalary = ref(0)
const sortBy = ref('discovered_at')
const sortDir = ref('desc')
const searchQuery = ref('')

// Detail modal
const selectedJob = ref(null)
const showDetail = ref(false)

// ─── Status config ───
const STATUS_ORDER = ['preparing', 'ready_for_review', 'approved', 'applied', 'interviews', 'offer_secured', 'skipped', 'denied']
const STATUS_CONFIG = {
  new:              { label: 'New',             color: 'var(--status-new)',       icon: '&#9679;' },
  screening:        { label: 'Screening',       color: 'var(--status-preparing)', icon: '&#9681;' },
  qualified:        { label: 'Qualified',       color: 'var(--accent-teal)',      icon: '&#10003;' },
  rejected:         { label: 'Rejected',        color: 'var(--status-denied)',    icon: '&#10005;' },
  preparing:        { label: 'Preparing',       color: 'var(--status-preparing)', icon: '&#9681;' },
  ready_for_review: { label: 'Ready for Review',color: 'var(--status-review)',    icon: '&#9733;' },
  approved:         { label: 'Approved',        color: 'var(--status-approved)',  icon: '&#10003;' },
  applied:          { label: 'Applied',         color: 'var(--status-applied)',   icon: '&#10148;' },
  interviews:       { label: 'Interviews',      color: 'var(--status-interviews)',icon: '&#9742;' },
  offer_secured:    { label: 'Offer Secured',   color: 'var(--status-offer)',     icon: '&#9733;' },
  skipped:          { label: 'Skipped',         color: 'var(--status-skipped)',   icon: '&#8212;' },
  denied:           { label: 'Denied',          color: 'var(--status-denied)',    icon: '&#10005;' },
}

// ─── Computed ───
const combined = computed(() => {
  return jobs.value.map(job => {
    const pkg = packages.value.find(p => p.job_id === job.id)
    return { ...job, package: pkg || null }
  })
})

const displayStatus = (row) => {
  if (row.package) return row.package.status
  return row.status
}

const filtered = computed(() => {
  let rows = combined.value

  // Search
  if (searchQuery.value.trim()) {
    const q = searchQuery.value.toLowerCase()
    rows = rows.filter(r =>
      r.company?.toLowerCase().includes(q) ||
      r.title?.toLowerCase().includes(q) ||
      r.location?.toLowerCase().includes(q)
    )
  }

  // Status filter
  if (filterStatus.value !== 'all') {
    rows = rows.filter(r => displayStatus(r) === filterStatus.value)
  }

  // Fit score filter
  if (filterMinFit.value > 0) {
    rows = rows.filter(r => (r.fit_score || 0) >= filterMinFit.value)
  }

  // Salary filter
  if (filterMinSalary.value > 0) {
    rows = rows.filter(r => (r.salary_max || r.salary_min || 0) >= filterMinSalary.value)
  }

  // Sort
  rows = [...rows].sort((a, b) => {
    let valA, valB
    if (sortBy.value === 'salary') {
      valA = a.salary_max || a.salary_min || 0
      valB = b.salary_max || b.salary_min || 0
    } else if (sortBy.value === 'fit_score') {
      valA = a.fit_score || 0
      valB = b.fit_score || 0
    } else if (sortBy.value === 'stage') {
      valA = STATUS_ORDER.indexOf(displayStatus(a))
      valB = STATUS_ORDER.indexOf(displayStatus(b))
      if (valA === -1) valA = 99
      if (valB === -1) valB = 99
    } else {
      valA = a.discovered_at || ''
      valB = b.discovered_at || ''
    }
    if (sortDir.value === 'desc') return valA > valB ? -1 : valA < valB ? 1 : 0
    return valA < valB ? -1 : valA > valB ? 1 : 0
  })

  return rows
})

// Stats
const stats = computed(() => {
  const all = combined.value
  const total = all.length
  const readyForReview = all.filter(r => displayStatus(r) === 'ready_for_review').length
  const applied = all.filter(r => ['applied', 'interviews', 'offer_secured'].includes(displayStatus(r))).length
  const interviews = all.filter(r => displayStatus(r) === 'interviews').length
  const offers = all.filter(r => displayStatus(r) === 'offer_secured').length
  const avgFit = total > 0 ? Math.round(all.reduce((s, r) => s + (r.fit_score || 0), 0) / total) : 0
  return { total, readyForReview, applied, interviews, offers, avgFit }
})

// Pipeline counts for visual
const pipelineCounts = computed(() => {
  const all = combined.value
  return STATUS_ORDER.map(status => ({
    status,
    config: STATUS_CONFIG[status],
    count: all.filter(r => displayStatus(r) === status).length
  }))
})

// ─── Data fetching ───
async function fetchData() {
  loading.value = true
  error.value = null
  try {
    const [jobsRes, pkgRes] = await Promise.all([
      supabase.from('job_postings').select('*').order('discovered_at', { ascending: false }),
      supabase.from('application_packages').select('*')
    ])
    if (jobsRes.error) throw jobsRes.error
    if (pkgRes.error) throw pkgRes.error
    jobs.value = jobsRes.data || []
    packages.value = pkgRes.data || []
  } catch (e) {
    error.value = e.message
  } finally {
    loading.value = false
  }
}

// ─── Actions ───
async function updatePackageStatus(pkg, newStatus) {
  const updates = { status: newStatus }
  if (newStatus === 'approved' || newStatus === 'applied') {
    updates.reviewed_at = new Date().toISOString()
  }
  const { error: err } = await supabase
    .from('application_packages')
    .update(updates)
    .eq('id', pkg.id)
  if (err) {
    alert('Failed to update: ' + err.message)
    return
  }
  await fetchData()
}

// ─── Helpers ───
function formatSalary(min, max) {
  if (!min && !max) return '—'
  const fmt = (n) => '$' + (n / 1000).toFixed(0) + 'K'
  if (min && max && min !== max) return `${fmt(min)} – ${fmt(max)}`
  return fmt(max || min)
}

function formatDate(dateStr) {
  if (!dateStr) return '—'
  const d = new Date(dateStr)
  const now = new Date()
  const diffMs = now - d
  const diffDays = Math.floor(diffMs / 86400000)
  if (diffDays === 0) return 'Today'
  if (diffDays === 1) return 'Yesterday'
  if (diffDays < 7) return `${diffDays}d ago`
  return d.toLocaleDateString('en-US', { month: 'short', day: 'numeric' })
}

function fitScoreColor(score) {
  if (!score) return 'var(--text-muted)'
  if (score >= 85) return 'var(--accent-green)'
  if (score >= 70) return 'var(--accent-amber)'
  return 'var(--accent-red)'
}

function openJob(row) {
  selectedJob.value = row
  showDetail.value = true
}

function closeDetail() {
  showDetail.value = false
  selectedJob.value = null
}

function toggleSort(field) {
  if (sortBy.value === field) {
    sortDir.value = sortDir.value === 'desc' ? 'asc' : 'desc'
  } else {
    sortBy.value = field
    sortDir.value = 'desc'
  }
}

onMounted(fetchData)
</script>

<template>
  <div class="dashboard">
    <!-- Header -->
    <header class="header">
      <div class="header-left">
        <div class="logo-mark">GCO</div>
        <div class="header-text">
          <h1>Getting Corporate Offers</h1>
          <p class="header-subtitle">Application Pipeline</p>
        </div>
      </div>
      <div class="header-right">
        <button class="btn btn-ghost" @click="fetchData" :disabled="loading">
          <span :class="{ spinning: loading }">&#8635;</span>
          Refresh
        </button>
      </div>
    </header>

    <!-- Stats Bar -->
    <div class="stats-bar">
      <div class="stat-card">
        <span class="stat-value">{{ stats.total }}</span>
        <span class="stat-label">Total Jobs</span>
      </div>
      <div class="stat-card stat-highlight" style="--highlight: var(--status-review)">
        <span class="stat-value">{{ stats.readyForReview }}</span>
        <span class="stat-label">Ready to Review</span>
      </div>
      <div class="stat-card" style="--highlight: var(--status-applied)">
        <span class="stat-value">{{ stats.applied }}</span>
        <span class="stat-label">Applied</span>
      </div>
      <div class="stat-card" style="--highlight: var(--status-interviews)">
        <span class="stat-value">{{ stats.interviews }}</span>
        <span class="stat-label">Interviews</span>
      </div>
      <div class="stat-card" style="--highlight: var(--status-offer)">
        <span class="stat-value">{{ stats.offers }}</span>
        <span class="stat-label">Offers</span>
      </div>
      <div class="stat-card">
        <span class="stat-value" :style="{ color: fitScoreColor(stats.avgFit) }">{{ stats.avgFit }}%</span>
        <span class="stat-label">Avg Fit</span>
      </div>
    </div>

    <!-- Pipeline Visual -->
    <div class="pipeline" v-if="combined.length > 0">
      <div
        class="pipeline-stage"
        v-for="stage in pipelineCounts"
        :key="stage.status"
        :class="{ active: filterStatus === stage.status, empty: stage.count === 0 }"
        @click="filterStatus = filterStatus === stage.status ? 'all' : stage.status"
      >
        <div class="pipeline-count" :style="{ color: stage.config.color }">{{ stage.count }}</div>
        <div class="pipeline-label">{{ stage.config.label }}</div>
        <div class="pipeline-bar" :style="{ background: stage.config.color, opacity: stage.count > 0 ? 1 : 0.15 }"></div>
      </div>
    </div>

    <!-- Controls -->
    <div class="controls">
      <div class="search-box">
        <span class="search-icon">&#128269;</span>
        <input
          type="text"
          v-model="searchQuery"
          placeholder="Search company, role, location..."
          class="search-input"
        />
      </div>
      <div class="control-group">
        <select v-model="filterStatus" class="control-select">
          <option value="all">All Statuses</option>
          <option v-for="(cfg, key) in STATUS_CONFIG" :key="key" :value="key">{{ cfg.label }}</option>
        </select>
        <select v-model="sortBy" class="control-select" @change="sortDir = 'desc'">
          <option value="discovered_at">Date Found</option>
          <option value="salary">Highest Salary</option>
          <option value="fit_score">Best Fit</option>
          <option value="stage">By Stage</option>
        </select>
        <button class="btn btn-icon" @click="sortDir = sortDir === 'desc' ? 'asc' : 'desc'" :title="sortDir === 'desc' ? 'Descending' : 'Ascending'">
          {{ sortDir === 'desc' ? '&#9660;' : '&#9650;' }}
        </button>
      </div>
    </div>

    <!-- Loading / Error / Empty -->
    <div v-if="loading" class="state-msg">
      <div class="loader"></div>
      <p>Loading pipeline...</p>
    </div>
    <div v-else-if="error" class="state-msg state-error">
      <p>&#9888; {{ error }}</p>
      <button class="btn btn-ghost" @click="fetchData">Retry</button>
    </div>
    <div v-else-if="combined.length === 0" class="state-msg state-empty">
      <div class="empty-icon">&#128188;</div>
      <h3>No jobs in the pipeline yet</h3>
      <p>Jobs will appear here once the ingestion workflows start feeding data.</p>
    </div>
    <div v-else-if="filtered.length === 0" class="state-msg state-empty">
      <p>No jobs match your current filters.</p>
      <button class="btn btn-ghost" @click="filterStatus = 'all'; filterMinFit = 0; filterMinSalary = 0; searchQuery = ''">Clear Filters</button>
    </div>

    <!-- Table -->
    <div v-else class="table-wrap">
      <table class="job-table">
        <thead>
          <tr>
            <th class="col-status">Status</th>
            <th class="col-company clickable" @click="toggleSort('discovered_at')">
              Company / Role
              <span v-if="sortBy === 'discovered_at'" class="sort-arrow">{{ sortDir === 'desc' ? '&#9660;' : '&#9650;' }}</span>
            </th>
            <th class="col-location">Location</th>
            <th class="col-salary clickable" @click="toggleSort('salary')">
              Salary
              <span v-if="sortBy === 'salary'" class="sort-arrow">{{ sortDir === 'desc' ? '&#9660;' : '&#9650;' }}</span>
            </th>
            <th class="col-fit clickable" @click="toggleSort('fit_score')">
              Fit
              <span v-if="sortBy === 'fit_score'" class="sort-arrow">{{ sortDir === 'desc' ? '&#9660;' : '&#9650;' }}</span>
            </th>
            <th class="col-date">Found</th>
            <th class="col-actions">Actions</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="row in filtered" :key="row.id" class="job-row" @click="openJob(row)">
            <td class="col-status">
              <span class="status-badge" :style="{ '--badge-color': (STATUS_CONFIG[displayStatus(row)] || STATUS_CONFIG.new).color }">
                <span class="status-dot"></span>
                {{ (STATUS_CONFIG[displayStatus(row)] || STATUS_CONFIG.new).label }}
              </span>
            </td>
            <td class="col-company">
              <div class="company-name">{{ row.company }}</div>
              <div class="role-name">{{ row.title }}</div>
            </td>
            <td class="col-location">
              <span class="location-text">{{ row.location || '—' }}</span>
            </td>
            <td class="col-salary">
              <span class="salary-text" :class="{ 'salary-good': (row.salary_max || row.salary_min || 0) >= 80000 }">
                {{ formatSalary(row.salary_min, row.salary_max) }}
              </span>
            </td>
            <td class="col-fit">
              <span v-if="row.fit_score" class="fit-badge" :style="{ color: fitScoreColor(row.fit_score) }">
                {{ row.fit_score }}%
              </span>
              <span v-else class="fit-none">—</span>
            </td>
            <td class="col-date">{{ formatDate(row.discovered_at) }}</td>
            <td class="col-actions" @click.stop>
              <div class="action-btns">
                <a v-if="row.url" :href="row.url" target="_blank" class="btn btn-xs btn-action" title="Open Job Posting">Apply &#8599;</a>
                <a href="/resume.docx" download class="btn btn-xs btn-action-alt" title="Download Resume">Resume &#128196;</a>
                <a v-if="row.package?.drive_folder_url" :href="row.package.drive_folder_url" target="_blank" class="btn btn-xs btn-action-alt" title="Open Documents">Docs &#128193;</a>
              </div>
            </td>
          </tr>
        </tbody>
      </table>
      <div class="table-footer">
        <span class="result-count">{{ filtered.length }} of {{ combined.length }} jobs</span>
      </div>
    </div>

    <!-- Detail Modal -->
    <Teleport to="body">
      <div v-if="showDetail && selectedJob" class="modal-overlay" @click.self="closeDetail">
        <div class="modal">
          <div class="modal-header">
            <div>
              <h2>{{ selectedJob.company }}</h2>
              <p class="modal-subtitle">{{ selectedJob.title }}</p>
            </div>
            <button class="btn btn-icon modal-close" @click="closeDetail">&#10005;</button>
          </div>

          <div class="modal-body">
            <!-- Status + Quick Actions -->
            <div class="modal-section">
              <div class="modal-status">
                <span class="status-badge status-badge-lg" :style="{ '--badge-color': (STATUS_CONFIG[displayStatus(selectedJob)] || STATUS_CONFIG.new).color }">
                  <span class="status-dot"></span>
                  {{ (STATUS_CONFIG[displayStatus(selectedJob)] || STATUS_CONFIG.new).label }}
                </span>
              </div>
              <div v-if="selectedJob.package" class="quick-actions">
                <button
                  v-if="displayStatus(selectedJob) === 'ready_for_review'"
                  class="btn btn-sm btn-approve"
                  @click="updatePackageStatus(selectedJob.package, 'approved')"
                >Approve</button>
                <button
                  v-if="displayStatus(selectedJob) === 'approved'"
                  class="btn btn-sm btn-applied"
                  @click="updatePackageStatus(selectedJob.package, 'applied')"
                >Mark Applied</button>
                <button
                  v-if="displayStatus(selectedJob) === 'applied'"
                  class="btn btn-sm btn-interviews"
                  @click="updatePackageStatus(selectedJob.package, 'interviews')"
                >Interviews</button>
                <button
                  v-if="displayStatus(selectedJob) === 'interviews'"
                  class="btn btn-sm btn-offer"
                  @click="updatePackageStatus(selectedJob.package, 'offer_secured')"
                >Offer Secured</button>
                <button
                  v-if="!['skipped', 'denied', 'offer_secured'].includes(displayStatus(selectedJob))"
                  class="btn btn-sm btn-skip"
                  @click="updatePackageStatus(selectedJob.package, 'skipped')"
                >Skip</button>
                <button
                  v-if="!['skipped', 'denied', 'offer_secured'].includes(displayStatus(selectedJob))"
                  class="btn btn-sm btn-deny"
                  @click="updatePackageStatus(selectedJob.package, 'denied')"
                >Denied</button>
              </div>
            </div>

            <!-- Details Grid -->
            <div class="detail-grid">
              <div class="detail-item">
                <span class="detail-label">Location</span>
                <span class="detail-value">{{ selectedJob.location || '—' }}</span>
              </div>
              <div class="detail-item">
                <span class="detail-label">Salary Range</span>
                <span class="detail-value">{{ formatSalary(selectedJob.salary_min, selectedJob.salary_max) }}</span>
              </div>
              <div class="detail-item">
                <span class="detail-label">Fit Score</span>
                <span class="detail-value" :style="{ color: fitScoreColor(selectedJob.fit_score) }">
                  {{ selectedJob.fit_score ? selectedJob.fit_score + '%' : '—' }}
                </span>
              </div>
              <div class="detail-item">
                <span class="detail-label">Source</span>
                <span class="detail-value" style="text-transform: capitalize">{{ selectedJob.source }}</span>
              </div>
              <div class="detail-item">
                <span class="detail-label">Discovered</span>
                <span class="detail-value">{{ selectedJob.discovered_at ? new Date(selectedJob.discovered_at).toLocaleDateString('en-US', { month: 'long', day: 'numeric', year: 'numeric' }) : '—' }}</span>
              </div>
              <div class="detail-item">
                <span class="detail-label">Location Match</span>
                <span class="detail-value">{{ selectedJob.location_match === true ? 'Yes' : selectedJob.location_match === false ? 'No' : '—' }}</span>
              </div>
            </div>

            <!-- Fit Reasoning -->
            <div v-if="selectedJob.fit_reasoning" class="modal-section">
              <h4>Fit Analysis</h4>
              <p class="fit-reasoning">{{ selectedJob.fit_reasoning }}</p>
            </div>

            <!-- Cover Letter Preview -->
            <div v-if="selectedJob.package?.cover_letter" class="modal-section">
              <h4>Cover Letter</h4>
              <pre class="cover-letter-preview">{{ selectedJob.package.cover_letter }}</pre>
            </div>

            <!-- Links -->
            <div class="modal-links">
              <a v-if="selectedJob.url" :href="selectedJob.url" target="_blank" class="btn btn-primary">
                Open Job Posting &#8599;
              </a>
              <a href="/resume.docx" download class="btn btn-secondary">
                Download Resume &#128196;
              </a>
              <a v-if="selectedJob.package?.drive_folder_url" :href="selectedJob.package.drive_folder_url" target="_blank" class="btn btn-secondary">
                Open Documents &#128193;
              </a>
            </div>
          </div>
        </div>
      </div>
    </Teleport>
  </div>
</template>

<style>
/* ─── Dashboard Layout ─── */
.dashboard {
  max-width: 1400px;
  margin: 0 auto;
  padding: 24px 32px;
  min-height: 100vh;
}

/* ─── Header ─── */
.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding-bottom: 24px;
  border-bottom: 1px solid var(--border-primary);
  margin-bottom: 24px;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 16px;
}

.logo-mark {
  font-family: var(--font-mono);
  font-weight: 600;
  font-size: 13px;
  letter-spacing: 0.1em;
  color: var(--bg-primary);
  background: var(--accent-blue);
  padding: 6px 10px;
  border-radius: var(--radius-sm);
}

.header h1 {
  font-size: 20px;
  font-weight: 600;
  letter-spacing: -0.02em;
  line-height: 1.2;
}

.header-subtitle {
  font-size: 12px;
  color: var(--text-tertiary);
  font-family: var(--font-mono);
  text-transform: uppercase;
  letter-spacing: 0.08em;
  margin-top: 2px;
}

/* ─── Stats Bar ─── */
.stats-bar {
  display: grid;
  grid-template-columns: repeat(6, 1fr);
  gap: 12px;
  margin-bottom: 20px;
}

.stat-card {
  background: var(--bg-secondary);
  border: 1px solid var(--border-primary);
  border-radius: var(--radius-md);
  padding: 16px;
  text-align: center;
  transition: border-color 0.2s;
}

.stat-card:hover { border-color: var(--bg-hover); }

.stat-value {
  display: block;
  font-family: var(--font-mono);
  font-size: 28px;
  font-weight: 600;
  line-height: 1;
  margin-bottom: 6px;
}

.stat-label {
  display: block;
  font-size: 11px;
  color: var(--text-tertiary);
  text-transform: uppercase;
  letter-spacing: 0.06em;
}

/* ─── Pipeline Visual ─── */
.pipeline {
  display: flex;
  gap: 2px;
  margin-bottom: 20px;
  background: var(--bg-secondary);
  border: 1px solid var(--border-primary);
  border-radius: var(--radius-md);
  padding: 12px 8px;
  overflow-x: auto;
}

.pipeline-stage {
  flex: 1;
  min-width: 100px;
  text-align: center;
  padding: 8px 6px;
  border-radius: var(--radius-sm);
  cursor: pointer;
  transition: background 0.15s;
  user-select: none;
}

.pipeline-stage:hover { background: var(--bg-tertiary); }
.pipeline-stage.active { background: var(--bg-elevated); }
.pipeline-stage.empty { opacity: 0.5; }

.pipeline-count {
  font-family: var(--font-mono);
  font-size: 20px;
  font-weight: 600;
  line-height: 1;
}

.pipeline-label {
  font-size: 10px;
  color: var(--text-tertiary);
  text-transform: uppercase;
  letter-spacing: 0.04em;
  margin: 4px 0 6px;
  white-space: nowrap;
}

.pipeline-bar {
  height: 3px;
  border-radius: 2px;
  transition: opacity 0.2s;
}

/* ─── Controls ─── */
.controls {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  margin-bottom: 16px;
}

.search-box {
  position: relative;
  flex: 1;
  max-width: 360px;
}

.search-icon {
  position: absolute;
  left: 12px;
  top: 50%;
  transform: translateY(-50%);
  font-size: 13px;
  opacity: 0.4;
}

.search-input {
  width: 100%;
  padding: 9px 12px 9px 36px;
  background: var(--bg-secondary);
  border: 1px solid var(--border-primary);
  border-radius: var(--radius-md);
  color: var(--text-primary);
  font-size: 13px;
  outline: none;
  transition: border-color 0.15s;
}

.search-input::placeholder { color: var(--text-muted); }
.search-input:focus { border-color: var(--accent-blue); }

.control-group {
  display: flex;
  align-items: center;
  gap: 8px;
}

.control-select {
  padding: 9px 12px;
  background: var(--bg-secondary);
  border: 1px solid var(--border-primary);
  border-radius: var(--radius-md);
  color: var(--text-secondary);
  font-size: 13px;
  outline: none;
  cursor: pointer;
}

.control-select:focus { border-color: var(--accent-blue); }

/* ─── Buttons ─── */
.btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 8px 14px;
  border: 1px solid var(--border-primary);
  border-radius: var(--radius-md);
  background: var(--bg-secondary);
  color: var(--text-primary);
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s;
  text-decoration: none;
  white-space: nowrap;
}

.btn:hover { background: var(--bg-tertiary); border-color: var(--bg-hover); text-decoration: none; }

.btn-ghost { background: transparent; border-color: transparent; color: var(--text-secondary); }
.btn-ghost:hover { background: var(--bg-tertiary); color: var(--text-primary); }

.btn-icon { padding: 8px 10px; min-width: 36px; justify-content: center; }

.btn-xs { padding: 4px 10px; font-size: 12px; border-radius: var(--radius-sm); }
.btn-sm { padding: 6px 12px; font-size: 12px; }

.btn-action { background: var(--accent-blue); color: white; border-color: var(--accent-blue); }
.btn-action:hover { opacity: 0.85; }

.btn-action-alt { background: var(--bg-tertiary); }
.btn-action-alt:hover { background: var(--bg-elevated); }

.btn-primary { background: var(--accent-blue); color: white; border-color: var(--accent-blue); }
.btn-primary:hover { opacity: 0.9; }

.btn-secondary { background: var(--bg-tertiary); }

.btn-approve { background: var(--accent-purple); color: white; border-color: var(--accent-purple); }
.btn-applied { background: var(--accent-teal); color: var(--bg-primary); border-color: var(--accent-teal); }
.btn-interviews { background: var(--accent-green); color: var(--bg-primary); border-color: var(--accent-green); }
.btn-offer { background: #22c55e; color: var(--bg-primary); border-color: #22c55e; }
.btn-skip { background: transparent; border-color: var(--text-muted); color: var(--text-secondary); }
.btn-deny { background: transparent; border-color: var(--accent-red); color: var(--accent-red); }

.spinning { display: inline-block; animation: spin 0.8s linear infinite; }
@keyframes spin { to { transform: rotate(360deg); } }

/* ─── State Messages ─── */
.state-msg {
  text-align: center;
  padding: 64px 24px;
  color: var(--text-secondary);
}

.state-msg h3 { margin-bottom: 8px; color: var(--text-primary); }
.state-error { color: var(--accent-red); }

.empty-icon {
  font-size: 48px;
  margin-bottom: 16px;
  opacity: 0.5;
}

.loader {
  width: 24px;
  height: 24px;
  border: 2px solid var(--border-primary);
  border-top-color: var(--accent-blue);
  border-radius: 50%;
  animation: spin 0.6s linear infinite;
  margin: 0 auto 12px;
}

/* ─── Table ─── */
.table-wrap {
  background: var(--bg-secondary);
  border: 1px solid var(--border-primary);
  border-radius: var(--radius-lg);
  overflow: hidden;
}

.job-table {
  width: 100%;
  border-collapse: collapse;
}

.job-table thead {
  background: var(--bg-tertiary);
}

.job-table th {
  padding: 10px 16px;
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: var(--text-tertiary);
  text-align: left;
  border-bottom: 1px solid var(--border-primary);
  user-select: none;
  white-space: nowrap;
}

.job-table th.clickable { cursor: pointer; }
.job-table th.clickable:hover { color: var(--text-secondary); }

.sort-arrow {
  font-size: 9px;
  margin-left: 4px;
  color: var(--accent-blue);
}

.job-row {
  cursor: pointer;
  transition: background 0.1s;
}

.job-row:hover { background: var(--bg-tertiary); }

.job-row td {
  padding: 12px 16px;
  border-bottom: 1px solid var(--border-subtle);
  vertical-align: middle;
}

.job-row:last-child td { border-bottom: none; }

/* Status Badge */
.status-badge {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  font-weight: 500;
  color: var(--badge-color);
  white-space: nowrap;
}

.status-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: var(--badge-color);
  flex-shrink: 0;
}

.status-badge-lg { font-size: 14px; }
.status-badge-lg .status-dot { width: 9px; height: 9px; }

/* Company/Role */
.company-name {
  font-weight: 600;
  font-size: 14px;
  line-height: 1.3;
}

.role-name {
  font-size: 12px;
  color: var(--text-secondary);
  margin-top: 2px;
}

/* Location */
.location-text {
  font-size: 13px;
  color: var(--text-secondary);
}

/* Salary */
.salary-text {
  font-family: var(--font-mono);
  font-size: 13px;
  font-weight: 500;
  color: var(--text-secondary);
}

.salary-good { color: var(--accent-green); }

/* Fit */
.fit-badge {
  font-family: var(--font-mono);
  font-size: 13px;
  font-weight: 600;
}

.fit-none { color: var(--text-muted); }

/* Date */
.col-date {
  font-size: 12px;
  color: var(--text-tertiary);
  font-family: var(--font-mono);
  white-space: nowrap;
}

/* Actions */
.action-btns {
  display: flex;
  gap: 6px;
}

/* Table Footer */
.table-footer {
  padding: 10px 16px;
  border-top: 1px solid var(--border-primary);
  font-size: 12px;
  color: var(--text-tertiary);
}

/* ─── Modal ─── */
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.7);
  backdrop-filter: blur(4px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  padding: 24px;
  animation: fadeIn 0.15s ease-out;
}

@keyframes fadeIn { from { opacity: 0; } }

.modal {
  background: var(--bg-secondary);
  border: 1px solid var(--border-primary);
  border-radius: var(--radius-lg);
  width: 100%;
  max-width: 640px;
  max-height: 80vh;
  overflow-y: auto;
  box-shadow: var(--shadow-lg);
  animation: slideUp 0.2s ease-out;
}

@keyframes slideUp { from { transform: translateY(12px); opacity: 0; } }

.modal-header {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  padding: 24px 24px 16px;
  border-bottom: 1px solid var(--border-primary);
}

.modal-header h2 {
  font-size: 20px;
  font-weight: 600;
  letter-spacing: -0.01em;
}

.modal-subtitle {
  font-size: 14px;
  color: var(--text-secondary);
  margin-top: 4px;
}

.modal-close {
  color: var(--text-tertiary);
  font-size: 16px;
}

.modal-body { padding: 24px; }

.modal-section { margin-bottom: 20px; }
.modal-section h4 {
  font-size: 12px;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: var(--text-tertiary);
  margin-bottom: 8px;
}

.modal-status { margin-bottom: 12px; }

.quick-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-bottom: 20px;
}

.detail-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 16px;
  margin-bottom: 20px;
  padding: 16px;
  background: var(--bg-tertiary);
  border-radius: var(--radius-md);
}

.detail-label {
  display: block;
  font-size: 11px;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: var(--text-tertiary);
  margin-bottom: 4px;
}

.detail-value {
  display: block;
  font-size: 14px;
  font-weight: 500;
}

.fit-reasoning {
  font-size: 13px;
  color: var(--text-secondary);
  line-height: 1.6;
  background: var(--bg-tertiary);
  padding: 12px 16px;
  border-radius: var(--radius-md);
}

.cover-letter-preview {
  font-size: 12px;
  color: var(--text-secondary);
  line-height: 1.7;
  background: var(--bg-tertiary);
  padding: 16px;
  border-radius: var(--radius-md);
  white-space: pre-wrap;
  font-family: var(--font-sans);
  max-height: 200px;
  overflow-y: auto;
}

.modal-links {
  display: flex;
  gap: 10px;
  margin-top: 20px;
  padding-top: 20px;
  border-top: 1px solid var(--border-primary);
}

/* ─── Responsive ─── */
@media (max-width: 1024px) {
  .dashboard { padding: 16px; }
  .stats-bar { grid-template-columns: repeat(3, 1fr); }
  .detail-grid { grid-template-columns: repeat(2, 1fr); }
}

@media (max-width: 768px) {
  .stats-bar { grid-template-columns: repeat(2, 1fr); }
  .controls { flex-direction: column; align-items: stretch; }
  .search-box { max-width: none; }
  .col-location, .col-date { display: none; }
  .pipeline { flex-wrap: wrap; }
}
</style>
