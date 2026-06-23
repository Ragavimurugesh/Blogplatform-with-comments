# Blogplatform-with-comments
import { useState, useEffect } from "react";

// ── Seed Data ──────────────────────────────────────────────────────────────
const SEED_USERS = [
  { id: 1, username: "alice", email: "alice@example.com", password: "alice123", avatar: "A", bio: "Full-stack dev & coffee addict ☕" },
  { id: 2, username: "bob",   email: "bob@example.com",   password: "bob123",   avatar: "B", bio: "Open source enthusiast 🌐" },
];

const SEED_POSTS = [
  {
    id: 1, authorId: 1, title: "Getting Started with React Hooks",
    content: `React Hooks changed everything when they landed in React 16.8. Before hooks, you had to use class components just to hold local state or tap into lifecycle methods — now a simple function component can do it all.\n\nuseState gives you reactive local state. useEffect replaces componentDidMount, componentDidUpdate, and componentDidUnmount in one unified API. And custom hooks let you extract and reuse stateful logic across components without any HOC magic.\n\nStart small: convert one class component to a function and add useState. Once it clicks, you'll never look back.`,
    tags: ["React", "JavaScript", "Frontend"],
    createdAt: new Date(Date.now() - 3 * 86400000).toISOString(),
    likes: [2],
  },
  {
    id: 2, authorId: 2, title: "RESTful API Design Best Practices",
    content: `Building an API others want to use is harder than it looks. A few principles that make a real difference:\n\n**Use nouns, not verbs.** Your endpoints represent resources — /posts not /getPosts. HTTP verbs (GET, POST, PUT, DELETE) already carry the action.\n\n**Version from day one.** /api/v1/ in your path costs nothing now and saves you from breaking clients later.\n\n**Return consistent shapes.** Every response should wrap data the same way: { data, error, meta }. Predictability is your API's best feature.\n\n**Use proper status codes.** 201 for created, 204 for deleted, 400 for bad input, 401 for unauthenticated, 403 for unauthorized, 404 for not found. Don't 200 everything.`,
    tags: ["API", "Backend", "Node.js"],
    createdAt: new Date(Date.now() - 1 * 86400000).toISOString(),
    likes: [1],
  },
];

const SEED_COMMENTS = [
  { id: 1, postId: 1, authorId: 2, content: "Great intro! useReducer is worth covering next — really shines once your state logic gets complex.", createdAt: new Date(Date.now() - 2 * 86400000).toISOString() },
  { id: 2, postId: 1, authorId: 1, content: "Totally agree — I'll add a follow-up post on useReducer + useContext as a lightweight Redux alternative.", createdAt: new Date(Date.now() - 1 * 86400000).toISOString() },
  { id: 3, postId: 2, authorId: 1, content: "The versioning point is huge. Learned that lesson the hard way on a production API 😅", createdAt: new Date(Date.now() - 12 * 3600000).toISOString() },
];

// ── Helpers ────────────────────────────────────────────────────────────────
const timeAgo = (iso) => {
  const diff = (Date.now() - new Date(iso)) / 1000;
  if (diff < 60) return "just now";
  if (diff < 3600) return `${Math.floor(diff / 60)}m ago`;
  if (diff < 86400) return `${Math.floor(diff / 3600)}h ago`;
  return `${Math.floor(diff / 86400)}d ago`;
};

const avatarColor = (letter) => {
  const colors = { A: "#6366f1", B: "#10b981", C: "#f59e0b", D: "#ef4444", E: "#8b5cf6" };
  return colors[letter] || "#64748b";
};

// ── Sub-components ─────────────────────────────────────────────────────────
function Avatar({ letter, size = 36 }) {
  return (
    <div style={{
      width: size, height: size, borderRadius: "50%",
      background: avatarColor(letter), color: "#fff",
      display: "flex", alignItems: "center", justifyContent: "center",
      fontWeight: 700, fontSize: size * 0.42, flexShrink: 0,
      fontFamily: "'Inter', sans-serif",
    }}>{letter}</div>
  );
}

function Tag({ label }) {
  return (
    <span style={{
      padding: "2px 10px", borderRadius: 20, fontSize: 12, fontWeight: 600,
      background: "#ede9fe", color: "#7c3aed", border: "1px solid #ddd6fe",
    }}>{label}</span>
  );
}

function Toast({ msg, type }) {
  if (!msg) return null;
  return (
    <div style={{
      position: "fixed", bottom: 28, left: "50%", transform: "translateX(-50%)",
      background: type === "error" ? "#fef2f2" : "#f0fdf4",
      color: type === "error" ? "#b91c1c" : "#15803d",
      border: `1px solid ${type === "error" ? "#fecaca" : "#bbf7d0"}`,
      borderRadius: 10, padding: "10px 22px", fontWeight: 600, fontSize: 14,
      boxShadow: "0 4px 20px rgba(0,0,0,0.1)", zIndex: 9999,
      fontFamily: "'Inter', sans-serif",
    }}>{msg}</div>
  );
}

// ── Auth Modal ─────────────────────────────────────────────────────────────
function AuthModal({ mode: initMode, onAuth, onClose }) {
  const [mode, setMode] = useState(initMode);
  const [form, setForm] = useState({ username: "", email: "", password: "", bio: "" });
  const [err, setErr] = useState("");

  const set = (k) => (e) => setForm(f => ({ ...f, [k]: e.target.value }));

  const submit = () => {
    setErr("");
    if (mode === "login") {
      onAuth("login", form, setErr);
    } else {
      if (!form.username.trim() || !form.email.trim() || !form.password) {
        setErr("All fields are required."); return;
      }
      onAuth("register", form, setErr);
    }
  };

  const inp = (placeholder, key, type = "text") => (
    <input
      type={type} placeholder={placeholder} value={form[key]} onChange={set(key)}
      onKeyDown={(e) => e.key === "Enter" && submit()}
      style={{
        width: "100%", padding: "10px 14px", borderRadius: 8, border: "1.5px solid #e2e8f0",
        fontSize: 14, outline: "none", boxSizing: "border-box", fontFamily: "'Inter',sans-serif",
        marginBottom: 10, transition: "border .2s",
      }}
      onFocus={e => e.target.style.borderColor = "#7c3aed"}
      onBlur={e => e.target.style.borderColor = "#e2e8f0"}
    />
  );

  return (
    <div style={{
      position: "fixed", inset: 0, background: "rgba(0,0,0,0.45)", display: "flex",
      alignItems: "center", justifyContent: "center", zIndex: 1000, padding: 16,
    }} onClick={e => e.target === e.currentTarget && onClose()}>
      <div style={{
        background: "#fff", borderRadius: 16, padding: "36px 32px", width: "100%", maxWidth: 400,
        boxShadow: "0 20px 60px rgba(0,0,0,0.15)", fontFamily: "'Inter',sans-serif",
      }}>
        <h2 style={{ margin: "0 0 6px", fontSize: 22, fontWeight: 800, color: "#0f172a" }}>
          {mode === "login" ? "Welcome back" : "Create account"}
        </h2>
        <p style={{ margin: "0 0 24px", color: "#64748b", fontSize: 14 }}>
          {mode === "login" ? "Sign in to your account" : "Join the community"}
        </p>

        {mode === "register" && inp("Username", "username")}
        {inp("Email", "email", "email")}
        {inp("Password", "password", "password")}
        {mode === "register" && (
          <input
            placeholder="Short bio (optional)" value={form.bio} onChange={set("bio")}
            style={{
              width: "100%", padding: "10px 14px", borderRadius: 8, border: "1.5px solid #e2e8f0",
              fontSize: 14, outline: "none", boxSizing: "border-box", fontFamily: "'Inter',sans-serif",
              marginBottom: 10,
            }}
          />
        )}

        {err && <p style={{ color: "#dc2626", fontSize: 13, margin: "0 0 10px" }}>{err}</p>}

        <button onClick={submit} style={{
          width: "100%", padding: "11px", background: "#7c3aed", color: "#fff",
          border: "none", borderRadius: 8, fontWeight: 700, fontSize: 15, cursor: "pointer",
          marginBottom: 16,
        }}>
          {mode === "login" ? "Sign In" : "Register"}
        </button>

        <p style={{ textAlign: "center", fontSize: 13, color: "#64748b", margin: 0 }}>
          {mode === "login" ? "No account? " : "Already have one? "}
          <span onClick={() => { setMode(mode === "login" ? "register" : "login"); setErr(""); }}
            style={{ color: "#7c3aed", cursor: "pointer", fontWeight: 600 }}>
            {mode === "login" ? "Register" : "Sign in"}
          </span>
        </p>

        {mode === "login" && (
          <p style={{ fontSize: 12, color: "#94a3b8", textAlign: "center", marginTop: 14 }}>
            Demo: alice@example.com / alice123
          </p>
        )}
      </div>
    </div>
  );
}

// ── Post Editor ────────────────────────────────────────────────────────────
function PostEditor({ post, onSave, onCancel }) {
  const [title, setTitle] = useState(post?.title || "");
  const [content, setContent] = useState(post?.content || "");
  const [tagsStr, setTagsStr] = useState(post?.tags?.join(", ") || "");
  const [err, setErr] = useState("");

  const save = () => {
    if (!title.trim() || !content.trim()) { setErr("Title and content are required."); return; }
    onSave({ title: title.trim(), content: content.trim(), tags: tagsStr.split(",").map(t => t.trim()).filter(Boolean) });
  };

  return (
    <div style={{ fontFamily: "'Inter',sans-serif" }}>
      <h2 style={{ fontSize: 20, fontWeight: 800, margin: "0 0 20px", color: "#0f172a" }}>
        {post ? "Edit Post" : "New Post"}
      </h2>
      <input value={title} onChange={e => setTitle(e.target.value)} placeholder="Post title…"
        style={{
          width: "100%", padding: "12px 14px", borderRadius: 8, border: "1.5px solid #e2e8f0",
          fontSize: 18, fontWeight: 700, marginBottom: 12, boxSizing: "border-box",
          outline: "none", fontFamily: "inherit",
        }}
      />
      <textarea value={content} onChange={e => setContent(e.target.value)} placeholder="Write your post…"
        rows={10}
        style={{
          width: "100%", padding: "12px 14px", borderRadius: 8, border: "1.5px solid #e2e8f0",
          fontSize: 14, lineHeight: 1.7, boxSizing: "border-box", resize: "vertical",
          outline: "none", fontFamily: "inherit", marginBottom: 12,
        }}
      />
      <input value={tagsStr} onChange={e => setTagsStr(e.target.value)} placeholder="Tags (comma-separated): React, Node.js…"
        style={{
          width: "100%", padding: "10px 14px", borderRadius: 8, border: "1.5px solid #e2e8f0",
          fontSize: 13, boxSizing: "border-box", marginBottom: 14, outline: "none", fontFamily: "inherit",
        }}
      />
      {err && <p style={{ color: "#dc2626", fontSize: 13, marginBottom: 10 }}>{err}</p>}
      <div style={{ display: "flex", gap: 10 }}>
        <button onClick={save} style={{
          padding: "10px 24px", background: "#7c3aed", color: "#fff", border: "none",
          borderRadius: 8, fontWeight: 700, cursor: "pointer", fontSize: 14,
        }}>Publish</button>
        <button onClick={onCancel} style={{
          padding: "10px 24px", background: "#f1f5f9", color: "#475569", border: "none",
          borderRadius: 8, fontWeight: 600, cursor: "pointer", fontSize: 14,
        }}>Cancel</button>
      </div>
    </div>
  );
}

// ── Post Card ──────────────────────────────────────────────────────────────
function PostCard({ post, users, comments, currentUser, onLike, onOpen }) {
  const author = users.find(u => u.id === post.authorId);
  const commentCount = comments.filter(c => c.postId === post.id).length;
  const liked = currentUser && post.likes.includes(currentUser.id);
  const preview = post.content.slice(0, 180) + (post.content.length > 180 ? "…" : "");

  return (
    <div onClick={onOpen} style={{
      background: "#fff", borderRadius: 14, padding: "22px 24px",
      border: "1.5px solid #e8e8f0", cursor: "pointer",
      transition: "box-shadow .2s, border-color .2s",
      boxShadow: "0 1px 4px rgba(0,0,0,0.04)",
      fontFamily: "'Inter',sans-serif",
    }}
      onMouseEnter={e => { e.currentTarget.style.boxShadow = "0 6px 24px rgba(124,58,237,0.1)"; e.currentTarget.style.borderColor = "#c4b5fd"; }}
      onMouseLeave={e => { e.currentTarget.style.boxShadow = "0 1px 4px rgba(0,0,0,0.04)"; e.currentTarget.style.borderColor = "#e8e8f0"; }}
    >
      <div style={{ display: "flex", alignItems: "center", gap: 10, marginBottom: 12 }}>
        <Avatar letter={author?.avatar || "?"} size={34} />
        <div>
          <div style={{ fontWeight: 700, fontSize: 13, color: "#0f172a" }}>{author?.username}</div>
          <div style={{ fontSize: 12, color: "#94a3b8" }}>{timeAgo(post.createdAt)}</div>
        </div>
      </div>

      <h3 style={{ margin: "0 0 8px", fontSize: 18, fontWeight: 800, color: "#0f172a", lineHeight: 1.3 }}>{post.title}</h3>
      <p style={{ margin: "0 0 14px", color: "#475569", fontSize: 14, lineHeight: 1.6 }}>{preview}</p>

      <div style={{ display: "flex", gap: 6, flexWrap: "wrap", marginBottom: 14 }}>
        {post.tags.map(t => <Tag key={t} label={t} />)}
      </div>

      <div style={{ display: "flex", gap: 18, color: "#64748b", fontSize: 13 }}>
        <span onClick={e => { e.stopPropagation(); onLike(post.id); }} style={{
          cursor: "pointer", display: "flex", alignItems: "center", gap: 4,
          color: liked ? "#7c3aed" : "#64748b", fontWeight: liked ? 700 : 400,
        }}>
          {liked ? "♥" : "♡"} {post.likes.length}
        </span>
        <span style={{ display: "flex", alignItems: "center", gap: 4 }}>
          💬 {commentCount}
        </span>
      </div>
    </div>
  );
}

// ── Post Detail ────────────────────────────────────────────────────────────
function PostDetail({ post, users, comments, currentUser, onLike, onComment, onDeleteComment, onEdit, onDelete, onBack }) {
  const [commentText, setCommentText] = useState("");
  const [editing, setEditing] = useState(false);
  const author = users.find(u => u.id === post.authorId);
  const postComments = comments.filter(c => c.postId === post.id);
  const liked = currentUser && post.likes.includes(currentUser.id);
  const isAuthor = currentUser && currentUser.id === post.authorId;

  const submitComment = () => {
    if (!commentText.trim()) return;
    onComment(post.id, commentText.trim());
    setCommentText("");
  };

  if (editing) return (
    <PostEditor post={post} onSave={(data) => { onEdit(post.id, data); setEditing(false); }} onCancel={() => setEditing(false)} />
  );

  return (
    <div style={{ fontFamily: "'Inter',sans-serif" }}>
      <button onClick={onBack} style={{
        background: "none", border: "none", color: "#7c3aed", cursor: "pointer",
        fontWeight: 700, fontSize: 14, padding: 0, marginBottom: 24, display: "flex", alignItems: "center", gap: 6,
      }}>← Back to posts</button>

      <div style={{ background: "#fff", borderRadius: 16, padding: "28px 28px", border: "1.5px solid #e8e8f0", marginBottom: 20 }}>
        <div style={{ display: "flex", alignItems: "center", gap: 12, marginBottom: 20 }}>
          <Avatar letter={author?.avatar || "?"} size={44} />
          <div>
            <div style={{ fontWeight: 700, color: "#0f172a" }}>{author?.username}</div>
            <div style={{ fontSize: 13, color: "#94a3b8" }}>{timeAgo(post.createdAt)}</div>
          </div>
          {isAuthor && (
            <div style={{ marginLeft: "auto", display: "flex", gap: 8 }}>
              <button onClick={() => setEditing(true)} style={{
                padding: "6px 14px", borderRadius: 7, background: "#f1f5f9", border: "none",
                color: "#475569", cursor: "pointer", fontWeight: 600, fontSize: 13,
              }}>Edit</button>
              <button onClick={() => onDelete(post.id)} style={{
                padding: "6px 14px", borderRadius: 7, background: "#fef2f2", border: "none",
                color: "#dc2626", cursor: "pointer", fontWeight: 600, fontSize: 13,
              }}>Delete</button>
            </div>
          )}
        </div>

        <h1 style={{ margin: "0 0 12px", fontSize: 26, fontWeight: 900, color: "#0f172a", lineHeight: 1.2 }}>{post.title}</h1>

        <div style={{ display: "flex", gap: 6, flexWrap: "wrap", marginBottom: 20 }}>
          {post.tags.map(t => <Tag key={t} label={t} />)}
        </div>

        <div style={{ color: "#334155", fontSize: 15, lineHeight: 1.8, whiteSpace: "pre-line", marginBottom: 24 }}>
          {post.content}
        </div>

        <div style={{ borderTop: "1px solid #f1f5f9", paddingTop: 16 }}>
          <button onClick={() => onLike(post.id)} style={{
            background: liked ? "#ede9fe" : "#f8f9fa", border: `1.5px solid ${liked ? "#c4b5fd" : "#e2e8f0"}`,
            borderRadius: 20, padding: "6px 18px", cursor: "pointer", fontWeight: 700,
            color: liked ? "#7c3aed" : "#64748b", fontSize: 14,
          }}>
            {liked ? "♥" : "♡"} {post.likes.length} {post.likes.length === 1 ? "like" : "likes"}
          </button>
        </div>
      </div>

      {/* Comments */}
      <div style={{ background: "#fff", borderRadius: 16, padding: "24px 28px", border: "1.5px solid #e8e8f0" }}>
        <h3 style={{ margin: "0 0 20px", fontSize: 17, fontWeight: 800, color: "#0f172a" }}>
          💬 {postComments.length} {postComments.length === 1 ? "Comment" : "Comments"}
        </h3>

        {postComments.length === 0 && (
          <p style={{ color: "#94a3b8", fontSize: 14, marginBottom: 20 }}>No comments yet. Be the first!</p>
        )}

        {postComments.map(c => {
          const cAuthor = users.find(u => u.id === c.authorId);
          const canDelete = currentUser && (currentUser.id === c.authorId || currentUser.id === post.authorId);
          return (
            <div key={c.id} style={{
              display: "flex", gap: 12, marginBottom: 18, paddingBottom: 18,
              borderBottom: "1px solid #f1f5f9",
            }}>
              <Avatar letter={cAuthor?.avatar || "?"} size={34} />
              <div style={{ flex: 1 }}>
                <div style={{ display: "flex", alignItems: "center", gap: 8, marginBottom: 4 }}>
                  <span style={{ fontWeight: 700, fontSize: 13, color: "#0f172a" }}>{cAuthor?.username}</span>
                  <span style={{ fontSize: 12, color: "#94a3b8" }}>{timeAgo(c.createdAt)}</span>
                  {canDelete && (
                    <button onClick={() => onDeleteComment(c.id)} style={{
                      marginLeft: "auto", background: "none", border: "none", color: "#94a3b8",
                      cursor: "pointer", fontSize: 12,
                    }}>✕</button>
                  )}
                </div>
                <p style={{ margin: 0, color: "#334155", fontSize: 14, lineHeight: 1.6 }}>{c.content}</p>
              </div>
            </div>
          );
        })}

        {currentUser ? (
          <div style={{ display: "flex", gap: 10, marginTop: 8 }}>
            <Avatar letter={currentUser.avatar} size={34} />
            <div style={{ flex: 1 }}>
              <textarea value={commentText} onChange={e => setCommentText(e.target.value)}
                placeholder="Add a comment…" rows={3}
                style={{
                  width: "100%", padding: "10px 14px", borderRadius: 8, border: "1.5px solid #e2e8f0",
                  fontSize: 14, resize: "none", boxSizing: "border-box", outline: "none", fontFamily: "inherit",
                  marginBottom: 8,
                }}
              />
              <button onClick={submitComment} style={{
                padding: "8px 20px", background: "#7c3aed", color: "#fff", border: "none",
                borderRadius: 7, fontWeight: 700, cursor: "pointer", fontSize: 13,
              }}>Post Comment</button>
            </div>
          </div>
        ) : (
          <p style={{ color: "#64748b", fontSize: 14, background: "#f8f9fa", borderRadius: 8, padding: "12px 16px" }}>
            Sign in to leave a comment.
          </p>
        )}
      </div>
    </div>
  );
}

// ── Main App ───────────────────────────────────────────────────────────────
export default function BlogApp() {
  const [users, setUsers] = useState(SEED_USERS);
  const [posts, setPosts] = useState(SEED_POSTS);
  const [comments, setComments] = useState(SEED_COMMENTS);
  const [currentUser, setCurrentUser] = useState(null);
  const [authModal, setAuthModal] = useState(null); // "login" | "register" | null
  const [view, setView] = useState("feed"); // "feed" | "new" | { postId }
  const [search, setSearch] = useState("");
  const [toast, setToast] = useState({ msg: "", type: "ok" });
  const [nextId, setNextId] = useState({ post: 3, comment: 4, user: 3 });

  const showToast = (msg, type = "ok") => {
    setToast({ msg, type });
    setTimeout(() => setToast({ msg: "", type: "ok" }), 2800);
  };

  const handleAuth = (mode, form, setErr) => {
    if (mode === "login") {
      const user = users.find(u => u.email === form.email && u.password === form.password);
      if (!user) { setErr("Invalid email or password."); return; }
      setCurrentUser(user);
      setAuthModal(null);
      showToast(`Welcome back, ${user.username}!`);
    } else {
      if (users.find(u => u.email === form.email)) { setErr("Email already registered."); return; }
      if (users.find(u => u.username === form.username)) { setErr("Username taken."); return; }
      const newUser = {
        id: nextId.user, username: form.username, email: form.email,
        password: form.password, avatar: form.username[0].toUpperCase(), bio: form.bio || "",
      };
      setUsers(u => [...u, newUser]);
      setNextId(n => ({ ...n, user: n.user + 1 }));
      setCurrentUser(newUser);
      setAuthModal(null);
      showToast(`Account created! Welcome, ${newUser.username}!`);
    }
  };

  const handleNewPost = (data) => {
    const post = {
      id: nextId.post, authorId: currentUser.id,
      createdAt: new Date().toISOString(), likes: [], ...data,
    };
    setPosts(p => [post, ...p]);
    setNextId(n => ({ ...n, post: n.post + 1 }));
    setView({ postId: post.id });
    showToast("Post published!");
  };

  const handleEditPost = (postId, data) => {
    setPosts(p => p.map(post => post.id === postId ? { ...post, ...data } : post));
    showToast("Post updated!");
  };

  const handleDeletePost = (postId) => {
    setPosts(p => p.filter(post => post.id !== postId));
    setComments(c => c.filter(comment => comment.postId !== postId));
    setView("feed");
    showToast("Post deleted.");
  };

  const handleLike = (postId) => {
    if (!currentUser) { setAuthModal("login"); return; }
    setPosts(p => p.map(post => {
      if (post.id !== postId) return post;
      const liked = post.likes.includes(currentUser.id);
      return { ...post, likes: liked ? post.likes.filter(id => id !== currentUser.id) : [...post.likes, currentUser.id] };
    }));
  };

  const handleComment = (postId, content) => {
    const c = { id: nextId.comment, postId, authorId: currentUser.id, content, createdAt: new Date().toISOString() };
    setComments(prev => [...prev, c]);
    setNextId(n => ({ ...n, comment: n.comment + 1 }));
  };

  const handleDeleteComment = (commentId) => {
    setComments(c => c.filter(comment => comment.id !== commentId));
  };

  const filteredPosts = posts.filter(p =>
    !search || p.title.toLowerCase().includes(search.toLowerCase()) ||
    p.tags.some(t => t.toLowerCase().includes(search.toLowerCase())) ||
    users.find(u => u.id === p.authorId)?.username.toLowerCase().includes(search.toLowerCase())
  );

  const activePost = view?.postId ? posts.find(p => p.id === view.postId) : null;

  return (
    <div style={{ minHeight: "100vh", background: "#f8f7ff", fontFamily: "'Inter',sans-serif" }}>
      {/* Header */}
      <header style={{
        background: "#fff", borderBottom: "1.5px solid #ede9fe",
        position: "sticky", top: 0, zIndex: 100,
        boxShadow: "0 2px 12px rgba(124,58,237,0.06)",
      }}>
        <div style={{ maxWidth: 760, margin: "0 auto", padding: "0 16px", display: "flex", alignItems: "center", height: 58, gap: 16 }}>
          <div onClick={() => setView("feed")} style={{ cursor: "pointer", display: "flex", alignItems: "center", gap: 8 }}>
            <div style={{ width: 32, height: 32, borderRadius: 8, background: "#7c3aed", display: "flex", alignItems: "center", justifyContent: "center", color: "#fff", fontSize: 16 }}>✍️</div>
            <span style={{ fontWeight: 900, fontSize: 18, color: "#0f172a", letterSpacing: "-0.5px" }}>devlog</span>
          </div>

          <input value={search} onChange={e => setSearch(e.target.value)} placeholder="Search posts…"
            style={{
              flex: 1, padding: "7px 14px", borderRadius: 20, border: "1.5px solid #ede9fe",
              background: "#f8f7ff", fontSize: 13, outline: "none", fontFamily: "inherit",
            }}
          />

          {currentUser ? (
            <div style={{ display: "flex", gap: 8, alignItems: "center" }}>
              <button onClick={() => setView("new")} style={{
                padding: "7px 16px", background: "#7c3aed", color: "#fff", border: "none",
                borderRadius: 20, fontWeight: 700, cursor: "pointer", fontSize: 13, whiteSpace: "nowrap",
              }}>+ New Post</button>
              <div style={{ display: "flex", alignItems: "center", gap: 6, cursor: "pointer" }}
                onClick={() => { setCurrentUser(null); setView("feed"); showToast("Signed out."); }}>
                <Avatar letter={currentUser.avatar} size={30} />
                <span style={{ fontSize: 12, color: "#64748b", fontWeight: 600 }}>Sign out</span>
              </div>
            </div>
          ) : (
            <div style={{ display: "flex", gap: 8 }}>
              <button onClick={() => setAuthModal("login")} style={{
                padding: "7px 16px", background: "#f1f5f9", color: "#475569", border: "none",
                borderRadius: 20, fontWeight: 700, cursor: "pointer", fontSize: 13,
              }}>Sign In</button>
              <button onClick={() => setAuthModal("register")} style={{
                padding: "7px 16px", background: "#7c3aed", color: "#fff", border: "none",
                borderRadius: 20, fontWeight: 700, cursor: "pointer", fontSize: 13,
              }}>Register</button>
            </div>
          )}
        </div>
      </header>

      {/* Main */}
      <main style={{ maxWidth: 760, margin: "0 auto", padding: "28px 16px" }}>
        {view === "new" ? (
          <div style={{ background: "#fff", borderRadius: 16, padding: "28px", border: "1.5px solid #e8e8f0" }}>
            <PostEditor onSave={handleNewPost} onCancel={() => setView("feed")} />
          </div>
        ) : activePost ? (
          <PostDetail
            post={activePost} users={users} comments={comments} currentUser={currentUser}
            onLike={handleLike} onComment={handleComment} onDeleteComment={handleDeleteComment}
            onEdit={handleEditPost} onDelete={handleDeletePost} onBack={() => setView("feed")}
          />
        ) : (
          <>
            <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 20 }}>
              <div>
                <h1 style={{ margin: 0, fontSize: 22, fontWeight: 900, color: "#0f172a" }}>Latest Posts</h1>
                <p style={{ margin: "4px 0 0", fontSize: 13, color: "#94a3b8" }}>
                  {filteredPosts.length} {filteredPosts.length === 1 ? "post" : "posts"}
                  {search ? ` for "${search}"` : ""}
                </p>
              </div>
            </div>

            {filteredPosts.length === 0 ? (
              <div style={{ textAlign: "center", padding: "60px 0", color: "#94a3b8" }}>
                <div style={{ fontSize: 40, marginBottom: 12 }}>📭</div>
                <div style={{ fontWeight: 700, fontSize: 16, color: "#64748b" }}>No posts found</div>
                <div style={{ fontSize: 13, marginTop: 6 }}>Try a different search, or be the first to write!</div>
              </div>
            ) : (
              <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
                {filteredPosts.map(p => (
                  <PostCard
                    key={p.id} post={p} users={users} comments={comments}
                    currentUser={currentUser} onLike={handleLike}
                    onOpen={() => setView({ postId: p.id })}
                  />
                ))}
              </div>
            )}
          </>
        )}
      </main>

      {authModal && <AuthModal mode={authModal} onAuth={handleAuth} onClose={() => setAuthModal(null)} />}
      <Toast msg={toast.msg} type={toast.type} />
    </div>
  );
}
