# Membangun Blog Statis dengan Hugo dan LoveIt


Hugo merupakan static site generator yang ditulis dalam bahasa Go dan dikenal karena kecepatan build yang luar biasa. Dipadukan dengan tema LoveIt yang kaya fitur, kombinasi ini menjadi pilihan ideal bagi siapa pun yang ingin membangun blog statis modern tanpa mengorbankan performa maupun estetika. Artikel ini memandu proses pembangunan blog dari tahap instalasi hingga deployment.

<!--more-->

{{< admonition info >}}
Artikel ini mengasumsikan pembaca telah familiar dengan terminal dan memiliki pemahaman dasar tentang Git.
{{< /admonition >}}

## Arsitektur Hugo

Hugo berbeda dari platform blogging dinamis seperti WordPress. Tidak ada database, tidak ada runtime server-side, dan tidak ada panel administrasi berbasis web. Hugo bekerja dengan cara mengonversi file Markdown menjadi halaman HTML statis pada saat build time.

Beberapa karakteristik utama Hugo:

1. **Binary tunggal.** Hugo dikompilasi menjadi satu file binary tanpa dependensi eksternal. Tidak diperlukan Node.js, Ruby, Python, atau runtime lain untuk menjalankannya.
2. **Kecepatan build.** Hugo mampu membangun ribuan halaman dalam hitungan milidetik berkat engine template berbasis Go yang dikompilasi secara native.
3. **Live reload.** Development server Hugo mendeteksi perubahan file dan secara otomatis me-refresh browser tanpa intervensi manual.
4. **Organisasi konten.** Konten disusun dalam struktur direktori yang fleksibel dengan dukungan front matter YAML, TOML, atau JSON.

## Tema LoveIt

[LoveIt](https://github.com/dillonzq/LoveIt) adalah tema Hugo yang mengedepankan keseimbangan antara estetika dan fungsionalitas. Tema ini menyediakan serangkaian fitur yang biasanya hanya ditemukan pada platform blogging berbayar.

| Fitur | Deskripsi |
|---|---|
| **Mode gelap/terang** | Adaptasi otomatis mengikuti preferensi sistem operasi |
| **Pencarian klien** | Lunr.js menyediakan pencarian teks penuh tanpa backend |
| **Syntax highlighting** | Highlight kode dengan berbagai skema warna melalui Chroma |
| **Light gallery** | Lightbox gambar dengan dukungan zoom dan navigasi keyboard |
| **MathJax & Mermaid** | Rendering rumus matematika dan diagram secara native |
| **Multilingual** | Dukungan penuh multi-bahasa, termasuk Bahasa Indonesia |
| **SEO** | Open Graph, Twitter Cards, JSON-LD, dan sitemap otomatis |

## Instalasi Hugo

### Windows

```powershell
# Chocolatey
choco install hugo-extended

# Scoop
scoop install hugo-extended
```

### macOS

```bash
brew install hugo
```

### Linux

```bash
# Debian/Ubuntu
sudo apt install hugo
```

Verifikasi versi Hugo yang terpasang:

```bash
hugo version
# Output: hugo v0.145.0+extended ...
```

## Inisialisasi Proyek

```bash
hugo new site blog-kita
cd blog-kita
```

Struktur direktori yang terbentuk:

```
blog-kita/
├── archetypes/     # Template front matter
├── assets/         # CSS, JS yang diproses Hugo Pipes
├── content/        # Konten blog
├── data/           # File data (JSON/YAML/TOML)
├── layouts/        # Template kustom
├── static/         # File statis
├── themes/         # Tema
└── config.toml     # Konfigurasi
```

## Menambahkan Tema

```bash
git init
git submodule add https://github.com/dillonzq/LoveIt.git themes/LoveIt
```

Aktifkan tema di `config.toml`:

```toml
baseURL = "https://example.com/"
languageCode = "id-ID"
defaultContentLanguage = "id"
title = "Nama Blog"
theme = "LoveIt"
```

## Konfigurasi Dasar

```toml
[author]
  name = "Nama Penulis"
  email = "email@example.com"

[params]
  title = "Nama Blog"
  subtitle = "Deskripsi singkat"
  keywords = ["blog", "hugo", "tutorial"]

  [params.home]
    profileMode = "normal"
```

{{< admonition tip >}}
Dokumentasi lengkap parameter konfigurasi tersedia di situs resmi [LoveIt](https://hugoloveit.com/).
{{< /admonition >}}

## Penulisan Konten

### Membuat Post Baru

```bash
hugo new posts/judul-artikel/index.md
```

Hugo akan menghasilkan file `content/posts/judul-artikel/index.md` dengan front matter dari template archetype.

### Struktur Front Matter

```yaml
---
title: "Judul Artikel"
date: 2026-07-24T10:00:00+07:00
lastmod: 2026-07-24T10:00:00+07:00
draft: false
author: "anwareset"
description: "Deskripsi SEO"
tags: ["Hugo", "Blog"]
categories: ["Blog"]
lightgallery: true
---
```

### Konvensi Penulisan

Beberapa konvensi yang perlu diperhatikan saat menulis konten:

- Gunakan `<!--more-->` sebagai pemisah ringkasan. Konten di atas divider ini muncul di kartu pratinjau.
- Setiap blok kode harus menyertakan bahasa, misalnya ` ```bash `, ` ```yaml `, ` ```toml `.
- Hindari penggunaan tanda seru dan tanda tanya pada heading.
- Hindari penggunaan em dash (U+2014) dan en dash (U+2013) pada seluruh teks artikel.
- Akhiri setiap artikel dengan heading `## Summary` dan `## References`.

## Development Server

```bash
hugo server -D
```

Flag `-D` memastikan post dengan status `draft: true` ikut ditampilkan. Buka `http://localhost:1313/` pada browser.

Build production:

```bash
hugo --minify --environment production
```

Output build tersedia di direktori `public/`.

## Strategi Deployment

### GitHub Pages

Menggunakan GitHub Actions:

```yaml
name: Deploy Hugo
on:
  push:
    branches: [master]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: '0.145.0'
          extended: true
      - run: hugo --minify --environment production
      - uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

### Netlify

Hubungkan repositori Git ke Netlify, atur build command ke `hugo --minify` dan publish directory ke `public`. Netlify akan otomatis men-deploy setiap push ke branch yang dikonfigurasi.

### GitLab Pages

```yaml
pages:
  image: hugomods/hugo:0.145.0
  script:
    - hugo --minify
  artifacts:
    paths:
      - public
```

## Custom CSS dan Override

Untuk menyesuaikan tampilan tanpa memodifikasi file tema:

- `assets/css/_custom.scss`: menambahkan aturan CSS baru.
- `assets/css/_override.scss`: mengubah nilai variabel SCSS tema seperti warna dan font.

Pendekatan ini memastikan perubahan styling tidak hilang saat tema di-upgrade.

## Summary

Hugo dan LoveIt menghadirkan solusi blogging statis yang cepat, aman, dan kaya fitur. Proses setup yang sederhana, mulai dari instalasi binary tunggal, inisialisasi proyek, penambahan tema, hingga penulisan konten dalam Markdown, memungkinkan fokus pada hal yang paling penting: konten itu sendiri. Dengan berbagai opsi deployment seperti GitHub Pages, Netlify, dan GitLab Pages, blog dapat dihosting secara gratis dengan performa optimal.

## References

- [gohugo.io](https://gohugo.io/)
- [github.com/dillonzq/LoveIt](https://github.com/dillonzq/LoveIt)
- [hugoloveit.com](https://hugoloveit.com/)
- [gohugo.io/documentation](https://gohugo.io/documentation/)
- [github.com/peaceiris/actions-gh-pages](https://github.com/peaceiris/actions-gh-pages)
- [gohugo.io/hosting-and-deployment/hosting-on-netlify](https://gohugo.io/hosting-and-deployment/hosting-on-netlify/)

