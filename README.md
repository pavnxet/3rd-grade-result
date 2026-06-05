# 🏛️ Rajasthan Staff Selection Board (RSSB) Grade 3 Merit List Search

A high-performance, serverless, client-side search engine and analytics dashboard for the Rajasthan Staff Selection Board (RSSB) Grade 3 Teacher (Level 2) results. Designed with a premium dark-themed glassmorphism interface featuring saffron and green accents, this tool allows candidates to quickly search, filter, and view detailed analytical insights from the official merit lists.

---

## 🚀 Key Features

*   **Multi-Subject Support**: Seamlessly toggle between merit lists for different subjects:
    *   🔬 **Science / Maths** (loads `merit_list.csv`)
    *   📖 **Hindi** (loads `3rd_hindi.csv`)
    *   🌍 **SST** (loads `3rd_sst.csv`)
*   **High-Performance Client-Side Parsing**: Powered by [PapaParse](https://www.papaparse.com/), the application streams and processes large CSV datasets (up to 60,000+ rows / 8MB+ per file) directly in the client's browser, eliminating the need for a database server or backend APIs.
*   **Granular Multi-Filter Search**: Locate candidates instantly with support for:
    *   Name & Roll Number
    *   Father's & Mother's Name
    *   Gender & Category (`GEN`, `OBC`, `SC`, `ST`, `EWS`, `MBC`, `SBC`)
    *   TSP / Non-TSP Areas
    *   Numeric Range search for Merit Ranks (e.g., `1-100` or `500`)
    *   Numeric Range search for Marks (e.g., `230-245` or `210.5`)
*   **📊 Advanced Candidate Insights (Analytics Modal)**: Click on any candidate to open an interactive insights panel containing:
    *   **Rank Overview**: Displays global rank and category-specific rank alongside candidate percentile stats.
    *   **Gap Analysis**: Measures marks gap (leads and deficits) between the candidate and the candidates ranked immediately above and below them.
    *   **Category Benchmarking**: Compares candidate marks against category average, highest marks, and lowest marks.
    *   **Global Performance Summary**: Displays candidates ahead/behind stats.
*   **📄 Clickable PDF Scorecard Export**: Save a candidate's detailed report card as a beautifully styled vector PDF. The entire document page acts as an embedded hyperlink—clicking or touching it anywhere redirects the reader back to the live website (`https://pavnxet.github.io/3rd-grade-result/`) for instant online verification.
*   **🔗 URL Hash Routing**: Native SPA routing using hashes (`#science`, `#hindi`, `#sst`). Allows direct sharing, bookmarking, and page refreshes on static hosts (like GitHub Pages) without triggering 404 server errors.
*   **🏠 Header Navigation & Loader Escape**: 
    *   Features a **Home** button in the header bar and clickable header logos/titles to easily return to the main subject selector.
    *   Includes a **Cancel & Go Back** button inside the loading section, allowing users to safely escape from stuck loading states if a network request hangs.
*   **Export Support**: Export filtered query search results back into a clean CSV file (`filtered_results.csv`) with a single click.
*   **Zero-Server Architecture**: Completely static and ready for direct deployment on GitHub Pages or any static host.

---

## 📁 Repository Structure

```filepath
├── index.html            # Primary multi-subject search dashboard with candidate insights modal
├── merit_search.html     # Dedicated search page configured to auto-load Science/Maths results
├── README.md             # Project documentation (this file)
├── counter.json          # Metrics/counter configuration file
├── index.html.backup     # Backup reference of index.html
│
├── [Datasets]
├── 3rd ग्रेड रिजल्ट.pdf     # Official PDF merit list released by RSSB
├── merit_list.csv        # Science / Maths parsed dataset
├── 3rd_hindi.csv         # Hindi parsed dataset
├── 3rd_sst.csv           # SST parsed dataset
└── merit_list.db         # SQLite database version of the results (for local SQL query purposes)
```

---

## 🛠️ Technology Stack

*   **Frontend**: HTML5, Vanilla JavaScript (ES6+), CSS3 (featuring HSL tailored colors, responsive CSS Grid/Flexbox layouts, glassmorphism UI, and keyframe animations).
*   **CSV Parsing**: [PapaParse v5.4.1](https://cdnjs.cloudflare.com/ajax/libs/PapaParse/5.4.1/papaparse.min.js) (configured with custom `transform` functions to trim and clean whitespaces/Windows carriage returns (`\r`) in headers and fields).
*   **Data Storage**: SQLite (`merit_list.db`) for secondary local analytics, CSVs for public-facing application delivery.

---

## 💻 Running the Project Locally

Because the application fetches CSV datasets using standard HTTP request APIs (`fetch`), opening the HTML files directly from your local filesystem (`file://`) will trigger browser CORS restrictions. You must run the application using a local web server.

### Option 1: Using Node.js (Recommended)
You can serve the directory instantly using `serve` or `http-server`:
```bash
# Serve using npx (no install required)
npx serve .
```

### Option 2: Using Python
If you have Python installed, run this in your terminal:
```bash
# For Python 3.x
python -m http.server 8000
```
Then navigate to `http://localhost:8000` in your web browser.

### Option 3: VS Code Live Server
If using VS Code, install the **Live Server** extension, right-click `index.html`, and choose **"Open with Live Server"**.

---

## ⚙️ How it Works Internally

1.  **Parsing & Streaming**: Upon subject selection, the application fetches the appropriate CSV file as a `Blob` and streams it through `Papa.parse`. Clean transformations are applied:
    ```javascript
    Papa.parse(file, {
      header: true,
      skipEmptyLines: true,
      transformHeader: h => h.trim().replace(/\r/g, ''),
      transform: v => (typeof v === 'string' ? v.trim().replace(/\r/g, '') : v),
      // ...
    });
    ```
2.  **State Management & Caching**: Parsed subjects are stored in a local cache object (`loadedSubjects`) memory cache. Switching back to a previously loaded subject restores data instantly without re-fetching or re-parsing the CSV.
3.  **Client-Side Indexing**: Category ranks, percentile values, and gap offsets are computed dynamically on-the-fly when loading candidate insights, keeping memory footprint low and search speeds fast.

---

## 🌐 Deployment

To deploy this project to **GitHub Pages**:
1. Push this repository to GitHub.
2. Go to **Settings** > **Pages** of your repository.
3. Set the build source to **Deploy from a branch** and select the branch (e.g., `main` or `class-3-subject-results-6877b`) and directory (`/root`).
4. Click **Save**. Your site will be live within minutes under `https://<username>.github.io/<repository-name>/`.