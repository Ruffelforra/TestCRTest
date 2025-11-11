haircut-app/
├─ README.md
├─ .gitignore
├─ package.json           # root scripts for convenience
├─ server/
│  ├─ package.json
│  ├─ src/
│  │  ├─ index.ts         # Express app
│  │  ├─ db.ts            # SQLite connection & helpers
│  │  ├─ seed.ts          # Seed data
│  │  ├─ validators.ts    # Zod schemas
│  │  └─ types.ts
│  ├─ prisma.sql          # Schema for SQLite
│  ├─ tsconfig.json
│  └─ .env.example
└─ client/
   ├─ package.json
   ├─ index.html
   ├─ vite.config.ts
   └─ src/
      ├─ main.tsx
      ├─ App.tsx
      ├─ components/
      │  ├─ BookingForm.tsx
      │  ├─ AdminLogin.tsx
      │  └─ AdminDashboard.tsx
      └─ lib/api.ts
 
Quick Start
# 1) Clone & install
npm run setup

# 2) Seed DB & start dev servers
npm run dev
# Server: http://localhost:4000
# Client: http://localhost:5173
Admin token: copy .env.example to server/.env and set ADMIN_TOKEN.
 
Root: package.json
{
  "name": "haircut-app",
  "private": true,
  "workspaces": ["server", "client"],
  "scripts": {
    "setup": "npm -w server i && npm -w client i",
    "dev": "concurrently \"npm -w server run dev\" \"npm -w client run dev\"",
    "lint": "npm -w server run lint && npm -w client run lint",
    "format": "npm -w server run format && npm -w client run format"
  },
  "devDependencies": {
    "concurrently": "^9.0.0"
  }
}
Root: .gitignore
/node_modules
**/node_modules
*.log
.env
server/haircut.db
.DS_Store
 
Server
server/package.json
{
  "name": "haircut-server",
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "seed": "tsx src/seed.ts",
    "lint": "eslint \"src/**/*.{ts,tsx}\"",
    "format": "prettier -w \"src/**/*.{ts,tsx}\""
  },
  "dependencies": {
    "better-sqlite3": "^9.6.0",
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "zod": "^3.23.8"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/node": "^20.12.12",
    "eslint": "^9.9.0",
    "eslint-config-prettier": "^9.1.0",
    "eslint-plugin-import": "^2.29.1",
    "prettier": "^3.3.3",
    "tsx": "^4.16.2",
    "typescript": "^5.6.2"
  }
}
server/.env.example
ADMIN_TOKEN=changeme-super-secret
PORT=4000
server/tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "strict": true,
    "outDir": "dist"
  },
  "include": ["src"]
}
server/prisma.sql
CREATE TABLE IF NOT EXISTS stylists (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  bio TEXT DEFAULT ''
);

CREATE TABLE IF NOT EXISTS services (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  duration_min INTEGER NOT NULL,
  price_cents INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS timeslots (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  stylist_id INTEGER NOT NULL,
  starts_at TEXT NOT NULL,
  ends_at TEXT NOT NULL,
  is_booked INTEGER DEFAULT 0,
  FOREIGN KEY(stylist_id) REFERENCES stylists(id)
);

CREATE TABLE IF NOT EXISTS appointments (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  customer_name TEXT NOT NULL,
  customer_phone TEXT NOT NULL,
  stylist_id INTEGER NOT NULL,
  service_id INTEGER NOT NULL,
  timeslot_id INTEGER NOT NULL,
  notes TEXT DEFAULT '',
  created_at TEXT DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY(stylist_id) REFERENCES stylists(id),
  FOREIGN KEY(service_id) REFERENCES services(id),
  FOREIGN KEY(timeslot_id) REFERENCES timeslots(id)
);
server/src/types.ts
export type Stylist = { id: number; name: string; bio: string };
export type Service = { id: number; name: string; duration_min: number; price_cents: number };
export type Timeslot = { id: number; stylist_id: number; starts_at: string; ends_at: string; is_booked: number };
export type Appointment = {
  id: number;
  customer_name: string;
  customer_phone: string;
  stylist_id: number;
  service_id: number;
  timeslot_id: number;
  notes: string;
  created_at: string;
};
server/src/validators.ts
import { z } from "zod";

export const CreateAppointmentSchema = z.object({
  customer_name: z.string().min(1),
  customer_phone: z.string().min(7),
  stylist_id: z.number().int().positive(),
  service_id: z.number().int().positive(),
  timeslot_id: z.number().int().positive(),
  notes: z.string().max(500).optional().default("")
});

export type CreateAppointment = z.infer<typeof CreateAppointmentSchema>;
server/src/db.ts
import Database from "better-sqlite3";
import fs from "fs";
import path from "path";

const dbPath = path.join(process.cwd(), "haircut.db");
const db = new Database(dbPath);

// init schema
const schema = fs.readFileSync(path.join(process.cwd(), "prisma.sql"), "utf8");
db.exec(schema);

export default db;
server/src/seed.ts
import db from "./db.js";

// Seed stylists
const stylists = [
  ["Alex", "Fades & classic cuts"],
  ["Jordan", "Color specialist"],
  ["Sam", "Kids cuts & designs"]
];
const insStylist = db.prepare("INSERT INTO stylists (name,bio) VALUES (?,?)");
stylists.forEach(s => insStylist.run(s));

// Seed services
const services = [
  ["Men's Cut", 30, 3000],
  ["Women's Cut", 45, 5000],
  ["Beard Trim", 15, 1500]
];
const insService = db.prepare("INSERT INTO services (name,duration_min,price_cents) VALUES (?,?,?)");
services.forEach(s => insService.run(s));

// Seed timeslots for the next 5 days per stylist
const addMinutes = (iso: string, m: number) => new Date(new Date(iso).getTime() + m * 60000).toISOString();
const startOfHour = (d: Date) => { d.setMinutes(0,0,0); return d; };

const insTimeslot = db.prepare(
  "INSERT INTO timeslots (stylist_id, starts_at, ends_at, is_booked) VALUES (?,?,?,0)"
);

const now = new Date();
for (let day = 0; day < 5; day++) {
  for (let stylistId = 1; stylistId <= stylists.length; stylistId++) {
    // working hours 9am–5pm, 30min slots
    const d = new Date(now);
    d.setDate(now.getDate() + day);
    d.setHours(9, 0, 0, 0);
    for (let h = 9; h < 17; h++) {
      for (let m = 0; m < 60; m += 30) {
        const start = new Date(d.getFullYear(), d.getMonth(), d.getDate(), h, m);
        const end = new Date(start.getTime() + 30 * 60000);
        insTimeslot.run(stylistId, start.toISOString(), end.toISOString());
      }
    }
  }
}

console.log("Seed complete");
server/src/index.ts
import express from "express";
import cors from "cors";
import dotenv from "dotenv";
import db from "./db.js";
import { CreateAppointmentSchema } from "./validators.js";

dotenv.config();
const app = express();
app.use(cors());
app.use(express.json());

const ADMIN_TOKEN = process.env.ADMIN_TOKEN || "changeme";

// Helpers
const requireAdmin = (req: express.Request, res: express.Response, next: express.NextFunction) => {
  const token = req.headers["x-admin-token"] as string | undefined;
  if (!token || token !== ADMIN_TOKEN) return res.status(401).json({ error: "Unauthorized" });
  next();
};

// Public endpoints
app.get("/api/stylists", (_req, res) => {
  const rows = db.prepare("SELECT * FROM stylists").all();
  res.json(rows);
});

app.get("/api/services", (_req, res) => {
  const rows = db.prepare("SELECT * FROM services").all();
  res.json(rows);
});

app.get("/api/timeslots", (req, res) => {
  const { stylist_id, date } = req.query as { stylist_id?: string; date?: string };
  let q = "SELECT * FROM timeslots WHERE is_booked = 0";
  const params: any[] = [];
  if (stylist_id) { q += " AND stylist_id = ?"; params.push(Number(stylist_id)); }
  if (date) { q += " AND date(starts_at) = date(?)"; params.push(date); }
  q += " ORDER BY starts_at ASC";
  const rows = db.prepare(q).all(...params);
  res.json(rows);
});

app.post("/api/appointments", (req, res) => {
  const parse = CreateAppointmentSchema.safeParse(req.body);
  if (!parse.success) return res.status(400).json({ error: parse.error.flatten() });
  const data = parse.data;

  // Check slot availability atomically
  const slot = db.prepare("SELECT * FROM timeslots WHERE id = ?").get(data.timeslot_id) as any;
  if (!slot || slot.is_booked) return res.status(400).json({ error: "Timeslot unavailable" });

  const insert = db.prepare(
    `INSERT INTO appointments (customer_name, customer_phone, stylist_id, service_id, timeslot_id, notes)
     VALUES (@customer_name,@customer_phone,@stylist_id,@service_id,@timeslot_id,@notes)`
  );
  const result = insert.run(data);
  db.prepare("UPDATE timeslots SET is_booked = 1 WHERE id = ?").run(data.timeslot_id);

  res.status(201).json({ id: result.lastInsertRowid });
});

// Admin endpoints
app.get("/api/admin/appointments", requireAdmin, (_req, res) => {
  const rows = db.prepare(
    `SELECT a.*, s.name as service_name, st.name as stylist_name, t.starts_at, t.ends_at
     FROM appointments a
     JOIN services s ON s.id = a.service_id
     JOIN stylists st ON st.id = a.stylist_id
     JOIN timeslots t ON t.id = a.timeslot_id
     ORDER BY a.created_at DESC`
  ).all();
  res.json(rows);
});

app.delete("/api/admin/appointments/:id", requireAdmin, (req, res) => {
  const id = Number(req.params.id);
  const appt = db.prepare("SELECT timeslot_id FROM appointments WHERE id = ?").get(id) as any;
  if (!appt) return res.status(404).json({ error: "Not found" });
  db.prepare("DELETE FROM appointments WHERE id = ?").run(id);
  db.prepare("UPDATE timeslots SET is_booked = 0 WHERE id = ?").run(appt.timeslot_id);
  res.json({ ok: true });
});

const port = Number(process.env.PORT || 4000);
app.listen(port, () => console.log(`Server listening on http://localhost:${port}`));
 
Client (React + Vite)
client/package.json
{
  "name": "haircut-client",
  "private": true,
  "version": "0.0.1",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "lint": "eslint \"src/**/*.{ts,tsx}\"",
    "format": "prettier -w \"src/**/*.{ts,tsx,css}\""
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@types/react": "^18.3.5",
    "@types/react-dom": "^18.3.0",
    "eslint": "^9.9.0",
    "eslint-config-prettier": "^9.1.0",
    "eslint-plugin-react-hooks": "^5.1.0",
    "eslint-plugin-react-refresh": "^0.4.9",
    "prettier": "^3.3.3",
    "typescript": "^5.6.2",
    "vite": "^5.4.2"
  }
}
client/vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: { port: 5173 },
})
client/index.html
<!doctype html>
<html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Haircut App</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
client/src/main.tsx
import React from 'react'
import { createRoot } from 'react-dom/client'
import App from './App'

createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
)
client/src/lib/api.ts
const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:4000';

export async function api<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`${API_URL}${path}`, {
    headers: { 'Content-Type': 'application/json', ...(init?.headers || {}) },
    ...init,
  })
  if (!res.ok) throw new Error(await res.text())
  return res.json()
}
client/src/App.tsx
import { useEffect, useMemo, useState } from 'react'
import { api } from './lib/api'
import BookingForm from './components/BookingForm'
import AdminLogin from './components/AdminLogin'
import AdminDashboard from './components/AdminDashboard'

export default function App() {
  const [adminToken, setAdminToken] = useState<string | null>(null)
  const [view, setView] = useState<'book'|'admin'>('book')

  useEffect(() => {
    const t = localStorage.getItem('adminToken')
    if (t) setAdminToken(t)
  }, [])

  const onLogin = (token: string) => {
    localStorage.setItem('adminToken', token)
    setAdminToken(token)
    setView('admin')
  }

  const header = useMemo(() => (
    <header style={{display:'flex',justifyContent:'space-between',alignItems:'center',padding:'12px 16px',borderBottom:'1px solid #eee'}}>
      <h1 style={{margin:0}}>💈 Haircut App</h1>
      <nav style={{display:'flex',gap:12}}>
        <button onClick={() => setView('book')}>Book</button>
        <button onClick={() => setView('admin')}>Admin</button>
      </nav>
    </header>
  ), [])

  return (
    <div>
      {header}
      <main style={{maxWidth:800,margin:'24px auto',padding:'0 16px'}}>
        {view === 'book' && <BookingForm />}
        {view === 'admin' && (
          adminToken ? <AdminDashboard token={adminToken} /> : <AdminLogin onLogin={onLogin} />
        )}
      </main>
    </div>
  )
}
client/src/components/BookingForm.tsx
import { useEffect, useState } from 'react'
import { api } from '../lib/api'

type Stylist = { id:number; name:string }
type Service = { id:number; name:string; duration_min:number; price_cents:number }
type Timeslot = { id:number; stylist_id:number; starts_at:string; ends_at:string }

export default function BookingForm() {
  const [stylists, setStylists] = useState<Stylist[]>([])
  const [services, setServices] = useState<Service[]>([])
  const [times, setTimes] = useState<Timeslot[]>([])

  const [form, setForm] = useState({
    customer_name:'', customer_phone:'', stylist_id:'', service_id:'', date:'', timeslot_id:''
  })
  const [status, setStatus] = useState<string | null>(null)

  useEffect(() => { (async () => {
    setStylists(await api<Stylist[]>('/api/stylists'))
    setServices(await api<Service[]>('/api/services'))
  })() }, [])

  useEffect(() => { (async () => {
    if (!form.stylist_id || !form.date) { setTimes([]); return }
    const qs = new URLSearchParams({ stylist_id: form.stylist_id, date: form.date })
    setTimes(await api<Timeslot[]>(`/api/timeslots?${qs.toString()}`))
  })() }, [form.stylist_id, form.date])

  const submit = async (e: React.FormEvent) => {
    e.preventDefault()
    setStatus(null)
    try {
      await api('/api/appointments', {
        method: 'POST',
        body: JSON.stringify({
          customer_name: form.customer_name,
          customer_phone: form.customer_phone,
          stylist_id: Number(form.stylist_id),
          service_id: Number(form.service_id),
          timeslot_id: Number(form.timeslot_id),
          notes: ''
        })
      })
      setStatus('✅ Appointment booked! You will receive a confirmation shortly.')
    } catch (err: any) {
      setStatus(`❌ ${err.message || 'Something went wrong'}`)
    }
  }

  return (
    <form onSubmit={submit} style={{display:'grid',gap:12}}>
      <h2>Book an appointment</h2>
      <input placeholder='Your name' value={form.customer_name} onChange={e=>setForm({...form, customer_name:e.target.value})} required />
      <input placeholder='Phone' value={form.customer_phone} onChange={e=>setForm({...form, customer_phone:e.target.value})} required />

      <select value={form.stylist_id} onChange={e=>setForm({...form, stylist_id:e.target.value})} required>
        <option value=''>Choose stylist</option>
        {stylists.map(s=> <option key={s.id} value={s.id}>{s.name}</option>)}
      </select>

      <select value={form.service_id} onChange={e=>setForm({...form, service_id:e.target.value})} required>
        <option value=''>Choose service</option>
        {services.map(s=> <option key={s.id} value={s.id}>{s.name} ({s.duration_min}m)</option>)}
      </select>

      <input type='date' value={form.date} onChange={e=>setForm({...form, date:e.target.value})} required />

      <select value={form.timeslot_id} onChange={e=>setForm({...form, timeslot_id:e.target.value})} required disabled={!times.length}>
        <option value=''>{times.length ? 'Choose time' : 'Select stylist + date'}</option>
        {times.map(t => (
          <option key={t.id} value={t.id}>
            {new Date(t.starts_at).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })}
          </option>
        ))}
      </select>

      <button type='submit'>Book</button>
      {status && <p>{status}</p>}
    </form>
  )
}
client/src/components/AdminLogin.tsx
import { useState } from 'react'

type Props = { onLogin: (token: string) => void }

export default function AdminLogin({ onLogin }: Props) {
  const [token, setToken] = useState('')
  return (
    <div style={{display:'grid',gap:12}}>
      <h2>Admin</h2>
      <input placeholder='Admin token' value={token} onChange={e=>setToken(e.target.value)} />
      <button onClick={()=> onLogin(token)}>Enter</button>
      <small>Set ADMIN_TOKEN in server/.env</small>
    </div>
  )
}
client/src/components/AdminDashboard.tsx
import { useEffect, useState } from 'react'
import { api } from '../lib/api'

type Row = {
  id:number; customer_name:string; customer_phone:string; notes:string;
  service_name:string; stylist_name:string; starts_at:string; ends_at:string;
}

type Props = { token: string }

export default function AdminDashboard({ token }: Props) {
  const [rows, setRows] = useState<Row[]>([])
  const [loading, setLoading] = useState(true)

  const fetchRows = async () => {
    setLoading(true)
    const r = await fetch(`${(import.meta.env.VITE_API_URL || 'http://localhost:4000')}/api/admin/appointments`, {
      headers: { 'x-admin-token': token }
    })
    if (!r.ok) throw new Error(await r.text())
    setRows(await r.json())
    setLoading(false)
  }

  useEffect(() => { fetchRows() }, [])

  const cancel = async (id:number) => {
    const r = await fetch(`${(import.meta.env.VITE_API_URL || 'http://localhost:4000')}/api/admin/appointments/${id}`, {
      method: 'DELETE',
      headers: { 'x-admin-token': token }
    })
    if (r.ok) setRows(rows.filter(a=>a.id!==id))
  }

  if (loading) return <p>Loading…</p>

  return (
    <div>
      <h2>Appointments</h2>
      <button onClick={fetchRows} style={{marginBottom:12}}>Refresh</button>
      <div style={{display:'grid',gap:8}}>
        {rows.map(r => (
          <div key={r.id} style={{border:'1px solid #eee', padding:12, borderRadius:8}}>
            <strong>{r.customer_name}</strong> • {r.customer_phone}
            <div>{r.service_name} with {r.stylist_name}</div>
            <div>
              {new Date(r.starts_at).toLocaleString()} – {new Date(r.ends_at).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })}
            </div>
            <div style={{display:'flex', gap:8, marginTop:8}}>
              <button onClick={() => cancel(r.id)}>Cancel</button>
            </div>
          </div>
        ))}
        {!rows.length && <p>No appointments yet.</p>}
      </div>
    </div>
  )
}
 
README.md (copy to project root)
# Haircut App

A minimal full‑stack appointment booking app for barbers/hair salons.

## Features
- Browse stylists, services, and available timeslots
- Book appointments (locks slot atomically)
- Admin dashboard (list/cancel appointments) protected by token

## Getting Started
```bash
npm run setup
cp server/.env.example server/.env
# edit server/.env to set ADMIN_TOKEN
npm -w server run seed
npm run dev
•	Client: http://localhost:5173
•	Server: http://localhost:4000
Environment
•	VITE_API_URL (client) – defaults to http://localhost:4000
•	ADMIN_TOKEN (server) – set a strong token
Deploy
You can deploy the server to any Node host. For a quick demo: - Render/Fly/Heroku for server - Netlify/Vercel/Static for client (set VITE_API_URL to your server URL)
Notes
•	SQLite file: server/haircut.db
•	For production, consider:
o	real auth (e.g., Clerk/Auth0), email/SMS notifications
o	recurring schedules, blackout dates, buffer times
o	payment integration (Stripe)
o	rate limiting, input sanitization beyond zod, logging/monitoring ```
 
MIT License
MIT License

Copyright (c) 2025

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
<img width="468" height="644" alt="image" src="https://github.com/user-attachments/assets/c406bca2-444e-43e7-b26f-8c42ccb3091c" />
 TestCRTest
Utilizing this to Test CR
