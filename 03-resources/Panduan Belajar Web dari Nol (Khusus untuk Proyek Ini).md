Dokumen ini dibuat untuk kamu yang **belum paham sama sekali** tentang JavaScript, frontend, React, Next.js, dan ekosistemnya.
Targetnya: setelah membaca ini, kamu bisa mengerti “ini proyek apa”, “bagian mana ngatur apa”, dan “harus belajar urut dari mana”.

---

## 1) Gambaran Besar: Apa itu Website Modern?

Secara sederhana, aplikasi web biasanya terdiri dari:

1. **Frontend**
   - Bagian yang dilihat user di browser (tampilan, tombol, animasi, form).
   - Teknologi umum: HTML, CSS, JavaScript, React.

2. **Backend**
   - Bagian server (logika bisnis, autentikasi, database, API).
   - Teknologi umum: Node.js/Express, Go, Python, Java, dll.

3. **Database**
   - Tempat menyimpan data permanen.
   - Contoh: PostgreSQL, MySQL, SQLite, MongoDB.

### Analogi gampang
- Frontend = ruang tamu yang dilihat tamu.
- Backend = dapur + sistem kerja internal.
- Database = gudang/arsip.

---

## 2) JavaScript itu apa?

**JavaScript (JS)** adalah bahasa pemrograman utama di browser.

Dengan JS, kamu bisa:
- Mengubah isi halaman secara dinamis.
- Merespons klik tombol.
- Memanggil API.
- Mengatur state (data sementara di UI).

Contoh sederhana:

```js
const nama = "Inggit";
console.log(`Halo ${nama}`);
```

Konsep dasar yang wajib kamu kuasai:
- Variable (`let`, `const`)
- Function
- Object & Array
- Conditional (`if`)
- Loop
- Async (`Promise`, `async/await`)

---

## 3) TypeScript itu apa? Kenapa dipakai di proyek ini?

**TypeScript (TS)** adalah JavaScript + tipe data statis.

Manfaat TS:
- Error ketahuan lebih cepat saat coding (sebelum dijalankan).
- Editor bisa kasih autocomplete lebih akurat.
- Kode tim lebih aman dan mudah dirawat.

Contoh:

```ts
function sapa(nama: string): string {
  return `Halo ${nama}`;
}
```

Di proyek ini, file UI pakai `.tsx` (TypeScript + JSX).

---

## 4) React itu apa?

**React** adalah library untuk membangun UI dari komponen kecil.
### Konsep utama React
- **Component**: blok UI yang bisa dipakai ulang.
- **Props**: data masuk ke komponen.
- **State**: data internal komponen yang bisa berubah.
- **Rendering**: React menampilkan UI berdasarkan data saat ini.

Contoh pola komponen:

```tsx
export function Badge({ text }: { text: string }) {
  return <span>{text}</span>;
}
```

Di proyek kamu, hampir semua UI dipecah jadi komponen pada folder `src/components`.

---

## 5) Next.js vs React + Vite (yang dipakai proyek ini)

Kamu bilang belum paham Next.js. Ini ringkasannya:

### React + Vite (proyek kamu sekarang)
- Fokus utama: frontend SPA (Single Page Application).
- Cepat untuk development.
- Routing biasanya manual atau pakai react-router (di proyek ini belum dipakai).

### Next.js
- Framework di atas React.
- Punya fitur bawaan: routing berbasis folder, SSR, SSG, API route, optimasi image, dsb.
- Cocok untuk aplikasi yang butuh SEO kuat atau rendering server.

### Kesimpulan untuk proyek ini
- **Proyek ini bukan Next.js.**
- Proyek ini: **Vite + React + TypeScript + Tailwind**.
- Jadi fokus belajarmu sekarang: React, TypeScript, struktur komponen, dan data mapping.

---

## 6) Struktur Proyek Kamu (dibaca seperti peta)

- `src/main.tsx`
  - Titik masuk aplikasi React.
  - Me-render komponen `App` ke elemen root HTML.

- `src/App.tsx`
  - Komposer halaman utama.
  - Menyusun urutan section: Navbar → Hero → About → Experience → Projects → Footer.

- `src/components/*`
  - Kumpulan komponen UI.
  - Contoh:
    - `Navbar.tsx`: navigasi + toggle theme + mobile menu
    - `Hero.tsx`: heading utama + CTA button
    - `About.tsx`: menampilkan skill berdasarkan kategori
    - `Experience.tsx`: timeline pengalaman
    - `ProjectCard.tsx`: kartu project

- `src/data/portfolio.ts`
  - Data statis utama: `PROJECTS`, `EXPERIENCES`, `SKILLS`.
  - Banyak UI di-render dari data ini (data-driven UI).

- `src/index.css`
  - Tailwind + token tema (light/dark) berbasis CSS variables.

- `vite.config.ts`
  - Konfigurasi Vite, alias path, env variable, dan setting HMR.

---

## 7) Data Flow di Proyek Ini (sangat penting)

Pola data di proyek ini sederhana dan bagus untuk pemula:

1. Data disimpan terpusat di `src/data/portfolio.ts`.
2. Komponen section membaca data tersebut.
3. Komponen melakukan `.map(...)` untuk render list.

Contoh alur nyata:
- `PROJECTS` dibaca di `App.tsx`, lalu tiap item dikirim ke `ProjectCard`.
- `SKILLS` dibaca di `About.tsx`, difilter per kategori, lalu ditampilkan jadi badge.
- `EXPERIENCES` dibaca di `Experience.tsx` untuk membentuk timeline.

Manfaat pola ini:
- Ubah konten cukup di 1 file data.
- Komponen tetap bersih dan reusable.

---

## 8) Styling yang Dipakai

Proyek ini menggunakan **Tailwind CSS v4** + token semantik.

Contoh class penting:
- `bg-background`
- `text-foreground`
- `border-border`
- `text-muted-foreground`

Kenapa penting?
- Saat ganti tema (light/dark), style tetap konsisten karena memakai token, bukan warna hardcode sembarangan.

Ada helper `cn()` di `src/lib/utils.ts` untuk gabung class dengan aman (`clsx` + `tailwind-merge`).

---

## 9) Animasi, Icon, dan Theme

- Animasi: library `motion` (import dari `motion/react`).
- Icon: `lucide-react`.
- Theme dark/light: `next-themes`, dibungkus di komponen `ThemeProvider`.

Di `Navbar.tsx`, ada:
- toggle dark/light,
- state mobile menu,
- `AnimatePresence` untuk animasi menu mobile.

---

## 10) Cara Menjalankan Proyek

> Jalankan dari root folder proyek.

1. Install dependency:

```bash
npm install
```

2. Jalankan dev server:

```bash
npm run dev
```

3. Build production:

```bash
npm run build
```

4. Preview hasil build:

```bash
npm run preview
```

5. Type check:

```bash
npm run lint
```

Catatan:
- Script `lint` di proyek ini sebenarnya menjalankan TypeScript check (`tsc --noEmit`).

---

## 11) Istilah Penting yang Sering Bikin Bingung

- **SPA**: halaman tidak reload penuh tiap pindah bagian; UI berubah di sisi client.
- **Component-driven UI**: UI dibangun dari komponen kecil.
- **Props**: data dari parent ke child.
- **State**: data internal komponen.
- **Build**: proses mengubah source code jadi bundle siap deploy.
- **HMR**: perubahan kode langsung terlihat tanpa reload penuh.
- **Env variable**: konfigurasi dari environment (contoh API key).

---

## 12) Urutan Belajar yang Saya Sarankan (Praktis)

### Minggu 1: Fondasi JavaScript
- Variable, function, object, array, map/filter.
- Async/await dasar.

### Minggu 2: TypeScript + React dasar
- Type annotation.
- Komponen, props, state, event handling.

### Minggu 3: React di proyek ini
- Baca `main.tsx` → `App.tsx`.
- Baca tiap komponen di `src/components`.
- Pahami data di `src/data/portfolio.ts`.

### Minggu 4: Styling + refactor kecil
- Pahami token warna di `index.css`.
- Ubah konten project/experience.
- Tambah 1 section kecil dengan pola yang sama.

---

## 13) Latihan Kecil (Hands-on)

Lakukan berurutan:

1. Tambah 1 skill baru di `SKILLS`, lihat hasil di About section.
2. Tambah 1 project baru di `PROJECTS`, lihat kartu baru muncul.
3. Ubah teks di Hero dan Footer.
4. Buat komponen section baru sederhana (misal `Certifications`) dengan data array.
5. Pastikan style pakai token semantik, bukan hardcoded warna random.

---

## 14) Kapan Belajar Next.js?

Setelah kamu nyaman dengan:
- JavaScript dasar
- TypeScript dasar
- React komponen/props/state
- data mapping dan struktur project

Baru masuk ke Next.js agar tidak bingung antara:
- konsep React murni, dan
- fitur framework (routing/SSR/SSG/API route).

---

## 15) Ringkasan Super Singkat

- Proyek kamu saat ini adalah **frontend SPA** berbasis **Vite + React + TypeScript**.
- Bukan Next.js.
- Pola utama: **data di `src/data/portfolio.ts` → di-render oleh komponen**.
- Kalau mau cepat paham, pelajari dari alur: `main.tsx` → `App.tsx` → `components` → `data`.

---

Kalau kamu mau, saya bisa lanjut buat **versi lanjutan** dokumen ini berisi:
- cheat sheet JavaScript paling penting,
- cheat sheet React hooks (`useState`, `useEffect`),
- dan roadmap transisi dari proyek ini ke Next.js step-by-step.
