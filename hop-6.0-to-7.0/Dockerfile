FROM ruby:3.4.8-bookworm

# Debian Bookworm - beda dari hop sebelumnya (Buster), belum tentu perlu redirect ke
# archive.debian.org (Bookworm rilis 2023, kemungkinan masih dalam masa dukungan normal).
# Coba dulu TANPA redirect saat build pertama - kalau apt-get update gagal, baru tambahkan
# pola sed archive.debian.org seperti hop-hop sebelumnya.
#
# PENTING - perubahan besar dari hop sebelumnya:
#   - Node.js wajib >=20 (naik dari >=16) - install via NodeSource setup_20.x
#   - Package manager JS berganti dari yarn ke PNPM (package.json: "packageManager":
#     "pnpm@10.29.1") - pakai corepack (bawaan Node >=16.9) untuk auto-install versi
#     pnpm yang tepat sesuai pin di package.json.
#   - mysql2/default-libmysqlclient-dev TIDAK DIPERLUKAN LAGI - MySQL sudah dihapus total
#     dari Zammad 7.0, cuma butuh libpq-dev untuk gem 'pg'.
RUN apt-get update && apt-get install -y \
       curl gnupg git build-essential libpq-dev \
       imagemagick poppler-utils ca-certificates shared-mime-info \
       pkg-config libimlib2-dev \
    && curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y nodejs \
    && corepack enable \
    && rm -rf /var/lib/apt/lists/*

# PENTING: sebelum COPY, folder app/ harus source Zammad 7.0.0 (git clone --branch 7.0.0).
COPY app/ /opt/zammad
WORKDIR /opt/zammad

# corepack akan otomatis fetch versi pnpm yang di-pin di package.json ("pnpm@10.29.1")
# saat pertama kali dipanggil - tidak perlu install manual versi tertentu.
RUN pnpm install --frozen-lockfile

RUN gem install bundler -v '2.6.9' && \
    bundle install --without development test

EXPOSE 3000
CMD ["bash", "-c", "rm -f tmp/pids/server.pid; bundle exec rake assets:precompile RAILS_ENV=production; bundle exec rails server -b 0.0.0.0"]
