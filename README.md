import { useState, useEffect, useReducer } from "react";

// ── Palette ──────────────────────────────────────────────────────────────────
// Deep basil green  #1B4332   background header
// Warm rice white   #FAF7F0   page bg
// Chili red         #C0392B   expense accent
// Gold turmeric     #E8A020   income accent
// Soft smoke        #6B7280   muted text
// ─────────────────────────────────────────────────────────────────────────────

const CATEGORIES_INCOME = ["ขายหน้าร้าน", "เดลิเวอรี่", "จัดเลี้ยง", "อื่นๆ"];
const CATEGORIES_EXPENSE = ["วัตถุดิบ", "ค่าแรง", "ค่าน้ำค่าไฟ", "ค่าเช่า", "แพ็คเกจจิ้ง", "อื่นๆ"];

function formatThb(n) {
  return n.toLocaleString("th-TH", { minimumFractionDigits: 2, maximumFractionDigits: 2 });
}

function todayStr() {
  return new Date().toISOString().slice(0, 10);
}

function initialState() {
  try {
    const saved = localStorage.getItem("kaprao_entries");
    return saved ? JSON.parse(saved) : [];
  } catch { return []; }
}

function reducer(state, action) {
  let next;
  switch (action.type) {
    case "ADD":
      next = [action.entry, ...state];
      break;
    case "DELETE":
      next = state.filter((_, i) => i !== action.index);
      break;
    default:
      return state;
  }
  try { localStorage.setItem("kaprao_entries", JSON.stringify(next)); } catch {}
  return next;
}

export default function KapraoTracker() {
  const [entries, dispatch] = useReducer(reducer, null, initialState);
  const [tab, setTab] = useState("dashboard"); // dashboard | add | history
  const [type, setType] = useState("income");
  const [amount, setAmount] = useState("");
  const [category, setCategory] = useState("");
  const [note, setNote] = useState("");
  const [date, setDate] = useState(todayStr());
  const [filterMonth, setFilterMonth] = useState(todayStr().slice(0, 7));
  const [toast, setToast] = useState(null);

  const cats = type === "income" ? CATEGORIES_INCOME : CATEGORIES_EXPENSE;

  // auto-pick first category when type changes
  useEffect(() => { setCategory(cats[0]); }, [type]);

  const monthEntries = entries.filter(e => e.date.startsWith(filterMonth));
  const totalIncome = monthEntries.filter(e => e.type === "income").reduce((s, e) => s + e.amount, 0);
  const totalExpense = monthEntries.filter(e => e.type === "expense").reduce((s, e) => s + e.amount, 0);
  const profit = totalIncome - totalExpense;

  function showToast(msg) {
    setToast(msg);
    setTimeout(() => setToast(null), 2000);
  }

  function handleAdd() {
    const n = parseFloat(amount);
    if (!n || n <= 0) return showToast("กรุณากรอกจำนวนเงิน");
    if (!category) return showToast("เลือกหมวดหมู่ก่อนนะ");
    dispatch({
      type: "ADD",
      entry: { type, amount: n, category, note: note.trim(), date }
    });
    setAmount(""); setNote(""); setDate(todayStr());
    showToast(type === "income" ? "✅ บันทึกรายรับแล้ว" : "✅ บันทึกรายจ่ายแล้ว");
    setTab("dashboard");
  }

  const todayEntries = entries.filter(e => e.date === todayStr());
  const todayIncome = todayEntries.filter(e => e.type === "income").reduce((s, e) => s + e.amount, 0);
  const todayExpense = todayEntries.filter(e => e.type === "expense").reduce((s, e) => s + e.amount, 0);

  // Group history by date
  const grouped = monthEntries.reduce((acc, e, i) => {
    if (!acc[e.date]) acc[e.date] = [];
    acc[e.date].push({ ...e, _idx: entries.indexOf(e) });
    return acc;
  }, {});
  const sortedDates = Object.keys(grouped).sort((a, b) => b.localeCompare(a));

  return (
    <div style={s.root}>
      {/* HEADER */}
      <div style={s.header}>
        <div style={s.leaf}>🌿</div>
        <div>
          <div style={s.shopName}>ร้านข้าวกะเพรา</div>
          <div style={s.tagline}>บันทึกรายรับ-รายจ่าย</div>
        </div>
      </div>

      {/* TODAY QUICK SUMMARY */}
      <div style={s.todayBar}>
        <div style={s.todayCard}>
          <span style={s.todayLabel}>รายรับวันนี้</span>
          <span style={{ ...s.todayAmt, color: "#E8A020" }}>฿{formatThb(todayIncome)}</span>
        </div>
        <div style={s.todayDivider} />
        <div style={s.todayCard}>
          <span style={s.todayLabel}>รายจ่ายวันนี้</span>
          <span style={{ ...s.todayAmt, color: "#C0392B" }}>฿{formatThb(todayExpense)}</span>
        </div>
        <div style={s.todayDivider} />
        <div style={s.todayCard}>
          <span style={s.todayLabel}>กำไรวันนี้</span>
          <span style={{ ...s.todayAmt, color: todayIncome - todayExpense >= 0 ? "#1B7A4A" : "#C0392B" }}>
            ฿{formatThb(todayIncome - todayExpense)}
          </span>
        </div>
      </div>

      {/* TABS */}
      <div style={s.tabBar}>
        {[["dashboard", "📊 สรุป"], ["add", "➕ บันทึก"], ["history", "📋 ประวัติ"]].map(([k, label]) => (
          <button key={k} style={tab === k ? { ...s.tabBtn, ...s.tabActive } : s.tabBtn}
            onClick={() => setTab(k)}>{label}</button>
        ))}
      </div>

      <div style={s.content}>

        {/* ── DASHBOARD ── */}
        {tab === "dashboard" && (
          <div>
            <div style={s.monthRow}>
              <label style={s.monthLabel}>เดือน</label>
              <input type="month" value={filterMonth} style={s.monthInput}
                onChange={e => setFilterMonth(e.target.value)} />
            </div>

            <div style={s.summaryGrid}>
              <div style={{ ...s.summaryCard, borderTop: "4px solid #E8A020" }}>
                <div style={s.summaryIcon}>💰</div>
                <div style={s.summaryTitle}>รายรับ</div>
                <div style={{ ...s.summaryAmt, color: "#E8A020" }}>฿{formatThb(totalIncome)}</div>
              </div>
              <div style={{ ...s.summaryCard, borderTop: "4px solid #C0392B" }}>
                <div style={s.summaryIcon}>🧾</div>
                <div style={s.summaryTitle}>รายจ่าย</div>
                <div style={{ ...s.summaryAmt, color: "#C0392B" }}>฿{formatThb(totalExpense)}</div>
              </div>
              <div style={{ ...s.summaryCard, borderTop: `4px solid ${profit >= 0 ? "#1B7A4A" : "#C0392B"}`, gridColumn: "1 / -1" }}>
                <div style={s.summaryIcon}>{profit >= 0 ? "🎉" : "⚠️"}</div>
                <div style={s.summaryTitle}>กำไรสุทธิ</div>
                <div style={{ ...s.summaryAmt, fontSize: 28, color: profit >= 0 ? "#1B7A4A" : "#C0392B" }}>
                  ฿{formatThb(profit)}
                </div>
              </div>
            </div>

            {/* Category breakdown */}
            {["income", "expense"].map(t => {
              const tEntries = monthEntries.filter(e => e.type === t);
              const total = tEntries.reduce((s, e) => s + e.amount, 0);
              const catMap = tEntries.reduce((acc, e) => {
                acc[e.category] = (acc[e.category] || 0) + e.amount;
                return acc;
              }, {});
              const color = t === "income" ? "#E8A020" : "#C0392B";
              return (
                <div key={t} style={s.breakdownBox}>
                  <div style={s.breakdownTitle}>{t === "income" ? "💰 รายรับแยกหมวด" : "🧾 รายจ่ายแยกหมวด"}</div>
                  {Object.keys(catMap).length === 0
                    ? <div style={s.empty}>ยังไม่มีข้อมูล</div>
                    : Object.entries(catMap).sort((a, b) => b[1] - a[1]).map(([cat, amt]) => (
                      <div key={cat} style={s.catRow}>
                        <span style={s.catName}>{cat}</span>
                        <div style={s.barWrap}>
                          <div style={{ ...s.bar, width: `${(amt / total) * 100}%`, background: color }} />
                        </div>
                        <span style={{ ...s.catAmt, color }}>฿{formatThb(amt)}</span>
                      </div>
                    ))
                  }
                </div>
              );
            })}
          </div>
        )}

        {/* ── ADD ENTRY ── */}
        {tab === "add" && (
          <div style={s.form}>
            <div style={s.typeToggle}>
              <button style={type === "income" ? { ...s.toggleBtn, ...s.toggleIncome } : s.toggleBtn}
                onClick={() => setType("income")}>💰 รายรับ</button>
              <button style={type === "expense" ? { ...s.toggleBtn, ...s.toggleExpense } : s.toggleBtn}
                onClick={() => setType("expense")}>🧾 รายจ่าย</button>
            </div>

            <label style={s.label}>จำนวนเงิน (บาท)</label>
            <div style={s.amtWrap}>
              <span style={s.baht}>฿</span>
              <input type="number" placeholder="0.00" value={amount} style={s.amtInput}
                onChange={e => setAmount(e.target.value)} inputMode="decimal" />
            </div>

            <label style={s.label}>หมวดหมู่</label>
            <div style={s.catGrid}>
              {cats.map(c => (
                <button key={c} style={category === c
                  ? { ...s.catBtn, background: type === "income" ? "#E8A020" : "#C0392B", color: "#fff", borderColor: "transparent" }
                  : s.catBtn}
                  onClick={() => setCategory(c)}>{c}</button>
              ))}
            </div>

            <label style={s.label}>วันที่</label>
            <input type="date" value={date} style={s.input}
              onChange={e => setDate(e.target.value)} />

            <label style={s.label}>หมายเหตุ (ถ้ามี)</label>
            <input type="text" placeholder="เช่น ซื้อพริก ข่า ตะไคร้..." value={note} style={s.input}
              onChange={e => setNote(e.target.value)} />

            <button style={{ ...s.saveBtn, background: type === "income" ? "#E8A020" : "#C0392B" }}
              onClick={handleAdd}>
              บันทึก{type === "income" ? "รายรับ" : "รายจ่าย"}
            </button>
          </div>
        )}

        {/* ── HISTORY ── */}
        {tab === "history" && (
          <div>
            <div style={s.monthRow}>
              <label style={s.monthLabel}>เดือน</label>
              <input type="month" value={filterMonth} style={s.monthInput}
                onChange={e => setFilterMonth(e.target.value)} />
            </div>
            {sortedDates.length === 0
              ? <div style={{ ...s.empty, marginTop: 48 }}>ยังไม่มีรายการในเดือนนี้</div>
              : sortedDates.map(d => (
                <div key={d} style={s.dayGroup}>
                  <div style={s.dayHeader}>
                    {new Date(d + "T00:00:00").toLocaleDateString("th-TH", { weekday: "short", day: "numeric", month: "long" })}
                  </div>
                  {grouped[d].map(e => (
                    <div key={e._idx} style={s.entryRow}>
                      <div style={{ ...s.dot, background: e.type === "income" ? "#E8A020" : "#C0392B" }} />
                      <div style={s.entryInfo}>
                        <div style={s.entryCat}>{e.category}</div>
                        {e.note ? <div style={s.entryNote}>{e.note}</div> : null}
                      </div>
                      <div style={{ ...s.entryAmt, color: e.type === "income" ? "#E8A020" : "#C0392B" }}>
                        {e.type === "income" ? "+" : "-"}฿{formatThb(e.amount)}
                      </div>
                      <button style={s.delBtn} onClick={() => dispatch({ type: "DELETE", index: e._idx })}>✕</button>
                    </div>
                  ))}
                </div>
              ))
            }
          </div>
        )}
      </div>

      {toast && <div style={s.toast}>{toast}</div>}
    </div>
  );
}

// ── Styles ────────────────────────────────────────────────────────────────────
const s = {
  root: { fontFamily: "'Sarabun', 'Noto Sans Thai', sans-serif", background: "#FAF7F0", minHeight: "100vh", maxWidth: 480, margin: "0 auto", position: "relative", paddingBottom: 32 },
  header: { background: "#1B4332", padding: "20px 20px 16px", display: "flex", alignItems: "center", gap: 12 },
  leaf: { fontSize: 32 },
  shopName: { color: "#fff", fontSize: 20, fontWeight: 700, letterSpacing: 0.5 },
  tagline: { color: "#86EFAC", fontSize: 13, marginTop: 2 },
  todayBar: { background: "#fff", display: "flex", borderBottom: "1px solid #EEE", padding: "10px 0" },
  todayCard: { flex: 1, display: "flex", flexDirection: "column", alignItems: "center", gap: 2 },
  todayLabel: { fontSize: 11, color: "#9CA3AF", fontWeight: 600 },
  todayAmt: { fontSize: 15, fontWeight: 700 },
  todayDivider: { width: 1, background: "#E5E7EB", margin: "4px 0" },
  tabBar: { display: "flex", background: "#fff", borderBottom: "2px solid #E5E7EB" },
  tabBtn: { flex: 1, padding: "12px 4px", border: "none", background: "transparent", fontSize: 13, color: "#6B7280", cursor: "pointer", fontFamily: "inherit", fontWeight: 600 },
  tabActive: { color: "#1B4332", borderBottom: "2px solid #1B4332", marginBottom: -2 },
  content: { padding: "16px 16px 0" },
  monthRow: { display: "flex", alignItems: "center", gap: 10, marginBottom: 16 },
  monthLabel: { fontSize: 13, color: "#6B7280", fontWeight: 600 },
  monthInput: { border: "1px solid #D1D5DB", borderRadius: 8, padding: "6px 10px", fontSize: 14, fontFamily: "inherit", color: "#1F2937", background: "#fff" },
  summaryGrid: { display: "grid", gridTemplateColumns: "1fr 1fr", gap: 12, marginBottom: 20 },
  summaryCard: { background: "#fff", borderRadius: 12, padding: 16, boxShadow: "0 1px 4px rgba(0,0,0,0.06)" },
  summaryIcon: { fontSize: 22, marginBottom: 4 },
  summaryTitle: { fontSize: 12, color: "#6B7280", fontWeight: 600 },
  summaryAmt: { fontSize: 20, fontWeight: 800, marginTop: 4 },
  breakdownBox: { background: "#fff", borderRadius: 12, padding: 16, marginBottom: 16, boxShadow: "0 1px 4px rgba(0,0,0,0.06)" },
  breakdownTitle: { fontSize: 13, fontWeight: 700, color: "#374151", marginBottom: 12 },
  catRow: { display: "flex", alignItems: "center", gap: 8, marginBottom: 10 },
  catName: { width: 90, fontSize: 12, color: "#374151", flexShrink: 0 },
  barWrap: { flex: 1, height: 8, background: "#F3F4F6", borderRadius: 99, overflow: "hidden" },
  bar: { height: "100%", borderRadius: 99, transition: "width 0.4s ease" },
  catAmt: { fontSize: 12, fontWeight: 700, width: 80, textAlign: "right" },
  empty: { color: "#9CA3AF", fontSize: 13, textAlign: "center", padding: "8px 0" },
  form: { display: "flex", flexDirection: "column", gap: 12 },
  typeToggle: { display: "flex", borderRadius: 10, overflow: "hidden", border: "1px solid #E5E7EB" },
  toggleBtn: { flex: 1, padding: "12px 0", border: "none", background: "#F9FAFB", fontSize: 15, fontWeight: 700, cursor: "pointer", fontFamily: "inherit", color: "#6B7280", transition: "all 0.2s" },
  toggleIncome: { background: "#FEF3C7", color: "#92400E" },
  toggleExpense: { background: "#FEE2E2", color: "#991B1B" },
  label: { fontSize: 13, fontWeight: 600, color: "#374151" },
  amtWrap: { display: "flex", alignItems: "center", background: "#fff", border: "2px solid #D1D5DB", borderRadius: 10, overflow: "hidden" },
  baht: { padding: "0 12px", fontSize: 18, color: "#6B7280", fontWeight: 700 },
  amtInput: { flex: 1, border: "none", outline: "none", fontSize: 24, fontWeight: 800, padding: "10px 12px 10px 0", fontFamily: "inherit", color: "#1F2937", background: "transparent" },
  catGrid: { display: "flex", flexWrap: "wrap", gap: 8 },
  catBtn: { padding: "8px 14px", border: "2px solid #E5E7EB", borderRadius: 20, fontSize: 13, fontWeight: 600, cursor: "pointer", background: "#F9FAFB", color: "#374151", fontFamily: "inherit", transition: "all 0.15s" },
  input: { border: "1.5px solid #D1D5DB", borderRadius: 10, padding: "10px 14px", fontSize: 14, fontFamily: "inherit", color: "#1F2937", background: "#fff", outline: "none" },
  saveBtn: { color: "#fff", border: "none", borderRadius: 12, padding: "14px 0", fontSize: 16, fontWeight: 800, cursor: "pointer", fontFamily: "inherit", marginTop: 4, letterSpacing: 0.5, boxShadow: "0 4px 14px rgba(0,0,0,0.15)", transition: "opacity 0.15s" },
  dayGroup: { marginBottom: 16 },
  dayHeader: { fontSize: 12, fontWeight: 700, color: "#6B7280", padding: "4px 0 8px", borderBottom: "1px solid #E5E7EB", marginBottom: 8, textTransform: "uppercase", letterSpacing: 0.5 },
  entryRow: { display: "flex", alignItems: "center", gap: 10, background: "#fff", borderRadius: 10, padding: "10px 12px", marginBottom: 8, boxShadow: "0 1px 3px rgba(0,0,0,0.06)" },
  dot: { width: 10, height: 10, borderRadius: "50%", flexShrink: 0 },
  entryInfo: { flex: 1 },
  entryCat: { fontSize: 14, fontWeight: 700, color: "#1F2937" },
  entryNote: { fontSize: 12, color: "#9CA3AF", marginTop: 2 },
  entryAmt: { fontSize: 15, fontWeight: 800 },
  delBtn: { border: "none", background: "transparent", color: "#D1D5DB", fontSize: 14, cursor: "pointer", padding: "4px 6px", borderRadius: 6 },
  toast: { position: "fixed", bottom: 24, left: "50%", transform: "translateX(-50%)", background: "#1B4332", color: "#fff", padding: "10px 22px", borderRadius: 30, fontSize: 14, fontWeight: 600, boxShadow: "0 4px 20px rgba(0,0,0,0.2)", zIndex: 999 },
};
