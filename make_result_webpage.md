# 🏛️ Master Blueprint: Building a High-Performance, Client-Side Exam Results Search Engine & Analytics Portal

This document serves as an exhaustive, top-class engineering blueprint, system design guide, and developer instruction manual for AI coding agents to build or modify a custom, serverless, client-side exam results search engine and performance analytics portal.

---

## 📋 Table of Contents
1. [Architectural Principles & Core Flow](#1-architectural-principles--core-flow)
2. [Data Engineering & Raw CSV Cleaning](#2-data-engineering--raw-csv-cleaning)
3. [CSV Parsing & Header Normalization](#3-csv-parsing--header-normalization)
4. [Routing & Navigation (Hash-Based SPA)](#4-routing--navigation-hash-based-spa)
5. [UI/UX & Interactive Design System](#5-uiux--interactive-design-system)
6. [State Management, Caching, & Search Filtering](#6-state-management-caching--search-filtering)
7. [Advanced On-the-Fly Analytics & Insights](#7-advanced-on-the-fly-analytics--insights)
8. [Vector PDF Scorecard Generation & Verification Linking](#8-vector-pdf-scorecard-generation--verification-linking)
9. [Verification, QA, & Boundary Case Checklist](#9-verification-qa--boundary-case-checklist)
10. [AI Agent Prompt Templates for Subagent Delegation](#10-ai-agent-prompt-templates-for-subagent-delegation)

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

## 4. Routing & Navigation (Hash-Based SPA)

On static hosting sites (like GitHub Pages), URLs like `/hindi` or `/sst` fail when the page is reloaded because the host tries to serve a directory or file that does not exist, resulting in a **404 Not Found** error. 

To solve this, implement a **Hash-Based Client-Side Router** (`#hindi`, `#sst`, `#science`). The hash stays in the browser and does not get sent to the server. This allows deep-linking and reloads to work seamlessly.

### 4.1 Router Mechanics:
1. Listen to `DOMContentLoaded` and `hashchange` window events.
2. Read the URL hash and clean it.
3. Map the hash to the configuration object.
4. If a valid subject hash is found, load the subject view.
5. If the hash is empty or invalid, return the user to the subject selection home page.

### 4.2 Router Implementation:
```javascript
// Data sources mapping
const DATA_SOURCES = {
  science: '3rd_science.csv', // Update with actual paths
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
 * Renders the home screen (Subject Selector grid)
 */
function showSelectorScreen() {
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

  // Trigger Fetch workflow
  showLoader(subject);
  
  // Implement a fetch abort controller for download cancellation
  window.currentFetchController = new AbortController();
  
  fetch(DATA_SOURCES[subject], { signal: window.currentFetchController.signal })
    .then(response => {
      if (!response.ok) throw new Error('Network response error');
      return response.text();
    })
    .then(text => {
      const cleaned = cleanCSVText(text);
      parseCleanedCSV(cleaned, subject);
    })
    .catch(err => {
      if (err.name === 'AbortError') {
        console.log('Fetch aborted by user.');
      } else {
        document.getElementById('progress-text').textContent = '❌ Failed to load dataset.';
        console.error(err);
      }
    });
}
```

---

## 5. UI/UX & Interactive Design System

The application should have a premium visual design that looks professional and builds trust.

### 5.1 Color Palette & Theme Tokens
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
```

### 5.2 Layout Requirements:
1. **Typography**: Use high-legibility Google Fonts. For numeric data and titles, use **Rajdhani** (sans-serif, bold letter spacing). For detailed candidate cards and data tables, use **Noto Sans**.
2. **Glassmorphism Panels**: Apply thin border accents (`1px solid var(--border)`) and subtle background blur:
   ```css
   .glass-panel {
     background: rgba(17, 28, 48, 0.7);
     backdrop-filter: blur(12px);
     border: 1px solid var(--border);
   }
   ```
3. **Responsive Grid Layout**: Search filters must display in a multi-column responsive grid (1 column on mobile, 2 on tablets, 3 or 4 on desktop screens).
4. **Interactive States**: Hover effects on tables, cards, and buttons should use smooth transitions (`transition: all 0.2s ease`).

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

## 8. Vector PDF Scorecard Generation & Verification Linking

The printable scorecard must look clean, print correctly on paper, and allow results to be verified online.

### 8.1 PDF Design Rules:
1. **Light Theme**: The generated PDF should use a clean, white background to save ink and look professional. Do not use the app's dark-mode styles in the PDF.
2. **Clickable Verification Area**: Wrap the entire template container in a link tag: `<a href="https://pavnxet.github.io/3rd-grade-result/" target="_blank">...</a>`. Modern PDF renderers convert this link wrapper into a clickable area across the whole page. Anyone viewing the PDF can click it to visit the online verification site.
3. **Verification Note**: Display a clear notice at the top of the PDF: `💡 Click anywhere on this document to verify this result online`.

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

      <!-- Verification Footer -->
      <div style="margin-top:40px; padding-top:15px; border-top:1px solid #eee; display:flex; justify-content:space-between; font-size:10px; color:#777;">
        <div>Generated via RSSB Merit Search Portal · pavnxet</div>
        <div>Date: ${new Date().toLocaleDateString()}</div>
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

## 10. AI Agent Prompt Templates for Subagent Delegation

Use the following detailed prompt templates when delegating tasks to subagents or initializing new sub-modules.

### 10.1 Template: CSV Cleansing Module
```markdown
You are a Data Engineering Agent. Implement a JavaScript CSV text-cleansing pipeline that takes a raw, polluted CSV string and outputs a cleaned CSV string.
Requirements:
1. Ignore empty lines, comments (lines starting with '#'), and lines consisting only of commas.
2. Ignore repeating headers (headers repeated on PDF page borders containing 'Merit,Marks,Roll').
3. Keep only the first header occurrence.
4. Filter out lines containing board-specific metadata noise ('Staff Selection Board', 'TEACHER LEVEL', 'ANNEXURE-1', 'Page X of Y').
5. Ensure data rows begin with a digit representing the Merit Rank.
Include complete test cases validating your logic against raw lines containing these boundary conditions.
```

### 10.2 Template: SPA Router & Loader Cancellation
```markdown
You are a Core Routing Agent. Implement a Hash-Based Client-Side SPA Router in JavaScript.
Requirements:
1. Map URL hashes (#science, #hindi, #sst) to subject keys.
2. Intercept 'DOMContentLoaded' and 'hashchange' events.
3. Show the Subject Selection page if the hash is empty/invalid.
4. Implement an active loading transition overlay with a cancel button when fetching a subject file.
5. In case of cancelation, abort the fetch request using AbortController, clear the loading state, and return to the subject selector home view.
```

### 10.3 Template: dynamic Scorecard PDF Export
```markdown
You are a UI-to-PDF Conversion Agent. Implement the candidate scorecard export function using html2pdf.js.
Requirements:
1. Generate an HTML node dynamically in memory containing candidate metrics (Global Rank, Category Rank, Percentile, Category Averages, Marks).
2. Apply clean light-themed styling suited for printed media (no dark background, clear padding, professional borders).
3. Wrap the entire scorecard wrapper element in an anchor tag pointing to the online verification dashboard 'https://pavnxet.github.io/3rd-grade-result/' to create an active verification hyperlink in the output PDF.
4. Download the document using the filename format: 'scorecard_[RollNo]_[CandidateName].pdf'.
```
