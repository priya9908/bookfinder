// BookFinder.jsx
// Single-file React component (default export) built with Tailwind + shadcn/ui style imports
// Usage: drop into a React app (Vite / Next.js). Tailwind must be configured.
// Libraries used (install as needed):
// react, framer-motion, lucide-react, shadcn/ui components (optional), date-fns
// Example: npm i framer-motion lucide-react date-fns

import React, { useEffect, useState, useMemo } from "react";
import { motion } from "framer-motion";
import { Search, BookOpen, Heart, Filter, X, Download } from "lucide-react";

// NOTE: shadcn/ui components are suggested; if not installed, simple semantic HTML is used instead.
// import { Card, CardContent } from "@/components/ui/card"; // optional

// ---------- Configuration / Helper functions ----------
const GOOGLE_BOOKS_API = "https://www.googleapis.com/books/v1/volumes"; // optional key can be added as &key=YOUR_KEY
const OPEN_LIBRARY_API = "https://openlibrary.org/search.json";

function buildGoogleBooksQuery(q, options = {}) {
  const params = new URLSearchParams();
  params.set("q", q);
  if (options.maxResults) params.set("maxResults", String(options.maxResults));
  if (options.startIndex) params.set("startIndex", String(options.startIndex));
  return `${GOOGLE_BOOKS_API}?${params.toString()}`;
}

function buildOpenLibraryQuery(paramsObj) {
  const params = new URLSearchParams(paramsObj);
  return `${OPEN_LIBRARY_API}?${params.toString()}`;
}

function safeGet(obj, path, fallback = undefined) {
  return path.split(".").reduce((acc, k) => (acc && acc[k] !== undefined ? acc[k] : undefined), obj) ?? fallback;
}

function extractGoogleBook(item) {
  const vol = item.volumeInfo || {};
  return {
    id: item.id,
    title: vol.title || "Untitled",
    authors: vol.authors || [],
    publisher: vol.publisher || "",
    publishedDate: vol.publishedDate || "",
    description: vol.description || "",
    pageCount: vol.pageCount || null,
    categories: vol.categories || [],
    thumbnail: safeGet(vol, "imageLinks.thumbnail")?.replace("http:", "https:") ?? null,
    previewLink: vol.previewLink || null,
    infoLink: vol.infoLink || null,
    source: "google",
  };
}

function extractOpenLibraryDoc(doc) {
  return {
    id: `ol:${doc.key}`,
    title: doc.title || "Untitled",
    authors: doc.author_name || [],
    publisher: (doc.publisher && doc.publisher[0]) || "",
    publishedDate: doc.first_publish_year || "",
    description: doc.subtitle || "",
    pageCount: doc.number_of_pages_median || null,
    categories: doc.subject || [],
    thumbnail: doc.cover_i ? `https://covers.openlibrary.org/b/id/${doc.cover_i}-M.jpg` : null,
    previewLink: doc.key ? `https://openlibrary.org${doc.key}` : null,
    infoLink: doc.key ? `https://openlibrary.org${doc.key}` : null,
    source: "openlibrary",
  };
}

function formatCitation(item, style = "APA") {
  // naive citation generator for demonstration. For production use a proper library.
  const authors = (item.authors || []).join(", ");
  const year = item.publishedDate ? `(${item.publishedDate})` : "(n.d.)";
  if (style === "APA") return `${authors} ${year}. ${item.title}. ${item.publisher || ""}`.trim();
  if (style === "MLA") return `${authors}. "${item.title}." ${item.publisher || ""}, ${item.publishedDate || "n.d."}`;
  return `${item.title} — ${authors}`;
}

// ---------- Mock/fallback data for offline / demo ----------
const DEMO_BOOKS = [
  {
    id: "demo:1",
    title: "Thinking in Systems",
    authors: ["Donella H. Meadows"],
    publisher: "Chelsea Green Publishing",
    publishedDate: "2008",
    description: "A primer on systems thinking...",
    thumbnail: null,
    source: "demo",
  },
  {
    id: "demo:2",
    title: "Atomic Habits",
    authors: ["James Clear"],
    publisher: "Avery",
    publishedDate: "2018",
    description: "Tiny changes, remarkable results...",
    thumbnail: null,
    source: "demo",
  },
];

// ---------- Main Component ----------
export default function BookFinder() {
  // Search & filters
  const [query, setQuery] = useState("");
  const [searchBy, setSearchBy] = useState("intitle"); // intitle, inauthor, isbn, subject
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [page, setPage] = useState(0);
  const [perPage] = useState(12);

  // UI states
  const [selected, setSelected] = useState(null);
  const [wishlist, setWishlist] = useState(() => {
    try {
      return JSON.parse(localStorage.getItem("bf:wishlist")) || [];
    } catch (e) {
      return [];
    }
  });
  const [source, setSource] = useState("both"); // google, openlibrary, both
  const [yearRange, setYearRange] = useState([1900, new Date().getFullYear()]);
  const [sortBy, setSortBy] = useState("relevance");

  useEffect(() => {
    localStorage.setItem("bf:wishlist", JSON.stringify(wishlist));
  }, [wishlist]);

  // ---------- Actions ----------
  async function doSearch(newPage = 0) {
    if (!query) {
      setResults(DEMO_BOOKS);
      return;
    }
    setLoading(true);
    setError(null);
    setPage(newPage);

    try {
      const promises = [];
      const qPrefix = searchBy === "isbn" ? `isbn:${query}` : searchBy === "inauthor" ? `inauthor:${query}` : searchBy === "subject" ? `subject:${query}` : `intitle:${query}`;

      if (source === "google" || source === "both") {
        const gUrl = buildGoogleBooksQuery(qPrefix, { maxResults: perPage, startIndex: newPage * perPage });
        promises.push(fetch(gUrl).then((r) => r.json()).then((j) => (j.items || []).map(extractGoogleBook)));
      }
      if (source === "openlibrary" || source === "both") {
        const olUrl = buildOpenLibraryQuery({ q: query, page: newPage + 1, limit: perPage });
        promises.push(fetch(olUrl).then((r) => r.json()).then((j) => (j.docs || []).map(extractOpenLibraryDoc)));
      }

      const arrays = await Promise.all(promises);
      const merged = arrays.flat();
      // naive de-duplication by title+authors
      const uniq = [];
      const seen = new Set();
      for (const it of merged) {
        const key = `${it.title}::${(it.authors || []).join(",")}`;
        if (!seen.has(key)) {
          seen.add(key);
          uniq.push(it);
        }
      }

      // Another simple filter: year range
      const filtered = uniq.filter((b) => {
        if (!b.publishedDate) return true;
        const year = parseInt(("" + b.publishedDate).slice(0, 4));
        if (Number.isNaN(year)) return true;
        return year >= yearRange[0] && year <= yearRange[1];
      });

      setResults(filtered);
    } catch (e) {
      console.error(e);
      setError("Search failed. Try again or check your network.");
      setResults(DEMO_BOOKS);
    } finally {
      setLoading(false);
    }
  }

  function toggleWishlist(item) {
    setWishlist((prev) => {
      const exists = prev.find((p) => p.id === item.id);
      if (exists) return prev.filter((p) => p.id !== item.id);
      return [item, ...prev];
    });
  }

  function handleQuickAddDemo() {
    setResults((r) => [...DEMO_BOOKS, ...r]);
  }

  // Derived values
  const isInWishlist = (id) => wishlist.some((w) => w.id === id);

  // ---------- Small subcomponents inline ----------
  function SearchBar() {
    return (
      <div className="w-full flex gap-2 items-center">
        <div className="flex-1 relative">
          <input
            value={query}
            onChange={(e) => setQuery(e.target.value)}
            onKeyDown={(e) => e.key === "Enter" && doSearch(0)}
            placeholder="Search by title, author, ISBN or subject"
            className="w-full rounded-lg border p-3 pl-10 bg-white"
          />
          <Search className="absolute left-3 top-3" size={16} />
        </div>
        <button
          onClick={() => doSearch(0)}
          className="px-4 py-2 rounded-lg bg-slate-900 text-white flex items-center gap-2"
        >
          Search
        </button>
        <button
          onClick={handleQuickAddDemo}
          title="Add demo items"
          className="px-3 py-2 rounded-lg border"
        >
          Demo
        </button>
      </div>
    );
  }

  function FiltersPanel() {
    return (
      <div className="p-4 rounded-lg border bg-white">
        <div className="flex gap-2 items-center mb-3">
          <Filter />
          <strong>Filters</strong>
        </div>
        <div className="grid grid-cols-2 gap-2">
          <div>
            <label className="text-sm">Source</label>
            <select value={source} onChange={(e) => setSource(e.target.value)} className="w-full mt-1 p-2 border rounded">
              <option value="both">Both (Google + OpenLibrary)</option>
              <option value="google">Google Books</option>
              <option value="openlibrary">Open Library</option>
            </select>
          </div>

          <div>
            <label className="text-sm">Search mode</label>
            <select value={searchBy} onChange={(e) => setSearchBy(e.target.value)} className="w-full mt-1 p-2 border rounded">
              <option value="intitle">Title</option>
              <option value="inauthor">Author</option>
              <option value="isbn">ISBN</option>
              <option value="subject">Subject</option>
            </select>
          </div>

          <div>
            <label className="text-sm">Sort</label>
            <select value={sortBy} onChange={(e) => setSortBy(e.target.value)} className="w-full mt-1 p-2 border rounded">
              <option value="relevance">Relevance</option>
              <option value="newest">Newest</option>
              <option value="oldest">Oldest</option>
            </select>
          </div>

          <div>
            <label className="text-sm">Year range (min)</label>
            <input type="number" value={yearRange[0]} onChange={(e) => setYearRange([Number(e.target.value), yearRange[1]])} className="w-full mt-1 p-2 border rounded" />
            <label className="text-sm mt-2">Year range (max)</label>
            <input type="number" value={yearRange[1]} onChange={(e) => setYearRange([yearRange[0], Number(e.target.value)])} className="w-full mt-1 p-2 border rounded" />
          </div>
        </div>
      </div>
    );
  }

  function ResultsGrid() {
    if (loading) return <div className="p-6">Loading results...</div>;
    if (error) return <div className="p-6 text-red-600">{error}</div>;
    if (!results || results.length === 0) return <div className="p-6">No results. Try a different search.</div>;

    return (
      <div className="grid grid-cols-1 md:grid-cols-3 lg:grid-cols-4 gap-4">
        {results.map((r) => (
          <motion.div key={r.id} layout whileHover={{ scale: 1.02 }} className="p-4 border rounded-lg bg-white">
            <div className="flex gap-3">
              <div className="w-20 h-28 bg-slate-100 rounded overflow-hidden flex items-center justify-center">
                {r.thumbnail ? <img src={r.thumbnail} alt={r.title} className="object-cover" /> : <BookOpen />}
              </div>
              <div className="flex-1">
                <h3 className="font-semibold text-sm line-clamp-2">{r.title}</h3>
                <p className="text-xs text-slate-600">{(r.authors || []).slice(0, 2).join(", ")}</p>
                <p className="text-xs mt-2">{r.publisher} {r.publishedDate ? `· ${r.publishedDate}` : ""}</p>

                <div className="flex gap-2 mt-3 items-center">
                  <button onClick={() => setSelected(r)} className="px-2 py-1 border rounded text-xs">Preview</button>
                  <button onClick={() => toggleWishlist(r)} className="px-2 py-1 border rounded text-xs flex items-center gap-1">
                    <Heart size={14} /> {isInWishlist(r.id) ? "Saved" : "Save"}
                  </button>
                </div>
              </div>
            </div>
          </motion.div>
        ))}
      </div>
    );
  }

  function WishlistPanel() {
    if (!wishlist || wishlist.length === 0) return <div className="p-4">No saved books yet.</div>;
    return (
      <div className="space-y-2">
        {wishlist.map((w) => (
          <div key={w.id} className="p-2 border rounded flex items-center justify-between">
            <div>
              <div className="text-sm font-medium">{w.title}</div>
              <div className="text-xs text-slate-600">{(w.authors || []).join(", ")}</div>
            </div>
            <div className="flex items-center gap-2">
              <button onClick={() => setSelected(w)} className="px-2 py-1 border rounded">Details</button>
              <button onClick={() => toggleWishlist(w)} className="px-2 py-1 border rounded">Remove</button>
            </div>
          </div>
        ))}
      </div>
    );
  }

  function PreviewModal({ item, onClose }) {
    if (!item) return null;
    return (
      <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/40 p-4">
        <motion.div initial={{ scale: 0.9, opacity: 0 }} animate={{ scale: 1, opacity: 1 }} className="max-w-3xl w-full bg-white rounded-lg p-6">
          <div className="flex justify-between items-start gap-4">
            <div>
              <h2 className="text-xl font-bold">{item.title}</h2>
              <p className="text-sm text-slate-600">{(item.authors || []).join(", ")}</p>
            </div>
            <div className="flex items-center gap-2">
              <button onClick={() => navigator.clipboard?.writeText(formatCitation(item, "APA"))} className="px-3 py-2 border rounded">Copy APA</button>
              <button onClick={() => onClose()} className="px-3 py-2 border rounded">Close</button>
            </div>
          </div>

          <div className="mt-4 grid grid-cols-1 md:grid-cols-3 gap-4">
            <div className="col-span-1">
              <div className="w-full h-64 bg-slate-100 rounded flex items-center justify-center">
                {item.thumbnail ? <img src={item.thumbnail} alt={item.title} className="object-contain h-full" /> : <BookOpen />}
              </div>
              <div className="mt-3 space-y-2">
                <div className="text-xs">Publisher: {item.publisher}</div>
                <div className="text-xs">Published: {item.publishedDate}</div>
                <div className="text-xs">Source: {item.source}</div>
              </div>
            </div>
            <div className="col-span-2">
              <p className="text-sm">{item.description || "No description available."}</p>

              <div className="mt-4">
                <a href={item.previewLink || item.infoLink} target="_blank" rel="noreferrer" className="underline">Open page</a>
              </div>

              <div className="mt-4 flex gap-2">
                <button onClick={() => toggleWishlist(item)} className="px-3 py-2 border rounded">{isInWishlist(item.id) ? "Remove from wishlist" : "Save to wishlist"}</button>
                <button onClick={() => { const t = formatCitation(item, "APA"); navigator.clipboard?.writeText(t); }} className="px-3 py-2 border rounded flex items-center gap-2"><Download size={14} /> Copy citation</button>
              </div>
            </div>
          </div>
        </motion.div>
      </div>
    );
  }

  // ---------- UI Layout ----------
  return (
    <div className="min-h-screen p-6 bg-slate-50 font-sans">
      <div className="max-w-7xl mx-auto">
        <header className="flex items-center justify-between mb-6">
          <div>
            <h1 className="text-2xl font-bold">Book Finder</h1>
            <p className="text-sm text-slate-600">Search books across Google Books & Open Library. Save to wishlist, preview, and copy citations.</p>
          </div>
          <div className="flex items-center gap-3">
            <button className="px-3 py-2 border rounded">Import CSV</button>
            <button className="px-3 py-2 border rounded">Export</button>
          </div>
        </header>

        <main className="grid grid-cols-1 lg:grid-cols-4 gap-6">
          <aside className="lg:col-span-1 space-y-4">
            <div className="p-4 bg-white rounded-lg border">
              <SearchBar />
            </div>

            <FiltersPanel />

            <div className="p-4 bg-white rounded-lg border">
              <h3 className="font-semibold mb-2">Wishlist</h3>
              <WishlistPanel />
            </div>
          </aside>

          <section className="lg:col-span-3">
            <div className="mb-4 flex items-center justify-between">
              <div className="text-sm">Results (showing {results.length})</div>
              <div className="flex gap-2 items-center">
                <label className="text-sm">Per page</label>
                <select value={perPage} disabled className="p-2 border rounded">
                  <option>{perPage}</option>
                </select>
              </div>
            </div>

            <ResultsGrid />

            <div className="mt-6 flex items-center justify-between">
              <div>
                <button onClick={() => doSearch(Math.max(0, page - 1))} className="px-3 py-2 border rounded mr-2">Prev</button>
                <button onClick={() => doSearch(page + 1)} className="px-3 py-2 border rounded">Next</button>
              </div>
              <div className="text-sm">Page: {page + 1}</div>
            </div>
          </section>
        </main>
      </div>

      <PreviewModal item={selected} onClose={() => setSelected(null)} />
    </div>
  );
}

/*
  README / Integration notes (inside the file for convenience):

  - This file is a single-file React component intended for prototyping.
  - To run:
      1) Create a Vite React app: npm create vite@latest my-app --template react
      2) Install dependencies: npm i framer-motion lucide-react
      3) Add Tailwind CSS: https://tailwindcss.com/docs/guides/vite
      4) Drop this file under src/components/BookFinder.jsx and import it in App.jsx: import BookFinder from "./components/BookFinder";
      5) npm run dev

  - API usage:
      * Google Books: no key required for basic queries, but you may supply a key for higher quota: append &key=YOUR_API_KEY to the Google query URL in buildGoogleBooksQuery.
      * Open Library: free endpoints. Respect rate limits.

  - Improvements you may want to add:
      * Pagination controls that read the total result count returned by APIs.
      * Authentication for saving user wishlist to a server or to sync across devices.
      * More robust citation generation (use citeproc or crosscite APIs).
      * Better error handling and retry/backoff.
*/
