# Comprehensive Analysis of org-roam-ui

**Repository**: https://github.com/org-roam/org-roam-ui

**Research Date**: 2025-11-04

---

## Executive Summary

**org-roam-ui** is a graphical frontend for visualizing and interacting with org-roam notes. It's a web-based application that runs alongside Emacs, providing an interactive force-directed graph visualization of your knowledge base. The application consists of two main components:

1. **Emacs Lisp backend** (`org-roam-ui.el`) - Serves the web application and manages WebSocket communication
2. **Next.js frontend** - Interactive React-based web interface for graph visualization and note preview

The application enables real-time synchronization between Emacs and the browser, allowing users to navigate their org-roam network visually while maintaining their Emacs workflow.

---

## Technology Stack

### Frontend
- **Framework**: Next.js 11 (React 17)
- **Language**: TypeScript
- **UI Library**: Chakra UI (component library)
- **Graph Visualization**:
  - `react-force-graph` (2D and 3D graph rendering)
  - `d3-force-3d` (force simulation)
  - Three.js (for 3D mode via SpriteText)
- **Real-time Communication**: ReconnectingWebSocket
- **Document Processing**:
  - `uniorg` - Org-mode parser
  - `unified` ecosystem - Document processing pipeline
  - `rehype-katex` - LaTeX math rendering
  - `remark` plugins - Markdown support
- **State Management**: React hooks, custom persistent state
- **Animation**: Framer Motion, es6-tween
- **Graph Algorithms**: jLouvain.js (community detection)

### Backend (Emacs)
- **Language**: Emacs Lisp
- **Dependencies**:
  - `org-roam` (v2.0.0+)
  - `simple-httpd` - HTTP server
  - `websocket` - WebSocket server
  - `f` - File utilities
- **Requirements**: Emacs >= 27.1 (for fast JSON parsing)

### Build & Development
- **Package Manager**: Yarn/NPM
- **Build Tool**: Next.js build system
- **Transpilation**: next-transpile-modules (for D3 packages)
- **CI/CD**: GitHub Actions
- **Code Quality**: ESLint, Prettier, TypeScript

---

## Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         EMACS                                │
│  ┌────────────────────────────────────────────────────┐     │
│  │         org-roam-ui.el (Elisp Backend)             │     │
│  │  • HTTP Server (simple-httpd) :35901               │     │
│  │  • WebSocket Server :35903                         │     │
│  │  • org-roam Database Query                         │     │
│  │  • Theme Extraction & Sync                         │     │
│  │  • Node/Link Data Provider                         │     │
│  └────────────────────────────────────────────────────┘     │
│                          ↕                                   │
│                    WebSocket + HTTP                          │
└──────────────────────────────────────────────────────────────┘
                           ↕
┌─────────────────────────────────────────────────────────────┐
│              BROWSER (localhost:35901)                       │
│  ┌────────────────────────────────────────────────────┐     │
│  │        Next.js Application (Frontend)              │     │
│  │                                                     │     │
│  │  ┌──────────────┐  ┌──────────────┐               │     │
│  │  │    Graph     │  │   Sidebar    │               │     │
│  │  │              │  │              │               │     │
│  │  │ • 2D/3D viz  │  │ • Note view  │               │     │
│  │  │ • Filtering  │  │ • Preview    │               │     │
│  │  │ • Navigation │  │ • Backlinks  │               │     │
│  │  └──────────────┘  └──────────────┘               │     │
│  │                                                     │     │
│  │  ┌──────────────────────────────────────────────┐  │     │
│  │  │           Tweaks Panel                       │  │     │
│  │  │  • Physics    • Filters   • Visuals          │  │     │
│  │  └──────────────────────────────────────────────┘  │     │
│  └────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

### Communication Protocol

**WebSocket Messages (Emacs → Browser):**
```typescript
{
  type: "graphdata" | "variables" | "theme" | "command" | "orgText",
  data: {
    // graphdata: { nodes, links, tags }
    // variables: { roamDir, dailyDir, attachDir, subDirs, katexMacros }
    // theme: { bg, fg, colors... }
    // command: { commandName: "follow" | "zoom" | "local" | "change-local-graph", id, ... }
  }
}
```

**WebSocket Messages (Browser → Emacs):**
```typescript
{
  command: "open" | "delete" | "create" | "getText",
  data: {
    id: string,          // Node ID
    title?: string,      // For create
    file?: string,       // For delete
    ref?: string        // For references
  }
}
```

---

## Key Components and Their Responsibilities

### 1. **Emacs Backend (`org-roam-ui.el`)**

**Core Responsibilities**:
- Start HTTP server on port 35901 to serve the web application
- Start WebSocket server on port 35903 for real-time communication
- Query org-roam database for nodes, links, and tags
- Send graph data to the frontend
- Handle commands from frontend (open node, delete, create)
- Sync Emacs theme to the web interface
- Provide HTTP endpoints for node content and images
- Track current node in Emacs and send updates

**Key Functions**:
- `org-roam-ui-mode` - Global minor mode to enable/disable ORUI
- `org-roam-ui--send-graphdata` - Query database and send graph structure
- `org-roam-ui--ws-on-message` - Handle incoming WebSocket commands
- `org-roam-ui--on-msg-open-node` - Open a node in Emacs buffer
- `org-roam-ui-follow-mode` - Track cursor position in Emacs
- `org-roam-ui--update-theme` - Extract and send theme colors

**HTTP Servlets**:
- `/node/:id` - Returns node content as text
- `/img/:file` - Serves images from org files

### 2. **Main Application Page (`pages/index.tsx`)**

**Responsibilities**:
- Application entry point and state management
- WebSocket connection initialization
- Graph data processing and transformation
- State management for:
  - Graph visualization (physics, visuals, filters)
  - Node selection and scope (global vs local view)
  - User interactions (mouse, behavior)
  - Theme and colors
- Generate synthetic links for heading hierarchy
- Handle node/link filtering based on user preferences

**Key State Variables**:
- `graphData` - Complete graph structure
- `scope` - Current view (global or local nodes)
- `filter` - Active filters (tags, directories, orphans, etc.)
- `physics` - Force simulation parameters
- `visuals` - Visual appearance settings
- `behavior` - Interaction behavior
- `emacsNodeId` - Currently focused node in Emacs

### 3. **Graph Component**

**Responsibilities**:
- Render 2D or 3D force-directed graph
- Handle node/link visualization
- Manage node colors based on:
  - Community detection (Louvain algorithm)
  - Degree centrality
  - Custom tag colors
- Display labels with customizable appearance
- Handle user interactions (click, hover, drag)
- Animate transitions and highlights
- Apply physics forces (charge, gravity, collision, centering)

**Visualization Features**:
- Directional particles on links
- Directional arrows
- Customizable node sizes based on connections
- Link color schemes (by type: cite, ref, id)
- Highlight animations
- Label rendering with various display modes

### 4. **Sidebar Component (`components/Sidebar/`)**

**Responsibilities**:
- Display note preview/content
- Show backlinks to current node
- Render org-mode or markdown content
- Handle internal links between nodes
- Display note metadata and tags
- Navigation history (undo/redo)
- Collapsible sections for long notes

**Sub-components**:
- `Note.tsx` - Main note renderer
- `Link.tsx` - Internal link handling
- `Backlinks.tsx` - Show nodes linking to current
- `TagBar.tsx` - Display and filter by tags
- `Toolbar.tsx` - Navigation controls
- `OrgImage.tsx` - Image display
- `Section.tsx` - Collapsible sections

### 5. **Tweaks Panel (`components/Tweaks/`)**

**Responsibilities**:
- User interface for customization
- Physics simulation controls
- Visual appearance settings
- Filter configuration
- Behavior customization
- Color scheme management

**Sub-sections**:
- **Physics**: Force parameters, gravity, collision, centering
- **Filter**: Tags, directories, orphans, dailies, citations
- **Visual**: Colors, labels, particles, arrows, animations
- **Behavior**: Follow mode, local graph behavior, zoom settings
- **NodesNLinks**: Node/link appearance and coloring

### 6. **Document Processing (`util/processOrg.tsx`)**

**Responsibilities**:
- Parse org-mode files using `uniorg`
- Parse markdown files using `remark`
- Convert to React components using `rehype-react`
- Handle LaTeX math rendering with KaTeX
- Process internal links (org-id, wiki-links)
- Handle attachments
- Apply custom components for rendering

**Processing Pipeline**:
```
Org/MD → Parse → Extract Keywords → Process Attachments
  → Convert to HTML → Render Math → Convert to React → Display
```

### 7. **Configuration (`components/config.ts`)**

**Responsibilities**:
- Define default settings for all aspects
- Export initial state for:
  - Physics parameters
  - Filter settings
  - Visual appearance
  - Behavior configuration
  - Mouse interactions
  - Local graph settings
  - Color schemes

### 8. **Utility Functions (`util/`)**

**Key Utilities**:
- `webSocketFunctions.ts` - WebSocket communication helpers
- `persistant-state.ts` - Local storage state management
- `getNodeColor.ts` - Node color calculation
- `getLinkColor.ts` - Link color calculation
- `nodeSize.ts` - Calculate node sizes
- `findNthNeighbour.ts` - Find neighbors for local graph
- `themecontext.tsx` - Theme management context
- `variablesContext.tsx` - Emacs variables context

---

## Integration Points with org-roam

### 1. **Database Queries**

The Elisp backend queries the org-roam SQLite database directly:

```elisp
;; Get all nodes with tags
(org-roam-db-query [:select [id file title level pos olp properties tags]
                     :from nodes
                     :left-join tags :on (= id node_id)
                     :group :by id])

;; Get all links
(org-roam-db-query [:select [source dest type]
                     :from links
                     :where (= type "id")])

;; Get citations (v2 schema)
(org-roam-db-query [:select [node-id cite-key]
                     :from citations
                     :left :outer :join refs :on (= cite-key ref)])
```

### 2. **Data Structures**

**OrgRoamNode**:
```typescript
{
  id: string,           // Org-roam node ID (UUID)
  file: string,         // Absolute file path
  title: string,        // Node title
  level: number,        // Heading level (0 = file node)
  pos: number,          // Position in file
  olp: string[],        // Outline path
  properties: {},       // Node properties (ROAM_REFS, etc.)
  tags: string[]       // Node tags
}
```

**OrgRoamLink**:
```typescript
{
  source: string,       // Source node ID
  target: string,       // Target node ID
  type: string         // "id", "cite", "ref", "heading", "parent"
}
```

### 3. **Real-time Synchronization**

- **Follow Mode**: When enabled, ORUI tracks your position in Emacs
  - Uses `post-command-hook` to detect node changes
  - Sends `follow` command with current node ID
  - Browser centers/zooms to the active node

- **Update on Save**: When saving org-roam buffers
  - Triggers database sync (`org-roam-db-sync`)
  - Sends updated graph data to browser
  - Graph updates in real-time

- **Theme Sync**: Extracts Emacs theme colors
  - Supports doom-themes automatically
  - Falls back to face attributes
  - Sends color scheme to browser

### 4. **Bidirectional Commands**

**Browser → Emacs**:
- Double-click node → Opens in Emacs buffer
- Delete command → Deletes file and syncs database
- Create command → Opens org-roam capture

**Emacs → Browser**:
- `(org-roam-ui-node-zoom)` → Zoom to node in graph
- `(org-roam-ui-node-local)` → Open local graph view
- `(org-roam-ui-sync-theme)` → Update theme

---

## Development Setup Instructions

### Prerequisites
- Emacs >= 27.1
- Node.js >= 16
- Yarn or NPM
- org-roam v2 installed and configured

### Setup Steps

1. **Clone the repository**:
```bash
git clone https://github.com/org-roam/org-roam-ui
cd org-roam-ui
```

2. **Install dependencies**:
```bash
yarn install
# or
npm install
```

3. **Development mode** (with hot reload):
```bash
yarn dev
# Server runs on http://localhost:3000
```

4. **Build for production**:
```bash
yarn build    # Creates optimized build
yarn export   # Exports static files to ./out/
```

5. **Install Emacs package**:

Via MELPA:
```elisp
M-x package-install RET org-roam-ui RET
```

Via straight.el:
```elisp
(use-package org-roam-ui
  :straight (:host github :repo "org-roam/org-roam-ui"
             :branch "main" :files ("*.el" "out"))
  :after org-roam
  :config
  (setq org-roam-ui-sync-theme t
        org-roam-ui-follow t
        org-roam-ui-update-on-save t
        org-roam-ui-open-on-start t))
```

6. **Start ORUI**:
```elisp
M-x org-roam-ui-mode RET
```

The application will:
- Start HTTP server on port 35901
- Start WebSocket server on port 35903
- Open your default browser to http://localhost:35901

---

## Key Findings and Architectural Decisions

### 1. **Dual-Component Architecture**
- **Rationale**: Emacs has direct access to org-roam database; browser provides modern visualization
- **Benefit**: Leverages strengths of both platforms
- **Trade-off**: Requires two components to be running simultaneously

### 2. **Pre-built Static Files Included**
- The `/out/` directory contains pre-built Next.js application
- Users don't need Node.js to run ORUI, only to develop it
- Simplifies installation via MELPA

### 3. **Force-Directed Graph**
- Uses physics simulation (d3-force-3d) for organic layout
- Highly customizable force parameters
- Can be computationally expensive for large graphs (>2000 nodes)

### 4. **Link Type Enrichment**
- Automatically creates synthetic links for heading hierarchies
- Distinguishes between file-level and heading-level links
- Supports multiple link types: id, cite, ref, heading, parent

### 5. **Local vs Global Views**
- **Global**: Shows entire graph
- **Local**: Shows selected nodes and N-th neighbors
- Smooth transitions between views
- Supports adding/removing nodes dynamically

### 6. **Org-mode Parsing**
- Uses `uniorg` library for org-mode parsing in JavaScript
- Maintains compatibility with org-mode features
- Supports both org and markdown files (via md-roam)

### 7. **Theme Synchronization**
- Automatically extracts Emacs theme colors
- Works best with doom-themes
- Fallback to face attributes for other themes
- Users can define custom themes

### 8. **Performance Optimizations**
- Conditional rendering based on zoom level
- Label display only when needed
- Particle systems can be disabled
- 2D mode is faster than 3D

### 9. **Persistent State**
- Settings saved to browser localStorage
- Preferences persist across sessions
- Per-user customization

### 10. **Community Detection**
- Uses Louvain algorithm (jLouvain.js)
- Identifies clusters in knowledge graph
- Can color nodes by community

---

## Notable Features

1. **Real-time Following**: Browser follows your position in Emacs
2. **Bidirectional Navigation**: Click nodes in browser to open in Emacs
3. **Note Preview**: View note content without leaving the graph
4. **Advanced Filtering**: By tags, directories, dates, orphans, citations
5. **3D Visualization**: Optional 3D graph for deeper exploration
6. **Citation Support**: Integrates with org-roam-bibtex
7. **Customizable Physics**: Full control over graph layout
8. **Theme Integration**: Matches your Emacs color scheme
9. **Local Graph Mode**: Focus on specific nodes and their neighbors
10. **Context Menu**: Right-click actions for nodes

---

## Comparison with org-roam-server

org-roam-ui is explicitly designed as a successor to org-roam-server with:
- More features (3D, local mode, better filtering)
- Active development and maintenance
- Better performance
- Modern tech stack (Next.js vs simple HTTP server)
- More customization options
- Real-time bidirectional communication

---

## Potential Use Cases for Integration

Based on this analysis, org-roam-ui could integrate with other org-roam tools by:

1. **Shared WebSocket Protocol**: Other tools could connect to the same WebSocket for graph data
2. **HTTP API**: The `/node/:id` endpoint could be used by other applications
3. **Database Queries**: The SQL queries in the Elisp file are reusable
4. **Theme Extraction**: The theme sync logic could be extracted as a library
5. **Graph Algorithms**: Community detection and neighbor finding could be reused
6. **Document Processing**: The uniorg processing pipeline could render org content elsewhere

---

## Conclusion

This comprehensive analysis provides a solid foundation for understanding how org-roam-ui works and how it might relate to or integrate with other tools in the org-roam ecosystem. The architecture is well-designed for extensibility, with clear separation between data provision (Emacs), visualization (React), and user interaction (WebSocket protocol).
