# 🏛️ Master Blueprint: Building a High-Performance, Client-Side Exam Results Search Engine & Analytics Portal

This document serves as an exhaustive, top-class engineering blueprint, system design guide, and developer instruction manual for AI coding agents to build or modify a custom, serverless, client-side exam results search engine and performance analytics portal.

---

## 📋 Table of Contents
* [0. Base HTML Skeleton](#0-base-html-skeleton)
* [1. Architectural Principles & Core Flow](#1-architectural-principles--core-flow)
* [2. Data Engineering & Raw CSV Cleaning](#2-data-engineering--raw-csv-cleaning)
* [3. CSV Parsing & Header Normalization](#3-csv-parsing--header-normalization)
* [4. Routing, Navigation, & Error Recovery](#4-routing-navigation--error-recovery)
* [5. UI/UX component Wireframe Layouts](#5-uiux-component-wireframe-layouts)
* [6. State Management, Caching, & Search Filtering](#6-state-management-caching--search-filtering)
* [7. Advanced On-the-Fly Analytics & Insights](#7-advanced-on-the-fly-analytics--insights)
* [8. Vector PDF Scorecard Generation & Verification Linking](#8-vector-pdf-scorecard-generation-and-verification-linking)
* [9. Verification, QA, & Boundary Case Checklist](#9-verification-qa--boundary-case-checklist)
* [10. AI Agent Prompt Templates with Performance & Accessibility Constraints](#10-ai-agent-prompt-templates-with-performance--accessibility-constraints)
* [11. Real-Time Visitor and Click Tracking (Client-Side)](#11-real-time-visitor-and-click-tracking-client-side)

---

## 0. Base HTML Skeleton

To ensure developers and AI subagents understand the structural DOM layouts, use the base HTML skeleton below. It establishes the load order of script dependencies and displays all primary functional container IDs.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>RSSB Merit List Search</title>
  
  <!-- CDN Load Order: PapaParse first for data parsing, html2pdf.js second for scorecards -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/PapaParse/5.4.1/papaparse.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/html2pdf.js/0.10.1/html2pdf.bundle.min.js"></script>
  
  <style>
    /* Global Typography, HSL Tokens, and Component CSS (See Section 5) */
  </style>
</head>
<body>
  <!-- Header -->
  <header>
    <div class="emblem">🦚</div>
    <div class="header-text">
      <h1>Rajasthan Staff Selection Board (RSSB)</h1>
      <p id="header-subtitle">Rajasthan Staff Selection Board · Teacher Level 2</p>
    </div>
    <div style="margin-left: auto; display: flex; align-items: center; gap: 12px;">
      <button id="home-btn" style="display: none;" onclick="goBack()">🏠 Home</button>
      <div class="badge" id="total-badge" style="display: none;">0 Records</div>
    </div>
  </header>

  <main class="main">
    <!-- Watermark Card -->
    <div class="watermark-section">
      <div class="watermark-content">
        <span class="watermark-text">Made with <span class="watermark-heart">💖</span> by 
          <a href="https://github.com/pavnxet/3rd-grade-result" class="watermark-link" target="_blank">pavnxet</a>
        </span>
        <div class="watermark-sub">GitHub CSV Search Page</div>
      </div>
    </div>

    <!-- Section 1: Subject Selector Section -->
    <section id="selector-section">
      <!-- Selector buttons generated here (See Section 5.1) -->
    </section>

    <!-- Section 2: Upload / Loader Section with Progress & Error Recovery -->
    <section id="upload-section" style="display: none;">
      <div class="upload-icon">⏳</div>
      <h2 id="loader-title">Loading Merit List...</h2>
      <p id="loader-subject">Fetching data...</p>
      
      <!-- Progress Indicator -->
      <div id="progress-bar-wrap">
        <p id="progress-text">Fetching records…</p>
        <div class="progress-track">
          <div class="progress-fill" id="progress-fill" style="width: 0%;"></div>
        </div>
      </div>
      
      <!-- Action buttons for cancelation & error recovery -->
      <div id="loader-actions" style="margin-top:20px;">
        <button id="retry-btn" style="display: none;" onclick="retryFetch()">🔄 Try Again</button>
        <button class="back-btn" onclick="goBack()">← Cancel & Go Back</button>
      </div>
      <input type="file" id="csv-input" accept=".csv" style="display:none;">
    </section>

    <!-- Section 3: Search Panel & Results Table Grid -->
    <section id="search-section" style="display: none;">
      <!-- Search Filter form grid and results table are housed here (See Section 5.2 / 5.3) -->
    </section>
  </main>

  <script>
    // State management, parser execution, and dynamic routing logic (See sections 2, 3, 4, 6, 7, 8)
  </script>
</body>
</html>
```

---

## 1. Architectural Principles & Core Flow

To eliminate hosting costs, server maintenance, database setup, and scalability issues during high-traffic result releases, this application is built entirely as a **serverless, static Single Page Application (SPA)**. 

### Key Architectural Concepts:
- **Zero Server Setup**: The application runs completely on client-side resources. Data is served as static flat CSV files (which can be large, e.g., 5MB–10MB+ per subject, containing 50,000 to 120,000 rows).
- **Static Hosting Friendly**: Perfect for zero-cost static hosting environments such as **GitHub Pages**, **Vercel**, **Netlify**, or **Cloudflare Pages**.
- **Dynamic Browser Streaming**: The browser fetches the raw CSV text, cleans it of metadata/noise in memory, streams it into JavaScript objects using [PapaParse](https://www.papaparse.com/), and runs multi-column search filters on 100k+ rows in less than 300ms.
- **On-the-Fly Computations**: The system computes percentiles, gender stats, category-specific ranks, and competitor gap analysis on-the-fly without pre-computed databases.

### System Data & Action Flow:
```
[User Action: Select Subject / Load Hash Route]
                       │
                       ▼
          [Check in-memory Cache]
             ├── YES ──> [Render Table & Load UI State]
             └── NO  ──> [Show Interactive Loader & Start Fetch]
                            │
                            ▼
                     [Fetch CSV Blob]
                            │
                            ▼
                  [Run cleanCSVText()]
         (Strips comments, page numbers, repeating headers)
                            │
                            ▼
              [PapaParse Streaming Stream]
         (Normalize headers, strip carriage returns '\r')
                            │
                            ▼
           [Calculate Initial Stats & Cache]
      (Gender ratios, category counts, global candidate count)
                            │
                            ▼
         [Bind UI Event Listeners & Handle Search]
```

---

## 2. Data Engineering & Raw CSV Cleaning

Exam merit lists exported from official PDF documents to CSV format are frequently polluted. They contain metadata, page headers, footer comments, empty lines, and carriage return characters (`\r`). If parsed directly, these anomalies cause the CSV parser to generate malformed columns or skip data rows.

### 2.1 CSV Anomalies to Handle:
1. **Comment Lines**: Lines starting with `#` or metadata comments.
2. **Repeating Page Boundaries**: Repeating table headers on every PDF page transition (e.g. lines starting with `Merit,Marks,Roll No...`).
3. **Empty Rows**: Rows that consist entirely of commas (e.g., `,,,,,,,,,,`).
4. **Metadata Trash**: Lines detailing board information (e.g. `Staff Selection Board`, `Teacher Level 2`, `ANNEXURE-1`, `Page 1 of 400`).
5. **Pre-Header Text**: Extraneous descriptors written before the actual table starts.

### 2.2 JavaScript Cleaning Pipeline:
Before passing the raw text to PapaParse, read the file/blob as plain text (e.g., using `FileReader.readAsText()` or the `.text()` method of a fetch `Response`) and execute the following text cleaning algorithm:

```javascript
/**
 * Cleans raw CSV text by stripping comments, repeating headers, page numbers,
 * and metadata trash, leaving only a single header row followed by clean data.
 * @param {string} text Raw CSV text content
 * @returns {string} Cleaned CSV text content
 */
function cleanCSVText(text) {
  const lines = text.split(/\r?\n/);
  const cleanedLines = [];
  let headerSeen = false;

  for (let i = 0; i < lines.length; i++) {
    const line = lines[i].trim();
    
    // 1. Skip empty lines or comment lines
    if (!line || line.startsWith('#')) {
      continue;
    }

    // 2. Skip lines consisting entirely of commas and whitespace (e.g. ,,,,,,,,,)
    if (/^,+$/.test(line.replace(/\s/g, ''))) {
      continue;
    }

    // 3. Skip PDF-to-CSV metadata noise words
    if (line.includes('ANNEXURE-1') || 
        line.includes('ANNEXURE - 1') ||
        line.includes('Staff Selection Board') || 
        line.includes('TEACHER LEVEL') || 
        line.includes('Merit Wise list') || 
        /^Page\s+\d+/i.test(line.replace(/^,+/,'').trim())) {
      continue;
    }

    // 4. Identify the column header row
    const lowerLine = line.toLowerCase();
    if (lowerLine.includes('merit') && (lowerLine.includes('roll') || lowerLine.includes('candidate'))) {
      if (!headerSeen) {
        headerSeen = true;
        cleanedLines.push(line); // Keep the first occurrence of the header
      }
      continue;
    }

    // 5. Skip any lines occurring before the primary header is identified
    if (!headerSeen) {
      continue;
    }

    // 6. Skip repeating header rows (found at PDF page breaks)
    if (lowerLine.startsWith('merit,marks,') || lowerLine.startsWith('merit rank,marks,')) {
      continue;
    }

    // 7. Retain legitimate data lines
    cleanedLines.push(line);
  }

  return cleanedLines.join('\n');
}
```

---

## 3. CSV Parsing & Header Normalization

Column headers in subject-wise CSV files are rarely uniform. One file may use `Roll no ` (with trailing space), another `RollNo`, and another `Roll`. Similarly, categories may be labeled as `Category` or `Cat`.

To avoid mapping failures and code duplication in UI filters, use PapaParse’s `transformHeader` hook to map all headers to a standardized, camel-case schema.

### 3.1 Standard Unified Schema:
- `Merit` (Numeric overall rank)
- `Marks` (Decimal obtained marks out of maximum)
- `Roll No` (Unique registration or roll number)
- `Candidate Name` (Name of candidate)
- `Father Name` (Father's name)
- `Mother Name` (Mother's name)
- `Gender` (Standardized to `MALE` / `FEMALE`)
- `Category` (Standardized categories: `GEN`, `OBC`, `SC`, `ST`, `EWS`, `MBC`)
- `Sub Category` (Special reservation category, e.g. `LD/CP`, `Ex-Servicemen`)
- `TSP` (Tribal Sub-Plan designation: `YES` / `NO`)

### 3.2 PapaParse Configuration:
```javascript
/**
 * Parses cleaned CSV text using PapaParse with header normalization and progress streaming.
 * @param {string} cleanedText Cleaned CSV text
 * @param {string} subject Subject code (e.g. 'hindi', 'sst')
 */
function parseCleanedCSV(cleanedText, subject) {
  let rowCount = 0;
  const tempArray = [];

  Papa.parse(cleanedText, {
    header: true,
    skipEmptyLines: true,
    worker: false, // Set to true if UI blocks during parsing, otherwise false for faster synchronous loading
    transformHeader: function(h) {
      const clean = h.trim().replace(/\r/g, '').toLowerCase();
      if (clean === 'merit' || clean === 'merit rank' || clean === 'sno') return 'Merit';
      if (clean === 'marks' || clean === 'total marks') return 'Marks';
      if (clean === 'roll no' || clean === 'rollno' || clean === 'roll no.' || clean === 'roll') return 'Roll No';
      if (clean === 'candidate name' || clean === 'name' || clean === 'candidate_name') return 'Candidate Name';
      if (clean === 'father name' || clean === 'fathername' || clean === 'father_name') return 'Father Name';
      if (clean === 'mother name' || clean === 'mothername' || clean === 'mother_name') return 'Mother Name';
      if (clean === 'gender' || clean === 'sex') return 'Gender';
      if (clean === 'category' || clean === 'cat') return 'Category';
      if (clean === 'sub category' || clean === 'subcategory' || clean === 'sub_category' || clean === 'subcat') return 'Sub Category';
      if (clean === 'tsp' || clean === 'tsp_area') return 'TSP';
      return h.trim().replace(/\r/g, ''); // Fallback
    },
    transform: function(value) {
      // Strip carriage returns and trim whitespace from individual values
      if (typeof value === 'string') {
        return value.trim().replace(/\r/g, '');
      }
      return value;
    },
    step: function(result) {
      // Stream parsed row data into our array
      tempArray.push(result.data);
      rowCount++;
      
      // Update the progress bar every 5,000 rows
      if (rowCount % 5000 === 0) {
        const pct = Math.min(95, (rowCount / 60000) * 100);
        document.getElementById('progress-fill').style.width = pct + '%';
        document.getElementById('progress-text').textContent = `Loading… ${rowCount.toLocaleString()} records`;
      }
    },
    complete: function() {
      document.getElementById('progress-fill').style.width = '100%';
      document.getElementById('progress-text').textContent = `✓ ${tempArray.length.toLocaleString()} records loaded!`;
      
      // Cache the parsed array
      loadedSubjects[subject] = tempArray;
      allData = tempArray;

      // Transition to results screen
      setTimeout(() => {
        showResults(subject);
      }, 400);
    }
  });
}
```

---

## 4. Routing, Navigation, & Error Recovery

On static hosting sites (like GitHub Pages), URLs like `/hindi` or `/sst` fail when the page is reloaded because the host tries to serve a directory or file that does not exist, resulting in a **404 Not Found** error. 

To solve this, implement a **Hash-Based Client-Side Router** (`#hindi`, `#sst`, `#science`). The hash stays in the browser and does not get sent to the server. This allows deep-linking and reloads to work seamlessly.

### 4.1 Router Mechanics:
1. Listen to `DOMContentLoaded` and `hashchange` window events.
2. Read the URL hash and clean it.
3. Map the hash to the configuration object.
4. If a valid subject hash is found, load the subject view.
5. If the hash is empty or invalid, return the user to the subject selection home page.

### 4.2 Network Error Recovery and Abort Flow:
When downloading files over 5MB, users on poor connections may experience network timeouts or drops. A robust client-side implementation must:
* **Abort Active Connections**: Use an `AbortController` to cancel any ongoing fetch request when the user cancels or switches views, preventing memory leaks and parallel download queues.
* **Recover Gracefully**: If the fetch fails, update the UI loader to an error state. Display a prominent retry control panel containing a "Try Again" button that re-triggers the subject fetch process and resets the controller.

### 4.3 Navigation & Fetch Controller Implementation:
```javascript
// Data sources mapping
const DATA_SOURCES = {
  science: 'merit_list.csv', 
  hindi: '3rd_hindi.csv',
  sst: '3rd_sst.csv'
};

const SUBJECT_NAMES = {
  science: 'Science / Maths',
  hindi: 'Hindi',
  sst: 'Social Studies (SST)'
};

let currentSubject = null;
let allData = [];
let filteredData = [];
let loadedSubjects = {}; // In-memory cache
let currentFetchController = null; // Instantiated AbortController

// Bind router events
window.addEventListener('DOMContentLoaded', handleHashRoute);
window.addEventListener('hashchange', handleHashRoute);

/**
 * Handles incoming URL hash routing and transitions state.
 */
function handleHashRoute() {
  const hash = window.location.hash.substring(1).toLowerCase();
  
  if (DATA_SOURCES[hash]) {
    if (currentSubject !== hash) {
      selectSubject(hash);
    }
  } else {
    // Navigate home if hash is empty or invalid
    if (document.getElementById('selector-section').style.display === 'none') {
      showSelectorScreen();
    }
  }
}

/**
 * Renders the home screen (Subject Selector grid) and aborts any active fetch.
 */
function showSelectorScreen() {
  // Abort active fetch if the user goes back during a load
  if (currentFetchController) {
    currentFetchController.abort();
    currentFetchController = null;
  }

  document.getElementById('selector-section').style.display = 'block';
  document.getElementById('upload-section').style.display = 'none';
  document.getElementById('search-section').style.display = 'none';
  document.getElementById('total-badge').style.display = 'none';
  
  const homeBtn = document.getElementById('home-btn');
  if (homeBtn) homeBtn.style.display = 'none';
  
  document.getElementById('header-subtitle').textContent = 'Rajasthan Staff Selection Board · Teacher Level 2';
  
  // Clear the window hash safely without triggering page refresh loops
  if (window.location.hash !== '') {
    window.history.replaceState(null, null, ' ');
  }
  
  currentSubject = null;
}

/**
 * Selects a subject, loads its CSV, and handles state transitions.
 * @param {string} subject Key identifier of the subject
 */
function selectSubject(subject) {
  currentSubject = subject;
  
  // Sync Hash to URL
  if (window.location.hash.substring(1).toLowerCase() !== subject) {
    window.location.hash = subject;
  }

  // Update header subtitle dynamically
  document.getElementById('header-subtitle').textContent = `Result Search Portal · ${SUBJECT_NAMES[subject]}`;

  // Check in-memory cache
  if (loadedSubjects[subject]) {
    allData = loadedSubjects[subject];
    showResults(subject);
    return;
  }

  // Set up loader screen state
  showLoader(subject);
  
  // Abort any active fetch before starting a new one
  if (currentFetchController) {
    currentFetchController.abort();
  }
  currentFetchController = new AbortController();
  
  fetch(DATA_SOURCES[subject], { signal: currentFetchController.signal })
    .then(response => {
      if (!response.ok) throw new Error(`HTTP status ${response.status} (${response.statusText})`);
      return response.text();
    })
    .then(text => {
      const cleaned = cleanCSVText(text);
      parseCleanedCSV(cleaned, subject);
    })
    .catch(err => {
      if (err.name === 'AbortError') {
        console.log('Fetch request was aborted by user action.');
        return;
      }
      console.error('Fetch error:', err);
      showLoaderErrorState(err.message || 'Check your internet connection.');
    });
}

/**
 * Shows the Loader screen with initial progress bar states.
 */
function showLoader(subject) {
  const name = SUBJECT_NAMES[subject];
  document.getElementById('loader-title').textContent = `Loading ${name} Results...`;
  document.getElementById('loader-subject').textContent = `Fetching data from ${DATA_SOURCES[subject]}`;
  document.getElementById('progress-fill').style.width = '0%';
  document.getElementById('progress-text').textContent = 'Fetching records…';
  document.querySelector('.upload-icon').textContent = '⏳';
  
  // Toggle visibility (Progress bar visible, Retry button hidden)
  document.getElementById('progress-bar-wrap').style.display = 'block';
  document.getElementById('retry-btn').style.display = 'none';
  
  document.getElementById('selector-section').style.display = 'none';
  document.getElementById('upload-section').style.display = 'block';
  document.getElementById('search-section').style.display = 'none';
}

/**
 * Transitions the loader view to a visible error block with recovery controls.
 */
function showLoaderErrorState(errorMessage) {
  document.querySelector('.upload-icon').textContent = '⚠️';
  document.getElementById('loader-title').textContent = 'Failed to Load Results';
  document.getElementById('loader-subject').textContent = 'The system was unable to download the results file.';
  document.getElementById('progress-text').textContent = `❌ Error: ${errorMessage}`;
  
  // Toggle controls (Hide progress bar, display Retry button)
  document.getElementById('progress-bar-wrap').style.display = 'none';
  document.getElementById('retry-btn').style.display = 'inline-block';
}

/**
 * Invoked by the 'Try Again' button in the loader view to restart the fetch.
 */
function retryFetch() {
  if (currentSubject) {
    selectSubject(currentSubject);
  }
}

/**
 * Triggers selector view when back actions are fired.
 */
function goBack() {
  showSelectorScreen();
}
```

---

## 5. UI/UX Component Wireframe Layouts

The application should have a premium visual design that looks professional and builds trust. Below are wireframe-level HTML snippets detailing exactly how the panels, selectors, tables, and modal elements should be structured.

### 5.1 Subject Selector Card Grid (`#selector-section`)
Displays interactive, large buttons with icons, titles, and target filenames.
```html
<div id="selector-section">
  <h2>Select a Subject to View Results</h2>
  <p>Choose the subject to load its merit list data</p>
  <div class="subject-buttons">
    <button class="subj-btn" onclick="selectSubject('science')">
      <span class="icon">🔬</span>
      <span class="name">Science / Maths</span>
      <span class="file">merit_list.csv</span>
    </button>
    <button class="subj-btn" onclick="selectSubject('hindi')">
      <span class="icon">📖</span>
      <span class="name">Hindi</span>
      <span class="file">3rd_hindi.csv</span>
    </button>
    <button class="subj-btn" onclick="selectSubject('sst')">
      <span class="icon">🌍</span>
      <span class="name">SST</span>
      <span class="file">3rd_sst.csv</span>
    </button>
  </div>
</div>
```

### 5.2 Search Filter Bar Layout
A multi-column grid input layout with selects, range inputs, search metrics, and control buttons.
```html
<div class="search-panel">
  <h3>🔍 Search Filters</h3>
  <div class="fields-grid">
    <div class="field-group">
      <label for="f-name">Candidate Name</label>
      <input type="text" id="f-name" placeholder="e.g. Ramesh or Kumari" oninput="handleSearchInput()">
    </div>
    <div class="field-group">
      <label for="f-roll">Roll Number</label>
      <input type="text" id="f-roll" placeholder="e.g. 493009" oninput="handleSearchInput()">
    </div>
    <div class="field-group">
      <label for="f-father">Father Name</label>
      <input type="text" id="f-father" placeholder="Partial name ok" oninput="handleSearchInput()">
    </div>
    <div class="field-group">
      <label for="f-mother">Mother Name</label>
      <input type="text" id="f-mother" placeholder="Partial name ok" oninput="handleSearchInput()">
    </div>
    <div class="field-group">
      <label for="f-gender">Gender</label>
      <select id="f-gender" onchange="doSearch()">
        <option value="">All</option>
        <option value="MALE">MALE</option>
        <option value="FEMALE">FEMALE</option>
      </select>
    </div>
    <div class="field-group">
      <label for="f-cat">Category</label>
      <select id="f-cat" onchange="doSearch()">
        <option value="">All</option>
        <option value="GEN">GEN</option>
        <option value="OBC">OBC</option>
        <option value="SC">SC</option>
        <option value="ST">ST</option>
        <option value="EWS">EWS</option>
        <option value="MBC">MBC</option>
      </select>
    </div>
    <div class="field-group">
      <label for="f-tsp">TSP Area</label>
      <select id="f-tsp" onchange="doSearch()">
        <option value="">All</option>
        <option value="YES">YES (TSP)</option>
        <option value="__no__">NO (Non-TSP)</option>
      </select>
    </div>
    <div class="field-group">
      <label for="f-merit">Merit Rank Range</label>
      <input type="text" id="f-merit" placeholder="e.g. 1-100 or 500" oninput="handleSearchInput()">
    </div>
    <div class="field-group">
      <label for="f-marks">Marks Range</label>
      <input type="text" id="f-marks" placeholder="e.g. 230-245" oninput="handleSearchInput()">
    </div>
  </div>
  <div class="btn-row">
    <button class="btn-search" onclick="doSearch()">Search</button>
    <button class="btn-clear" onclick="clearAll()">Clear All</button>
    <div class="result-count" id="result-count">Found <span>0</span> results</div>
  </div>
</div>
```

### 5.3 Results Table + Pagination Controls
Scrollable table wrapper with sticky headers, table columns matching our normalized schema, and pagination.
```html
<div class="table-wrap">
  <div class="table-header-bar">
    <h3>Results</h3>
    <button class="export-btn" onclick="exportResults()">⬇ Export Results CSV</button>
  </div>
  <div class="table-scroll">
    <table id="results-table">
      <thead>
        <tr>
          <th>Merit</th>
          <th>Marks</th>
          <th>Roll No</th>
          <th>Candidate Name</th>
          <th>Father Name</th>
          <th>Mother Name</th>
          <th>Gender</th>
          <th>Category</th>
          <th>Sub Cat</th>
          <th>TSP</th>
          <th>Action</th>
        </tr>
      </thead>
      <tbody id="table-body">
        <!-- Rows populated dynamically by renderTable() -->
      </tbody>
    </table>
  </div>
  <div class="pagination" id="pagination">
    <!-- Populated dynamically by renderPagination() -->
  </div>
</div>
```

### 5.4 Candidate Detail Modal Shell
The HTML structure generated on-the-fly or toggled in the DOM when a candidate row is clicked.
```html
<div class="modal-overlay" onclick="if(event.target===this) closeInsight()">
  <div class="modal-content">
    <div class="modal-header">
      <div class="modal-title">📊 Candidate Insights</div>
      <div style="display:flex; align-items:center; gap:10px;">
        <button class="export-btn" id="modal-pdf-btn">📄 Save PDF</button>
        <button class="modal-close" onclick="closeInsight()">×</button>
      </div>
    </div>
    <div class="modal-body">
      <div class="insight-grid">
        <!-- dynamic Insight Cards: Rank Overview, Gap Analysis, Category Benchmark, Quick Stats -->
      </div>
      <div class="nav-candidate">
        <button class="nav-btn" id="prev-candidate-btn">← Previous Candidate</button>
        <button class="nav-btn" id="next-candidate-btn">Next Candidate →</button>
      </div>
    </div>
  </div>
</div>
```

### 5.5 Visual Theme Stylesheet Tokens
Use modern dark-mode surfaces styled with custom CSS variables (themes referencing official RSSB colors like saffron and green):

```css
:root {
  --saffron: #FF6B00;
  --saffron-light: #FF8C38;
  --saffron-dim: rgba(255, 107, 0, 0.12);
  --green: #138808;
  --green-light: #4CAF50;
  --navy: #0A1628;        /* Body background */
  --navy2: #0F1F3D;       /* Card/Header backgrounds */
  --navy3: #162847;       /* Input backgrounds */
  --surface: #111C30;     /* Box Container */
  --surface2: #1A2840;    /* Interactive elements */
  --border: rgba(255, 255, 255, 0.08);
  --text: #E8EDF5;        /* Crisp white text */
  --muted: #7A8FA8;       /* Subtle text gray */
  --white: #ffffff;
}

/* Glassmorphism Panel Utility */
.glass-panel {
  background: rgba(17, 28, 48, 0.7);
  backdrop-filter: blur(12px);
  border: 1px solid var(--border);
}
```

---

## 6. State Management, Caching, & Search Filtering

For datasets exceeding 100k rows, search filters must run efficiently in memory.

### 6.1 State Variable Scope:
```javascript
let loadedSubjects = {};  // Cache object mapping subject keys to parsed arrays
let allData = [];         // Active dataset array (unfiltered)
let filteredData = [];    // Filtered subset array matching active queries
let currentPage = 1;      // Current pagination page
const PAGE_SIZE = 50;     // Records to render per page
```

### 6.2 Search Debouncing:
Text filters must be debounced by **300ms** to prevent rendering delays and UI stuttering on each keystroke.

```javascript
let searchDebounceTimer;
function handleSearchInput() {
  clearTimeout(searchDebounceTimer);
  searchDebounceTimer = setTimeout(doSearch, 300);
}
```

### 6.3 Combined Range & Multi-Column Filtering:
Users must be able to filter by text matches (Roll No, Candidate Name, Father Name, Mother Name), drop-down selects (Gender, Category, TSP), and numeric ranges (Marks range and Merit Rank range).

```javascript
/**
 * Parses range queries like '1-100' or single values like '150'.
 * @param {string} str Input query text
 * @returns {object|null} Object containing {min, max} or null
 */
function parseRange(str) {
  str = str.trim();
  if (!str) return null;
  if (str.includes('-')) {
    const parts = str.split('-');
    return { min: parseFloat(parts[0]) || 0, max: parseFloat(parts[1]) || Infinity };
  }
  const val = parseFloat(str);
  if (!isNaN(val)) return { min: val, max: val };
  return null;
}

/**
 * Filter data in-memory based on values in the UI inputs.
 */
function doSearch() {
  const nameQ = document.getElementById('f-name').value.trim().toUpperCase();
  const rollQ = document.getElementById('f-roll').value.trim();
  const fatherQ = document.getElementById('f-father').value.trim().toUpperCase();
  const motherQ = document.getElementById('f-mother').value.trim().toUpperCase();
  const genderQ = document.getElementById('f-gender').value;
  const categoryQ = document.getElementById('f-cat').value;
  const tspQ = document.getElementById('f-tsp').value;
  
  const meritRange = parseRange(document.getElementById('f-merit').value);
  const marksRange = parseRange(document.getElementById('f-marks').value);

  filteredData = allData.filter(row => {
    if (!row) return false;

    // Fetch normalized keys (with fallback safeguards)
    const name = (row['Candidate Name'] || '').toUpperCase();
    const roll = row['Roll No'] || '';
    const father = (row['Father Name'] || '').toUpperCase();
    const mother = (row['Mother Name'] || '').toUpperCase();
    const gender = (row['Gender'] || '').toUpperCase();
    const category = (row['Category'] || '').toUpperCase();
    const tsp = (row['TSP'] || '').toUpperCase();
    
    const meritVal = parseInt(row.Merit) || 0;
    const marksVal = parseFloat(row.Marks) || 0;

    // Search validation
    if (nameQ && !name.includes(nameQ)) return false;
    if (rollQ && !roll.includes(rollQ)) return false;
    if (fatherQ && !father.includes(fatherQ)) return false;
    if (motherQ && !mother.includes(motherQ)) return false;
    if (genderQ && gender !== genderQ) return false;
    if (categoryQ && category !== categoryQ) return false;
    
    if (tspQ === 'YES' && tsp !== 'YES') return false;
    if (tspQ === '__no__' && tsp === 'YES') return false; // Non-TSP selection

    if (meritRange) {
      if (meritVal < meritRange.min || meritVal > meritRange.max) return false;
    }
    if (marksRange) {
      if (marksVal < marksRange.min || marksVal > marksRange.max) return false;
    }

    return true;
  });

  currentPage = 1;
  renderTable();
  renderPagination();
}
```

---

## 7. Advanced On-the-Fly Analytics & Insights

Clicking a candidate's row opens a details modal. The modal computes and displays contextual analytics on-the-fly.

### 7.1 Percentile Rank
Calculates the candidate's percentile relative to the total candidates in the subject dataset:
$$\text{Percentile} = \left( \frac{\text{Total Candidates} - \text{Candidate Global Merit}}{\text{Total Candidates}} \right) \times 100$$
```javascript
const totalCandidates = allData.length;
const globalMerit = parseInt(candidate.Merit);
const percentile = ((totalCandidates - globalMerit) / totalCandidates * 100).toFixed(1);
```

### 7.2 Category Rank
Determines the candidate's rank within their specific social category (e.g. OBC, EWS):
1. Filter the entire dataset for candidates matching the category.
2. Sort them by marks ascending.
3. Find the candidate's index in this list.
4. Calculate category rank (reversing the list order for descending rank).
```javascript
const category = candidate.Category.toUpperCase();
const sameCatList = allData
  .filter(item => item && (item.Category || '').toUpperCase() === category)
  .sort((a, b) => parseFloat(a.Marks || 0) - parseFloat(b.Marks || 0));

const roll = candidate['Roll No'];
const catPosition = sameCatList.findIndex(item => item['Roll No'] === roll) + 1;
const totalInCat = sameCatList.length;
const categoryRank = totalInCat - catPosition + 1;
```

### 7.3 Gap Analysis (Competitor Gaps)
Find the candidates directly above and below the selected candidate in the search list, and calculate the score differences:
```javascript
const prevCandidate = index > 0 ? filteredData[index - 1] : null;
const nextCandidate = index < filteredData.length - 1 ? filteredData[index + 1] : null;

const gapAbove = prevCandidate ? (parseFloat(prevCandidate.Marks) - parseFloat(candidate.Marks)).toFixed(2) : null;
const gapBelow = nextCandidate ? (parseFloat(candidate.Marks) - parseFloat(nextCandidate.Marks)).toFixed(2) : null;
```

### 7.4 Category Benchmarking
Compare the candidate's score against the average and top marks in their category:
```javascript
const categoryAvg = sameCatList.reduce((sum, item) => sum + parseFloat(item.Marks || 0), 0) / totalInCat;
const categoryTopMarks = parseFloat(sameCatList[sameCatList.length - 1].Marks || 0);
const categoryLowestMarks = parseFloat(sameCatList[0].Marks || 0);
```

---

## 8. Vector PDF Scorecard Generation and Verification Linking

The printable scorecard must look clean, print correctly on paper, and allow results to be verified online.

> [!WARNING]
> **Relying on anchor wrapper tags (`<a>`) for PDF hyperlinks is best-effort and browser-dependent.**
> Many PDF rendering engines, mobile viewers (such as iOS Safari built-in preview, Chrome mobile PDF reader), and standalone desktop applications (like Adobe Acrobat Reader or Apple Preview) strip out bounding-box hyperlink interactions or fail to parse HTML `<a>` tags wrapped around structural grid blocks.
>
> **Fallback Requirement:**
> To ensure the result remains verifiable on paper or inside incompatible viewers, developers must render the validation URL as high-contrast visible text at the bottom of the scorecard. For example:
> `Verification Online: https://pavnxet.github.io/3rd-grade-result/`

### 8.1 PDF Design Rules:
1. **Light Theme**: The generated PDF should use a clean, white background to save ink and look professional. Do not use the app's dark-mode styles in the PDF.
2. **Clickable Verification Area**: Wrap the entire template container in a link tag: `<a href="https://pavnxet.github.io/3rd-grade-result/" target="_blank">...</a>`.
3. **Verification Note**: Display a clear notice at the top of the PDF: `💡 Click anywhere on this document to verify this result online`.
4. **Fallback Text URL**: Print a visible verification URL in high-contrast text at the bottom of the document sheet.

### 8.2 PDF Generation Code:
Include `html2pdf.js` via CDN:
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/html2pdf.js/0.10.1/html2pdf.bundle.min.js"></script>
```

```javascript
/**
 * Renders an off-screen HTML scorecard element, applies light styles,
 * embeds a verification link, and triggers a download using html2pdf.js.
 * @param {number} index Index of candidate in the filteredData array
 */
function downloadCandidatePDF(index) {
  const candidate = filteredData[index];
  if (!candidate) return;

  const subjectName = currentSubject ? SUBJECT_NAMES[currentSubject] : 'Teacher Level 2';
  const name = candidate['Candidate Name'] || 'N/A';
  const roll = candidate['Roll No'] || 'N/A';
  const father = candidate['Father Name'] || 'N/A';
  const mother = candidate['Mother Name'] || 'N/A';
  const marks = candidate['Marks'] || '0';
  const merit = candidate['Merit'] || '0';
  const gender = candidate['Gender'] || 'N/A';
  const category = candidate['Category'] || 'N/A';
  const tsp = candidate['TSP'] === 'YES' ? 'YES (TSP Area)' : 'NO (Non-TSP Area)';
  const subCat = candidate['Sub Category'] || 'None';

  // Perform rank calculations
  const sameCatList = allData.filter(d => (d.Category||'').toUpperCase() === category.toUpperCase()).sort((a,b) => parseFloat(a.Marks||0) - parseFloat(b.Marks||0));
  const catPosition = sameCatList.findIndex(d => d['Roll No'] === roll) + 1;
  const totalInCat = sameCatList.length;
  const catRank = totalInCat - catPosition + 1;
  const totalCandidates = allData.length;
  const percentile = ((totalCandidates - parseInt(merit)) / totalCandidates * 100).toFixed(1);
  const catAvg = (sameCatList.reduce((sum, d) => sum + parseFloat(d.Marks||0), 0) / totalInCat).toFixed(2);
  const catTopMarks = parseFloat(sameCatList[sameCatList.length - 1].Marks || 0).toFixed(2);

  // Create template container element
  const element = document.createElement('div');
  element.style.padding = '40px';
  element.style.fontFamily = "system-ui, -apple-system, sans-serif";
  element.style.color = '#111';
  element.style.background = '#fff';
  element.style.border = '2px solid #FF6B00';
  element.style.borderRadius = '8px';
  element.style.width = '700px';
  element.style.margin = '0 auto';

  // Wrap the content in the verification hyperlink
  element.innerHTML = `
    <a href="https://pavnxet.github.io/3rd-grade-result/" target="_blank" style="text-decoration: none; color: inherit; display: block; cursor: pointer;">
      <!-- Verification Header -->
      <div style="text-align:center; border-bottom:3px double #FF6B00; padding-bottom:15px; margin-bottom:25px;">
        <div style="font-size:10px; color:#666; margin-bottom:8px; font-style:italic;">💡 Click anywhere on this document to verify this result online</div>
        <h2 style="font-size:22px; margin:0; color:#0A1628; text-transform:uppercase;">Rajasthan Staff Selection Board (RSSB)</h2>
        <h3 style="font-size:16px; margin:5px 0; color:#FF6B00;">Candidate Performance Scorecard</h3>
        <p style="font-size:12px; margin:5px 0; color:#666;">Subject: ${subjectName} · Level 2 Exam</p>
      </div>

      <!-- Details -->
      <div style="margin-bottom:25px;">
        <h4 style="border-bottom:1px solid #ddd; padding-bottom:4px; margin-bottom:10px; color:#0A1628; text-transform:uppercase; font-size:11px; letter-spacing:0.5px;">Candidate Information</h4>
        <table style="width:100%; border-collapse:collapse; font-size:13px; line-height: 1.8;">
          <tr>
            <td style="font-weight:bold; width:25%;">Roll Number:</td>
            <td style="width:25%;">${roll}</td>
            <td style="font-weight:bold; width:25%;">Merit Rank:</td>
            <td style="width:25%; color:#FF6B00; font-weight:bold;">#${merit}</td>
          </tr>
          <tr>
            <td style="font-weight:bold;">Candidate Name:</td>
            <td colspan="3">${name}</td>
          </tr>
          <tr>
            <td style="font-weight:bold;">Father's Name:</td>
            <td colspan="3">${father}</td>
          </tr>
          <tr>
            <td style="font-weight:bold;">Mother's Name:</td>
            <td colspan="3">${mother}</td>
          </tr>
          <tr>
            <td style="font-weight:bold;">Gender:</td>
            <td>${gender}</td>
            <td style="font-weight:bold;">Category:</td>
            <td>${category}</td>
          </tr>
          <tr>
            <td style="font-weight:bold;">Sub Category:</td>
            <td>${subCat}</td>
            <td style="font-weight:bold;">TSP Candidate:</td>
            <td>${tsp}</td>
          </tr>
        </table>
      </div>

      <!-- Performance Badging -->
      <div style="display:flex; gap:15px; margin-bottom:25px;">
        <div style="flex:1; border:1px solid #ddd; padding:12px; border-radius:6px; background:#fafafa; text-align:center;">
          <div style="font-size:9px; text-transform:uppercase; color:#666; font-weight:bold;">Obtained Marks</div>
          <div style="font-size:24px; font-weight:bold; color:#138808; margin-top:5px;">${marks}</div>
          <div style="font-size:9px; color:#888; margin-top:2px;">out of 300</div>
        </div>
        <div style="flex:1; border:1px solid #ddd; padding:12px; border-radius:6px; background:#fafafa; text-align:center;">
          <div style="font-size:9px; text-transform:uppercase; color:#666; font-weight:bold;">Category Rank</div>
          <div style="font-size:24px; font-weight:bold; color:#FF6B00; margin-top:5px;">#${catRank}</div>
          <div style="font-size:9px; color:#888; margin-top:2px;">out of ${totalInCat}</div>
        </div>
        <div style="flex:1; border:1px solid #ddd; padding:12px; border-radius:6px; background:#fafafa; text-align:center;">
          <div style="font-size:9px; text-transform:uppercase; color:#666; font-weight:bold;">Overall Percentile</div>
          <div style="font-size:24px; font-weight:bold; color:#0A1628; margin-top:5px;">${percentile}%</div>
          <div style="font-size:9px; color:#888; margin-top:2px;">Top ${(100-percentile).toFixed(1)}%</div>
        </div>
      </div>

      <!-- Benchmark Comparison -->
      <div style="margin-bottom:25px;">
        <h4 style="border-bottom:1px solid #ddd; padding-bottom:4px; margin-bottom:10px; color:#0A1628; text-transform:uppercase; font-size:11px; letter-spacing:0.5px;">Category Comparison (${category})</h4>
        <table style="width:100%; border-collapse:collapse; font-size:12px; text-align:left;">
          <thead>
            <tr style="background:#f5f5f5; border-bottom:1px solid #ddd;">
              <th style="padding:6px 8px;">Metric</th>
              <th style="padding:6px 8px;">Score</th>
              <th style="padding:6px 8px;">Deviation Details</th>
            </tr>
          </thead>
          <tbody>
            <tr style="border-bottom:1px solid #eee;">
              <td style="padding:6px 8px; font-weight:bold;">Category Top Marks:</td>
              <td style="padding:6px 8px; font-weight:bold;">${catTopMarks}</td>
              <td style="padding:6px 8px; color:#F44336;">-${(parseFloat(catTopMarks) - parseFloat(marks)).toFixed(2)} marks behind topper</td>
            </tr>
            <tr style="border-bottom:1px solid #eee;">
              <td style="padding:6px 8px; font-weight:bold;">Candidate Score:</td>
              <td style="padding:6px 8px; font-weight:bold; color:#138808;">${marks}</td>
              <td style="padding:6px 8px; color:#138808; font-weight:bold;">Reference Position</td>
            </tr>
            <tr style="border-bottom:1px solid #eee;">
              <td style="padding:6px 8px; font-weight:bold;">Category Average:</td>
              <td style="padding:6px 8px;">${catAvg}</td>
              <td style="padding:6px 8px; color:${parseFloat(marks) >= parseFloat(catAvg) ? '#138808' : '#F44336'}; font-weight:bold;">
                ${parseFloat(marks) >= parseFloat(catAvg) ? '+' + (parseFloat(marks) - parseFloat(catAvg)).toFixed(2) + ' above avg' : (parseFloat(marks) - parseFloat(catAvg)).toFixed(2) + ' below avg'}
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- Fallback Verification URL & Footer -->
      <div style="margin-top:40px; padding-top:15px; border-top:1px solid #eee; display:flex; justify-content:space-between; font-size:10px; color:#777;">
        <div>
          <span>Generated via RSSB Merit Search Portal</span><br>
          <span style="font-weight: bold; color: #FF6B00;">Verification Online: https://pavnxet.github.io/3rd-grade-result/</span>
        </div>
        <div style="text-align: right;">
          <span>Developer: pavnxet</span><br>
          <span>Date: ${new Date().toLocaleDateString()}</span>
        </div>
      </div>
    </a>
  `;

  const opt = {
    margin:       15,
    filename:     `scorecard_${roll}_${name.replace(/[^a-z0-9]/gi, '_').toLowerCase()}.pdf`,
    image:        { type: 'jpeg', quality: 0.98 },
    html2canvas:  { scale: 2, useCORS: true, logging: false },
    jsPDF:        { unit: 'mm', format: 'a4', orientation: 'portrait' }
  };

  html2pdf().set(opt).from(element).save();
}
```

---

## 9. Verification, QA, & Boundary Case Checklist

AI Agents working on this system must verify all checklist items before completing changes:

- [ ] **No Page Hangs on First Load**: Fetching and parsing huge CSV files (8MB+) must happen smoothly with visual loader feedback and cancellation controls.
- [ ] **Dynamic Subtitle & Title Sync**: The header subtitle matches the active subject (e.g., `Science / Maths`, `Hindi`, `SST`) and resets to `Teacher Level 2` on the selector page.
- [ ] **Navigation Check**: Clicking the header emblem, header text, or the header "Home" button successfully takes the user back to the selector page.
- [ ] **Cache Hits Validation**: Switching back to a previously loaded subject loads instantly from memory with no secondary HTTP fetches.
- [ ] **Hyperlink Verification in PDF**: The downloaded PDF's entire page is clickable and opens the online results validation site.
- [ ] **Empty State Boundaries**: Invalid filter inputs render a clean empty search state table rather than throwing a JavaScript crash error.

---

## 10. AI Agent Prompt Templates with Performance & Accessibility Constraints

Use the following highly specialized prompt templates when delegating tasks to subagents or initializing modules.

### 10.1 Template: CSV Cleansing Module
```markdown
You are a Data Engineering Agent. Implement a JavaScript CSV text-cleansing pipeline that takes a raw, polluted CSV string and outputs a cleaned CSV string.
Requirements & Constraints:
1. Ignore empty lines, comments (lines starting with '#'), and lines consisting only of commas.
2. Ignore repeating headers (headers repeated on PDF page borders containing 'Merit,Marks,Roll').
3. Keep only the first header occurrence.
4. Filter out lines containing board-specific metadata noise ('Staff Selection Board', 'TEACHER LEVEL', 'ANNEXURE-1', 'Page X of Y').
5. Ensure data rows begin with a digit representing the Merit Rank.
6. Performance Budget: Must process a 10MB CSV file in under 150ms on a standard low-end mobile device. Avoid inefficient nested loops or regex patterns susceptible to catastrophic backtracking.
7. Encoding Support: Gracefully handle UTF-8 with Byte Order Mark (BOM) signatures and non-ASCII character encoding.
Include complete unit tests validating your logic against garbage headers, carriage return variations (\r\n vs \n), and empty list arrays.
```

### 10.2 Template: Routing, Abort Controller, & Error UI
```markdown
You are a Core Routing Agent. Implement a Hash-Based Client-Side SPA Router in JavaScript.
Requirements & Constraints:
1. Map URL hashes (#science, #hindi, #sst) to subject keys.
2. Intercept 'DOMContentLoaded' and 'hashchange' events.
3. Show the Subject Selection page if the hash is empty/invalid.
4. Implement an active loading transition overlay with a Cancel button when fetching a subject file.
5. Error Recovery & Aborting: If a network fetch fails, transition the loader panel to a visible error state displaying the failure reason. Provide a "Try Again" button to re-trigger the download. Reset and invoke AbortController to cleanly abort fetch requests when users cancel or navigate away.
6. Accessibility Requirements: Wrap loader alerts and error states in semantic live-regions (role="alert" or aria-live="assertive"). Ensure keyboard accessibility is maintained: focus must trap within the error/loader box and allow full keyboard operability.
7. Mobile Breakpoints: All navigation buttons must have a minimum touch-target size of 44x44px. The loader box must scale responsively down to 320px viewports without horizontal clipping.
```

### 10.3 Template: Dynamic PDF Scorecard Generator
```markdown
You are a UI-to-PDF Conversion Agent. Implement the candidate scorecard export function using html2pdf.js.
Requirements & Constraints:
1. Generate an HTML node dynamically in memory containing candidate metrics (Global Rank, Category Rank, Percentile, Category Averages, Marks).
2. Apply clean light-themed styling suited for printed media (no dark background, clear padding, professional borders).
3. Wrap the scorecard block inside an anchor tag pointing to the online verification dashboard 'https://pavnxet.github.io/3rd-grade-result/'.
4. Fallback URL requirement: Display a high-contrast, human-readable textual verification URL ('Verification Online: https://pavnxet.github.io/3rd-grade-result/') at the bottom of the scorecard. This ensures verifiability if viewers or printouts strip the anchor bounding box.
5. Rendering Budget: PDF rendering must execute in under 2 seconds. Clean up DOM nodes immediately after download triggers to prevent browser memory leaks.
6. Print Formatting: Constrain the layout to fit cleanly on a single A4 page without vertical page overflows. Use print-safe color values ensuring a minimum WCAG AA contrast ratio of 4.5:1.
7. Vector Output quality: Configure html2canvas to render at a minimum scale of 2 to support crisp, vector-grade printing.
```

---

## 11. Real-Time Visitor and Click Tracking (Client-Side)

To track usage metrics without incurring backend server costs, database maintenance, or storage fees, the application integrates a client-side click/view tracking system. It leverages a free, serverless, and privacy-friendly key-value API (`countapi.mileshilliard.com`).

### 11.1 Tracking Strategy
1. **Subject-Specific Keying**: Each subject has its own unique count key in the format `pavnxet-3rd-grade-result-{subject}`.
2. **Dynamic UI Badging**: Counts are displayed as small rounded badges (e.g. `👁️ 1,234`) inside each subject selection button.
3. **On-Load Retrieval**: When the home selector screen is rendered, the application makes parallel async GET requests to retrieve the current counts for all subjects using `/api/v1/get/`.
4. **Triggered Increments**: When a subject is selected (either by clicking the subject button or by navigating directly via a URL hash route like `#hindi`), the application triggers an increment GET request to `/api/v1/hit/` and updates the respective UI badge in real-time.
5. **Direct View Tracking**: For standalone views like `merit_search.html` (which skips the selector screen and auto-loads Science/Maths), a silent fetch hits the Science API endpoint on `DOMContentLoaded` to capture direct arrivals.

### 11.2 Key Code Implementation

#### CSS Styling
```css
.subj-btn .views {
  font-size: 11px;
  color: var(--saffron);
  background: rgba(255,107,0,0.1);
  padding: 2px 8px;
  border-radius: 20px;
  font-weight: 500;
  margin-top: 4px;
  display: inline-flex;
  align-items: center;
  gap: 4px;
  border: 1px solid rgba(255,107,0,0.2);
  transition: all 0.2s;
}
```

#### HTML Placement
```html
<button class="subj-btn" onclick="selectSubject('science')">
  <span class="icon">🔬</span>
  <span class="name">Science / Maths</span>
  <span class="file">merit_list.csv</span>
  <span class="views" id="count-science" style="display: none;">👁️ --</span>
</button>
```

#### JavaScript Logic
```javascript
// Load view counts for all subjects
function loadViewCounts() {
  const subjects = ['science', 'hindi', 'sst'];
  subjects.forEach(subject => {
    fetch(`https://countapi.mileshilliard.com/api/v1/get/pavnxet-3rd-grade-result-${subject}`)
      .then(response => {
        if (!response.ok) throw new Error('API unreachable');
        return response.json();
      })
      .then(data => {
        const el = document.getElementById(`count-${subject}`);
        if (el && typeof data.value !== 'undefined') {
          el.textContent = `👁️ ${data.value.toLocaleString()}`;
          el.style.display = 'inline-flex';
        }
      })
      .catch(err => {
        console.warn(`Failed to fetch count for ${subject}:`, err);
      });
  });
}

// Increment view count for a subject
function incrementSubjectCount(subject) {
  fetch(`https://countapi.mileshilliard.com/api/v1/hit/pavnxet-3rd-grade-result-${subject}`)
    .then(response => {
      if (!response.ok) throw new Error('API unreachable');
      return response.json();
    })
    .then(data => {
      const el = document.getElementById(`count-${subject}`);
      if (el && typeof data.value !== 'undefined') {
        el.textContent = `👁️ ${data.value.toLocaleString()}`;
        el.style.display = 'inline-flex';
      }
    })
    .catch(err => {
      console.warn(`Failed to increment count for ${subject}:`, err);
    });
}
```

### 11.3 Fallback and Ad-Blocker Safeguards
Since these requests hit an external third-party domain, they may be blocked by user privacy extensions or ad-blockers (e.g. uBlock Origin). To prevent JavaScript runtime crashes:
* Wrap fetch requests in promise-chains with robust `.catch()` blocks.
* Keep counter placeholders set to `display: none` by default; only toggle them to `inline-flex` once a successful JSON payload with a valid `value` property is received.
* Ensure key UI elements (like progress loader text or CSV file selectors) are not bound to or dependent on the success of these counting endpoints.
