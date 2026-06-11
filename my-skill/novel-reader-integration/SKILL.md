---
name: novel-reader-integration
description: Complete client-server engineering blueprint for building a high-fidelity e-book reader. Details client-side layout, page-sliding animations, TTS word highlighting, ambient mixer, annotations, bookshelf management, and Legado-compatible backend parsing.
risk: safe
source: user-personal
date_added: "2026-06-09"
---

# End-to-End Novel Reading Integration & Engine Skill

This skill documents the complete database schemas, server-side Legado parsers, concurrent APIs, client-side state providers, and premium user interface components required to build a feature-complete novel reading platform.

---

## 🎨 1. Core Architecture Blueprint

```mermaid
graph TD
    subgraph Client Application (Flutter / Dart)
        Shelf[Bookshelf Tab] --> Reader[NovelReaderScreen]
        Oasis[Discover Oasis Tab] --> Reader
        Reader --> PageView[ReaderPageView Widget]
        Reader --> Settings[Settings Panel]
        Reader --> Notes[Annotations Sheet]
        
        State[NovelState Provider] --> ClientAPI[NovelApiClient]
        State --> LocalDB[Local SQLite Database]
    end

    subgraph Backend Parser Server (FastAPI / Python)
        API[Router Endpoints] --> DB[(PostgreSQL / SQLite)]
        API --> Parser[Legado Parsing Engine]
        API --> SearchConduit[Parallel Search Pipeline]
        Parser --> Scraper[Multi-page Scraper & Auto-decoder]
    end

    ClientAPI -- HTTPS/SSE --> API
    Scraper -- Scrape HTML/JSON --> ThirdParty[Third-Party Book Sources]
```

---

## 🗄️ 2. Database & Data Models

### SQLAlchemy Server Models
```python
import uuid
from sqlalchemy import Column, String, Boolean, DateTime, Integer, ForeignKey
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.sql import func
from app.core.database import Base

class BookSourceRule(Base):
    """Stores Legado-compatible rules parsing configurations."""
    __tablename__ = "book_source_rules"

    id = Column(String(255), primary_key=True)
    source_name = Column(String(255), nullable=False)
    source_url = Column(String(255), nullable=False)
    search_url = Column(String, nullable=True)
    rule_json = Column(String, nullable=True)  # Raw JSON string configuration
    is_active = Column(Boolean, default=True)
    is_valid = Column(Boolean, default=True)
    is_private = Column(Boolean, default=False)
    is_abyss = Column(Boolean, default=False)  # Private network source flag
    last_checked_at = Column(DateTime(timezone=True), server_default=func.now(), nullable=False)

class UserReadingProgress(Base):
    """Tracks active reading records using Max-Progress-Wins logic."""
    __tablename__ = "user_reading_progress"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    user_id = Column(UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False, index=True)
    book_id = Column(String(255), nullable=False, index=True)
    book_title = Column(String(255), nullable=True)
    book_author = Column(String(255), nullable=True)
    cover_url = Column(String, nullable=True)
    source_id = Column(String(255), nullable=True)
    book_url = Column(String, nullable=True)
    last_read_chapter_index = Column(Integer, default=0, nullable=False)
    last_read_char_offset = Column(Integer, default=0, nullable=False)
    is_abyss = Column(Boolean, default=False)
    in_bookshelf = Column(Boolean, default=True, nullable=False)
    updated_at = Column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now(), nullable=False)
```

### Dart Client Models
```dart
class Book {
  final String id;
  final String title;
  final String author;
  final String coverUrl;
  final String summary;
  final String? currentSourceId;
  final bool isAbyss;
  final String? bookUrl;

  Book({
    required this.id,
    required this.title,
    required this.author,
    required this.coverUrl,
    required this.summary,
    this.currentSourceId,
    required this.isAbyss,
    this.bookUrl,
  });

  factory Book.fromJson(Map<String, dynamic> json) => Book(
    id: json['id'] ?? '',
    title: json['title'] ?? '',
    author: json['author'] ?? '',
    coverUrl: json['cover_url'] ?? '',
    summary: json['summary'] ?? '',
    currentSourceId: json['current_source_id'],
    isAbyss: json['is_abyss'] ?? false,
    bookUrl: json['book_url'],
  );
}

class BookChapter {
  final String id;
  final String bookId;
  final int chapterIndex;
  final String title;
  final String sourceChapterUrl;
  final String? content;

  BookChapter({
    required this.id,
    required this.bookId,
    required this.chapterIndex,
    required this.title,
    required this.sourceChapterUrl,
    this.content,
  });

  factory BookChapter.fromJson(Map<String, dynamic> json) => BookChapter(
    id: json['id'] ?? '',
    bookId: json['book_id'] ?? '',
    chapterIndex: json['chapter_index'] ?? 0,
    title: json['title'] ?? '',
    sourceChapterUrl: json['source_chapter_url'] ?? '',
    content: json['content'],
  );
}
```

---

## ⚙️ 3. Server-Side Legado Parsing Engine (Python)

Converts Legado configurations into BeautifulSoup Jsoup actions, XPath evaluations, JSONPath selectors, JS bridges, and regex capture translations.

```python
import re
import json
import base64
import hashlib
from typing import List, Dict, Any, Union
from bs4 import BeautifulSoup, Tag
from jsonpath_ng.ext import parse as jsonpath_parse
from lxml import etree

class LegadoJsoupIndex:
    def __init__(self, before_rule: str, split_char: str, indexes: List[Union[int, slice]]):
        self.before_rule = before_rule
        self.split_char = split_char
        self.indexes = indexes

class LegadoJSBridge:
    def __init__(self, variables: Dict[str, Any]):
        self.variables = variables
    def get(self, key: str) -> Any: return self.variables.get(key, '')
    def put(self, key: str, val: Any) -> Any:
        self.variables[key] = val
        return val
    def md5(self, s: str) -> str: return hashlib.md5(str(s).encode('utf-8')).hexdigest()
    def base64Encode(self, s: str) -> str: return base64.b64encode(str(s).encode('utf-8')).decode('utf-8')
    def base64Decode(self, s: str) -> str:
        try: return base64.b64decode(str(s)).decode('utf-8')
        except Exception: return ""

class LegadoParserEngine:
    @staticmethod
    def parse_legado_jsoup_index(rule_str: str) -> LegadoJsoupIndex:
        rule_str = rule_str.strip()
        if not rule_str: return LegadoJsoupIndex('', ' ', [])
        
        # 1. Bracket selection: tag.div[-1, 0:3] or tag.div[!0:2]
        if rule_str.endswith(']'):
            idx = rule_str.rfind('[')
            if idx != -1:
                before_rule = rule_str[:idx].strip()
                bracket_content = rule_str[idx + 1:-1].strip()
                split_char = '.'
                if bracket_content.startswith('!'):
                    split_char = '!'
                    bracket_content = bracket_content[1:].strip()
                
                indexes = []
                for part in bracket_content.split(','):
                    part = part.strip()
                    if ':' in part:
                        subparts = part.split(':')
                        start = int(subparts[0]) if subparts[0] else 0
                        end = int(subparts[1]) if len(subparts) > 1 and subparts[1] else None
                        step = int(subparts[2]) if len(subparts) > 2 and subparts[2] else 1
                        indexes.append(slice(start, end, step))
                    else:
                        try: indexes.append(int(part))
                        except ValueError: pass
                return LegadoJsoupIndex(before_rule, split_char, indexes)
        return LegadoJsoupIndex(rule_str, ' ', [])

    @staticmethod
    def apply_legado_indexes(elements: List[Tag], split_char: str, indexes: List[Any]) -> List[Tag]:
        if not elements or split_char == ' ': return elements
        n = len(elements)
        index_set = []
        for item in indexes:
            if isinstance(item, slice):
                stop = item.stop if item.stop is not None else n
                index_set.extend(list(range(n)[slice(item.start, stop, item.step)]))
            elif isinstance(item, int):
                idx = item if item >= 0 else item + n
                if 0 <= idx < n: index_set.append(idx)
        
        index_set = list(dict.fromkeys(index_set))
        return [elements[i] for i in range(n) if i not in index_set] if split_char == '!' else [elements[idx] for idx in index_set]

    @staticmethod
    def evaluate_selector(element: Any, selector_str: str, variables: Dict[str, Any]) -> str:
        if element is None or not selector_str: return ''
        
        # Variable scoping parser: selector@put:{varName: selector}
        if '@put:' in selector_str:
            parts = selector_str.split('@put:')
            base_selector, put_content = parts[0].strip(), parts[1].strip()
            if put_content.startswith('{') and put_content.endswith('}'):
                inner = put_content[1:-1].strip()
                if ':' in inner:
                    idx = inner.find(':')
                    var_name, var_selector = inner[:idx].strip(), inner[idx+1:].strip()
                    variables[var_name] = LegadoParserEngine.evaluate_selector(element, var_selector, variables)
            selector_str = base_selector

        # Regex replace parser (selector##pattern##replacement)
        base_selector, regex_part = selector_str, ''
        if '##' in selector_str:
            parts = selector_str.split('##')
            base_selector = parts[0].strip()
            regex_part = '##' + "##".join(parts[1:])

        # JS interpreter block parser
        js_code = ''
        if '@js:' in base_selector:
            idx = base_selector.find('@js:')
            js_code = base_selector[idx:]
            base_selector = base_selector[:idx].strip()

        # XPath & JSONPath routers
        if base_selector.startswith('//') or base_selector.startswith('xpath:'):
            xml_str = element if isinstance(element, str) else str(element)
            res = LegadoParserEngine.evaluate_xpath(xml_str, base_selector)
            result_text = "\n".join(res)
        elif base_selector.startswith('$.') or base_selector.startswith('json:'):
            json_data = json.loads(element) if isinstance(element, str) else element
            val = LegadoParserEngine.evaluate_jsonpath(json_data, base_selector)
            result_text = str(val)
        else:
            # HTML Jsoup evaluation
            html_el = element if isinstance(element, Tag) else BeautifulSoup(str(element), 'html.parser')
            parsed = LegadoParserEngine.parse_legado_jsoup_index(base_selector)
            selected = html_el.select(parsed.before_rule) if parsed.before_rule else [html_el]
            filtered = LegadoParserEngine.apply_legado_indexes(selected, parsed.split_char, parsed.indexes)
            result_text = "\n".join(el.get_text().strip() for el in filtered)

        if js_code:
            result_text = LegadoParserEngine.evaluate_js(result_text, js_code, variables)
        if regex_part:
            result_text = LegadoParserEngine.apply_regex_replacements(result_text, regex_part)
            
        return result_text

    @staticmethod
    def apply_regex_replacements(text: str, rule_str: str) -> str:
        parts = rule_str.split('##')
        curr = text
        for i in range(1, len(parts), 2):
            pat = parts[i]
            rep = parts[i+1] if i+1 < len(parts) else ''
            # Convert Javascript capture group reference ($1) to Python pattern group (\1)
            rep = re.sub(r'\$(\d)', r'\\\1', rep)
            curr = re.sub(pat, rep, curr, flags=re.MULTILINE | re.DOTALL)
        return curr

    @staticmethod
    def evaluate_js(text: str, js_code: str, variables: Dict[str, Any]) -> str:
        code = js_code[4:] if js_code.startswith('@js:') else js_code
        try:
            import js2py
            context = js2py.EvalJs()
            context.put("java", LegadoJSBridge(variables))
            context.put("result", text)
            context.execute(code)
            return str(context.result) if context.result is not None else ''
        except Exception:
            return text
```

---

## ⚡ 4. Backend Endpoints (FastAPI)

### SSE Search Stream API
Streams search outputs as they compile using isolated worker semaphores to maintain concurrency:

```python
from fastapi import APIRouter, Query, Depends
from fastapi.responses import StreamingResponse
import asyncio, httpx, json

router = APIRouter(prefix="/novel")

@router.get("/search/stream")
async def search_novels_stream(q: str = Query(...), db: Session = Depends(get_db)):
    sources = db.query(BookSourceRule).filter(BookSourceRule.is_active == True).all()

    async def event_generator():
        # Maximum concurrent sockets
        sem = asyncio.Semaphore(12)
        async with httpx.AsyncClient(verify=False) as client:
            tasks = [search_single_source(s, q, client, sem) for s in sources]
            for future in asyncio.as_completed(tasks):
                try:
                    books = await future
                    if books:
                        yield f"data: {json.dumps({'status': 'progress', 'books': books}, ensure_ascii=False)}\n\n"
                except Exception:
                    pass
        yield "data: {\"status\": \"done\"}\n\n"

    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

### Adaptive Charset Decoder
```python
def decode_content(content_bytes: bytes, charset: str = 'utf-8') -> str:
    try: return content_bytes.decode(charset)
    except Exception: pass
    for enc in ['utf-8', 'gbk', 'gb2312', 'latin-1']:
        try: return content_bytes.decode(enc)
        except Exception: pass
    return content_bytes.decode('utf-8', errors='ignore')
```

### Chapter Pagination Assembler (Multi-page Crawler)
Iteratively crawls sub-pages of single long chapters to merge full content:

```python
async def parse_multipage_chapter(client: httpx.AsyncClient, first_page_url: str, content_selector: str, next_page_selector: str) -> str:
    current_url, contents, visited = first_page_url, [], set()
    while current_url and current_url not in visited:
        visited.add(current_url)
        res = await client.get(current_url, timeout=10.0)
        if res.status_code != 200: break
        
        html = decode_content(res.content)
        page_text = LegadoParserEngine.evaluate_selector(html, content_selector, {})
        if page_text: contents.append(page_text)
        
        next_path = LegadoParserEngine.evaluate_selector(html, next_page_selector, {})
        if next_path:
            from urllib.parse import urljoin
            current_url = urljoin(current_url, next_path)
        else:
            current_url = None
    return "\n\n".join(contents)
```

---

## 📱 5. Client State Provider (Flutter / Riverpod)

Tracks settings, local SQLite bookshelf, TTS threads, and handles zero-flicker prefetching.

```dart
class NovelState {
  final List<Book> bookshelf;
  final List<Book> searchResults;
  final ReadingProgress? currentBookProgress;
  final List<BookChapter> chapters;
  final BookChapter? currentChapter;
  final bool isBookshelfLoading;
  final bool isContentLoading;
  final bool isTtsPlaying;
  final int ttsHighlightCharIndex;

  NovelState({
    this.bookshelf = const [],
    this.searchResults = const [],
    this.currentBookProgress,
    this.chapters = const [],
    this.currentChapter,
    this.isBookshelfLoading = false,
    this.isContentLoading = false,
    this.isTtsPlaying = false,
    this.ttsHighlightCharIndex = 0,
  });

  NovelState copyWith({
    List<Book>? bookshelf,
    List<Book>? searchResults,
    ReadingProgress? currentBookProgress,
    List<BookChapter>? chapters,
    BookChapter? currentChapter,
    bool? isBookshelfLoading,
    bool? isContentLoading,
    bool? isTtsPlaying,
    int? ttsHighlightCharIndex,
  }) => NovelState(
    bookshelf: bookshelf ?? this.bookshelf,
    searchResults: searchResults ?? this.searchResults,
    currentBookProgress: currentBookProgress ?? this.currentBookProgress,
    chapters: chapters ?? this.chapters,
    currentChapter: currentChapter ?? this.currentChapter,
    isBookshelfLoading: isBookshelfLoading ?? this.isBookshelfLoading,
    isContentLoading: isContentLoading ?? this.isContentLoading,
    isTtsPlaying: isTtsPlaying ?? this.isTtsPlaying,
    ttsHighlightCharIndex: ttsHighlightCharIndex ?? this.ttsHighlightCharIndex,
  );
}

class NovelNotifier extends StateNotifier<NovelState> {
  final NovelApiClient _apiClient;
  final Map<String, BookChapter> _prefetchCache = {};

  NovelNotifier(this._apiClient) : super(NovelState());

  /// Loads chapter text. Instantly resolves from cache if prefetched, preventing UI flashing.
  Future<void> loadChapter(String bookId, int chapterIndex) async {
    final cacheKey = '${bookId}_$chapterIndex';
    state = state.copyWith(isContentLoading: true);

    if (_prefetchCache.containsKey(cacheKey)) {
      final cached = _prefetchCache[cacheKey]!;
      state = state.copyWith(currentChapter: cached, isContentLoading: false);
      _triggerPrefetchSequence(bookId, chapterIndex);
      return;
    }

    try {
      final chapter = await _apiClient.getChapterContent(bookId, chapterIndex);
      _prefetchCache[cacheKey] = chapter;
      state = state.copyWith(currentChapter: chapter, isContentLoading: false);
      _triggerPrefetchSequence(bookId, chapterIndex);
    } catch (e) {
      state = state.copyWith(isContentLoading: false);
    }
  }

  void _triggerPrefetchSequence(String bookId, int chapterIndex) {
    // Prefetch surrounding pages (N+1, N+2, N-1) in the background asynchronously
    for (int offset in [1, 2, -1]) {
      final target = chapterIndex + offset;
      if (target < 0) continue;
      final key = '${bookId}_$target';
      if (_prefetchCache.containsKey(key)) continue;

      _apiClient.getChapterContent(bookId, target).then((chap) {
        _prefetchCache[key] = chap;
      }).catchError((_) {});
    }
  }
}
```

---

## 🎨 6. Premium UI Components (Flutter / Dart)

### ReaderPageView Widget
Features a custom matrix transformation animating page splits, viewport padding constraints, clock and progress footers, and live TTS word highlight layers.

```dart
class ReaderPageView extends ConsumerStatefulWidget {
  final String bookId;
  final NovelState state;
  final String content;
  final TextStyle textStyle;
  final double marginHorizontal;
  final int activeThemeIndex;
  final List<Color> themeBgColors;
  final VoidCallback onNextChapter;
  final VoidCallback onPrevChapter;

  const ReaderPageView({
    super.key,
    required this.bookId,
    required this.state,
    required this.content,
    required this.textStyle,
    required this.marginHorizontal,
    required this.activeThemeIndex,
    required this.themeBgColors,
    required this.onNextChapter,
    required this.onPrevChapter,
  });

  @override
  ConsumerState<ReaderPageView> createState() => _ReaderPageViewState();
}

class _ReaderPageViewState extends ConsumerState<ReaderPageView> {
  late PageController _pageController;
  final Map<int, TextSpan> _spanCache = {};

  @override
  void initState() {
    super.initState();
    _pageController = PageController();
  }

  @override
  Widget build(BuildContext context) {
    final double padding = widget.marginHorizontal;
    final double reservedHeight = 65.0 + MediaQuery.of(context).padding.top + MediaQuery.of(context).padding.bottom;
    
    return LayoutBuilder(
      builder: (context, constraints) {
        final double viewWidth = constraints.maxWidth - (2 * padding);
        final double viewHeight = constraints.maxHeight - reservedHeight;

        // Paginate raw text into character offset slices
        final List<int> pageOffsets = NovelTextPaginator.paginate(
          widget.content,
          widget.textStyle,
          viewWidth,
          viewHeight,
        );
        final int totalPages = pageOffsets.length - 1;

        return PageView.builder(
          controller: _pageController,
          itemCount: totalPages,
          itemBuilder: (context, index) {
            final int start = pageOffsets[index];
            final int end = pageOffsets[index + 1];
            final String pageText = widget.content.substring(start, end);

            // Matrix simulation for premium page-sliding shadow overlays
            return AnimatedBuilder(
              animation: _pageController,
              builder: (context, child) {
                double pageVal = _pageController.hasClients ? (_pageController.page ?? 0.0) : 0.0;
                double delta = index - pageVal;
                
                double translationX = 0.0;
                double scale = 1.0;
                double opacity = 1.0;

                if (delta < 0) {
                  scale = 1.0 + (delta * 0.06);
                  opacity = (1.0 + delta).clamp(0.1, 1.0);
                  translationX = -delta * constraints.maxWidth * 0.45;
                }

                return Transform(
                  transform: Matrix4.translationValues(translationX, 0.0, 0.0)..scale(scale),
                  child: Opacity(
                    opacity: opacity,
                    child: child,
                  ),
                );
              },
              child: Container(
                color: widget.themeBgColors[widget.activeThemeIndex],
                padding: EdgeInsets.symmetric(horizontal: padding),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    // Chapter Title Header
                    Text(
                      widget.state.currentChapter?.title ?? '',
                      style: TextStyle(fontSize: 12, color: widget.textStyle.color?.withOpacity(0.4)),
                    ),
                    const SizedBox(height: 15),
                    // Core Reading Text
                    Expanded(
                      child: TextHighlightLayer(
                        text: pageText,
                        style: widget.textStyle,
                        highlightIndex: widget.state.ttsHighlightCharIndex - start,
                        isHighlighting: widget.state.isTtsPlaying,
                      ),
                    ),
                    // Clock and Page Index Footer
                    Row(
                      mainAxisAlignment: MainAxisAlignment.spaceBetween,
                      children: [
                        Text(
                          "${DateTime.now().hour}:${DateTime.now().minute.toString().padLeft(2, '0')}",
                          style: TextStyle(fontSize: 11, color: widget.textStyle.color?.withOpacity(0.4)),
                        ),
                        Text(
                          '第 ${index + 1} / $totalPages 页',
                          style: TextStyle(fontSize: 11, color: widget.textStyle.color?.withOpacity(0.4)),
                        ),
                      ],
                    ),
                  ],
                ),
              ),
            );
          },
        );
      },
    );
  }
}
```

### Interactive Text Highlight Layer
Highlights spoken words in the text paragraph:

```dart
class TextHighlightLayer extends StatelessWidget {
  final String text;
  final TextStyle style;
  final int highlightIndex;
  final bool isHighlighting;

  const TextHighlightLayer({
    super.key,
    required this.text,
    required this.style,
    required this.highlightIndex,
    required this.isHighlighting,
  });

  @override
  Widget build(BuildContext context) {
    if (!isHighlighting || highlightIndex < 0 || highlightIndex >= text.length) {
      return RichText(text: TextSpan(text: text, style: style));
    }

    final int start = highlightIndex;
    final int end = (start + 12).clamp(start, text.length); // Spoken block length

    return RichText(
      text: TextSpan(
        style: style,
        children: [
          TextSpan(text: text.substring(0, start)),
          TextSpan(
            text: text.substring(start, end),
            style: const TextStyle(color: Colors.amber, backgroundColor: Colors.black12, fontWeight: FontWeight.bold),
          ),
          TextSpan(text: text.substring(end)),
        ],
      ),
    );
  }
}
```

### Reader Settings & Annotation Note Sheets
- **Settings Sheet**: Builds options overlays utilizing sliders mapping to `NovelNotifier` parameters (FontSize, LineSpacing, FontSerif vs FontSans, theme selections).
- **Annotation Overlay**: Binds long-press gesture details:
  ```dart
  GestureDetector(
    onLongPressStart: (details) {
      // Open modal bottom sheet asking for note content:
      showModalBottomSheet(
        context: context,
        builder: (context) => CreateNoteSheet(
          chapterId: state.currentChapter!.id,
          startOffset: start,
          endOffset: end,
        ),
      );
    },
  )
  ```

---

## 🚀 7. Step-by-Step Implementation Checklist

This roadmap outlines how to construct the complete application package from scratch:

1.  **Backend Foundations**:
    - [ ] Initialize Python environment with FastAPI, SQLAlchemy, `bs4`, `js2py`, and `httpx`.
    - [ ] Create PostgreSQL database schemas for `book_source_rules` and `user_reading_progress`.
    - [ ] Run initialization migrations to load Legado JSON rules ruleset.
2.  **Scraping & Parsing Pipeline**:
    - [ ] Write the `LegadoParserEngine` supporting CSS/Jsoup bracket indexing and variables cache.
    - [ ] Write the Javascript bridge class `LegadoJSBridge` mocking MD5, Base64, and AES methods.
    - [ ] Hook up `asyncio.Semaphore` concurrent HTTP request pools.
    - [ ] Program subsequent pagination scraper loop (`nextContentUrl`).
3.  **Core API Router**:
    - [ ] Implement SSE streaming route `/search/stream`.
    - [ ] Implement race ranking router `/explore/rankings`.
    - [ ] Implement progress synchronizer `/progress` validating offsets using Max-Progress-Wins logic.
    - [ ] Launch hourly background health check query loop.
4.  **Client Application Structure**:
    - [ ] Setup Flutter app, bind packages `flutter_riverpod`, `just_audio`, and `flutter_tts`.
    - [ ] Set up local SQLite caching to mirror the server user progress.
    - [ ] Connect `NovelApiClient` mapping search and contents JSON endpoints.
5.  **State Management Layer**:
    - [ ] Create `NovelState` and notifier provider.
    - [ ] Implement memory cache prefetch algorithm checking $N \pm 3$ pages.
    - [ ] Bind TTS handlers syncing highlighted character offsets.
    - [ ] Initialize `just_audio` background ambient looping channels.
6.  **Interactive Reading UI**:
    - [ ] Construct main bookshelf grid and exploring oasis screen.
    - [ ] Write `NovelTextPaginator` layout builder.
    - [ ] Build `ReaderPageView` with page-sliding matrix transforms.
    - [ ] Hook up settings panel overlay (text size, parchment theme selector) and annotations creator.
