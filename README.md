# RF Product PDF Processor - GitHub Repository Setup

## 1. Repository Overview
This repository contains a Node.js/React application designed to extract data from RF (Radio Frequency) product PDFs (specifically frequency hopping switch/filter products), generate standardized product SKUs following the `OSBP-[frequency-code]-[power-value]` naming convention, and output individual PDF datasheets and CSV files for each product. The application supports multi-product PDFs, page segmentation (vertical split for dual-product pages), and user-specified single-product pages to optimize processing accuracy.

## 2. README.md (Full English Documentation)
```markdown
# RF Product PDF Data Extractor & Generator

A robust Node.js/React application that automates data extraction from RF product PDFs (frequency hopping filters/switches), generates standardized SKUs, and outputs individual PDF datasheets and CSV files for each product.

## Key Features
- **Multi-Product PDF Support**: Process entire PDFs containing multiple RF products (supports 45+ page documents).
- **Intelligent Page Segmentation**: 
  - Automatically split pages vertically (center line) for dual-product layouts (left/right isolation).
  - User-specified single-product pages (no segmentation for designated pages).
- **Standardized SKU Generation**:
  - Follows `OSBP-[frequency-code]-[power-value]` naming rule (e.g., `OSBP-003009-0.01W`).
  - Frequency range to code mapping (e.g., 30MHz~90MHz → 003009, 108MHz~400MHz → 01080400).
  - Power value extraction (from "跳频滤波器" prefix in product titles, e.g., "0.01W").
- **Instant Per-Page Processing**:
  - Processes one page at a time (no full-document pre-scan).
  - Generates/downloads PDF/CSV immediately after page/segment analysis.
  - Error resilience (skips faulty products, continues processing).
- **Gemini AI-Powered Extraction**: Direct English data extraction (no secondary translation) for speed/accuracy.

## Prerequisites
- Node.js (v18+ recommended)
- Gemini API Key (access to Gemini 3 Flash Preview model)

## Installation & Local Run
### 1. Clone the Repository
```bash
git clone https://github.com/[your-username]/rf-pdf-processor.git
cd rf-pdf-processor
```

### 2. Install Dependencies
```bash
npm install
```

### 3. Configure Environment Variables
Create a `.env.local` file in the root directory:
```env
GEMINI_API_KEY=your_gemini_api_key_here
```

### 4. Run the Development Server
```bash
npm run dev
```

### 5. Access the App
Open your browser and navigate to `http://localhost:3000`.

## Usage Guide
### 1. Basic Workflow
1. **Upload PDF**: Drag/drop or select the RF product PDF file.
2. **Specify Single-Product Pages (Optional)**:
   - In the "Processing Settings" panel, enter pages that do NOT need segmentation (e.g., `1, 3-5, 10`).
   - Supported formats: single page (1), range (3-5), comma-separated combinations.
3. **Start Processing**: Click "Process PDF" to begin.
4. **Download Files**:
   - For single-product pages: PDF/CSV download triggers immediately after page analysis.
   - For segmented pages: Left/right segments are processed sequentially, with downloads for each segment.

### 2. SKU Naming Rules
| Frequency Range | Corresponding Code | Power Value Source |
|-----------------|--------------------|--------------------|
| 30MHz ~ 90MHz   | 003009             | "跳频滤波器" prefix in title (e.g., 0.01W) |
| 30MHz ~ 512MHz  | 0030512            | Same as above      |
| 108MHz ~ 400MHz | 01080400           | Same as above      |
| 1400MHz ~ 1700MHz | 14001700         | Same as above      |
| 225MHz ~ 375MHz | 02250375           | Same as above      |
| [All other ranges per specs] | [Corresponding code] | Same as above |

Final SKU format: `OSBP-[code]-[power]` (e.g., `OSBP-003009-0.01W`).

## Customization Guide
### 1. Modify SKU Naming Rules
#### Where to Edit: `src/index.tsx` (function: `generateRFProductId`)
```typescript
// Locate the frequency mapping object
const frequencyCodeMap: Record<string, string> = {
  "30-90": "003009",
  "30-512": "0030512",
  // Add/modify frequency range → code pairs here
  "NEW-RANGE": "NEW-CODE", // Example: 500-1000MHz → 05001000
};

// Modify power value cleaning (if needed)
const cleanPowerValue = (power: string) => {
  return power.trim().replace(/\s+/g, "").toUpperCase(); // Adjust regex/formatting here
};
```

### 2. Adjust Page Segmentation Logic
#### Where to Edit: `src/index.tsx` (function: `processPage`)
```typescript
// Modify split ratio (default: 50% vertical split)
const splitPage = async (page: PDFPageProxy) => {
  const viewport = page.getViewport({ scale: 1 });
  const halfWidth = viewport.width / 2; // Change to custom ratio (e.g., 0.4 for 40/60 split)
  
  // Left segment
  const leftContent = await page.getTextContent({
    normalizeWhitespace: true,
    clip: { x: 0, y: 0, width: halfWidth, height: viewport.height },
  });
  
  // Right segment
  const rightContent = await page.getTextContent({
    normalizeWhitespace: true,
    clip: { x: halfWidth, y: 0, width: halfWidth, height: viewport.height },
  });
  
  return { leftContent, rightContent };
};
```

### 3. Modify AI Extraction Prompt
#### Where to Edit: `src/index.tsx` (function: `runSynthesis`)
```typescript
const extractionPrompt = `
  Extract the following data from this RF product page segment (OUTPUT ONLY JSON):
  {
    "productTitle": "Full product title (English)",
    "powerValue": "Power value before '跳频滤波器' (e.g., 0.01W, NO SPACES)",
    "frequencyRange": "Frequency range (e.g., 30-90, 108-400)",
    "specs": [{"key": "Spec Name", "value": "Spec Value"}]
  }
  // Modify prompt to add/remove fields (e.g., add "gain": "XX dB")
`;
```

### 4. Change Download Behavior
#### Where to Edit: `src/index.tsx` (function: `downloadFiles`)
```typescript
// Adjust download delay (default: 0.6s between files)
const DOWNLOAD_DELAY = 600; // Change to 1000 for 1s delay

// Modify CSV columns
const generateCSV = (productData: ProductData) => {
  const csvHeaders = ["SKU", "Product Title", "Frequency Range", "Power Value", "Spec 1", "Spec 2"];
  // Add/remove columns (match with extracted specs)
  const csvRows = [
    [
      productData.sku,
      productData.productTitle,
      productData.frequencyRange,
      productData.powerValue,
      productData.specs.find(s => s.key === "Spec 1")?.value || "",
      // Add new specs here
    ]
  ];
  // Generate CSV content
};
```

### 5. Update Gemini Model
#### Where to Edit: `src/index.tsx` (function: `runSynthesis`)
```typescript
// Change model name (ensure compatibility with generateContent API)
const model = genAI.getGenerativeModel({ model: "gemini-3-flash-preview" });
// Example: switch to gemini-pro → model: "gemini-pro"
```

## Troubleshooting
### Common Errors
1. **404 Error (Gemini Model)**:
   - Ensure model name is correct (use `gemini-3-flash-preview` or valid model from Gemini ListModels API).
   - Verify API key has access to the model.
2. **TypeError: fieldValue.toUpperCase is not a function**:
   - Ensure AI returns string values for power/frequency fields (check extraction prompt).
   - Add type checks: `const safeValue = String(fieldValue || "").toUpperCase();`
3. **No File Downloads**:
   - Check browser download permissions (allow popups/downloads from `localhost:3000`).
   - Verify page segmentation logic (ensure content is detected in segments).
4. **Processing Stops Abruptly**:
   - Check API rate limits (Gemini API may block excessive requests).
   - Increase `DOWNLOAD_DELAY` to reduce request frequency.

### Logging
- Console logs show page processing status, errors, and AI response issues.
- Check browser dev tools (Console tab) for detailed error messages.

## License
[MIT License] (Add your license here)

## Contributing
1. Fork the repository.
2. Create a feature branch (`git checkout -b feature/your-feature`).
3. Commit changes (`git commit -m 'Add your feature'`).
4. Push to the branch (`git push origin feature/your-feature`).
5. Open a Pull Request.
```

## 3. Additional Repository Files
### .env.local.example
```env
# Copy this file to .env.local and replace with your actual API key
GEMINI_API_KEY=your_gemini_api_key_here
```

### .gitignore
```
# Dependencies
node_modules/
.pnp/
.pnp.js

# Environment variables
.env.local
.env.development.local
.env.test.local
.env.production.local

# Next.js
.next/
out/

# Build artifacts
build/
dist/

# Misc
.DS_Store
*.pem
npm-debug.log*
yarn-debug.log*
yarn-error.log*
```

## 4. Repository Structure Recommendation
```
rf-pdf-processor/
├── src/
│   ├── index.tsx          # Core application logic (main file modified in the process)
│   ├── components/        # (Optional) Split UI components (e.g., Uploader.tsx, Settings.tsx)
│   ├── utils/             # (Optional) Helper functions (e.g., skuGenerator.ts, pdfSplitter.ts)
│   └── styles/            # (Optional) CSS/Tailwind styles
├── .env.local.example     # Example env file
├── .gitignore             # Git ignore rules
├── package.json           # Dependencies/scripts
└── README.md              # Main documentation
```

## 5. Key Notes for GitHub Upload
1. **Remove Sensitive Data**: Ensure no hardcoded API keys, personal data, or proprietary PDF content are included.
2. **Dependency Validation**: Verify `package.json` includes all required dependencies (e.g., `@google/generative-ai`, `pdfjs-dist`, `react`, `next`, `tailwindcss`).
3. **Version Tagging**: (Optional) Tag the initial release (e.g., `v1.0.0`) for clarity.
4. **Issue Template**: (Optional) Add an issue template to guide bug reports/feature requests.

This setup provides a complete, professional GitHub repository with clear documentation for users to run, customize, and contribute to the application. The README covers all critical aspects: installation, usage, customization (with specific code references), and troubleshooting—aligned with the iterative development process outlined in your Gemini conversation.
