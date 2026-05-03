---
globs:
  - "frontend/**"
---

# Frontend Rules

## Tech Stack
- React.js (CRA hoặc Vite)
- Tailwind CSS (utility-first)
- Fetch API / Axios cho HTTP requests

## Component Conventions
- Functional components + hooks (không dùng class components)
- File naming: PascalCase cho components (`CameraGrid.jsx`)
- Folder structure: group by feature, không by type

## Pages
- **Command Center**: Real-time camera grid, live risk scores
- **Incident Data Viewer**: Raw incident data table
- **Analytics Dashboard**: Multi-layer charts and statistics
- **Vigilance Terminal**: Agentic RAG chatbot interface

## API Integration
- Backend: FastAPI chatbot tại `http://localhost:8000`
- Endpoint: `POST /api/chat` cho RAG queries
- WebSocket (future): real-time alert push

## Styling
- Dùng Tailwind utility classes, tránh custom CSS khi có thể
- Dark theme là default (security monitoring UI)
- Color coding: red = danger/violence, green = safe, yellow = warning

## Performance
- Lazy load heavy components (camera grid, charts)
- Debounce search/filter inputs (300ms)
- Pagination cho data tables (50 rows/page)
