FROM ruby:2.6.6-stretch

# Debian "stretch" sudah EOL, jadi repo apt normal (deb.debian.org & security.debian.org)
# sudah tidak menyediakan paketnya lagi. Semua paket stretch dipindah ke archive.debian.org.
# Baris sed di bawah ini:
#   1. Mengarahkan ulang sources.list ke archive.debian.org
#   2. Menghapus baris "stretch-updates" (repo ini TIDAK ada di archive, kalau
#      dibiarkan apt-get update akan gagal 404)
#   3. Mematikan pengecekan "Valid-Until" karena Release file di archive sudah
#      lewat tanggal kedaluwarsanya
#
# Package "shared-mime-info" ditambahkan untuk mendukung gem mimemagic >= 0.3.10
# (mimemagic 0.3.5 di-yank dari rubygems.org karena masalah lisensi GPL; versi 0.3.10
# membaca database mime dari sistem lewat package ini — lihat NOTES.md).
#
# Package "libimlib2-dev" ditambahkan untuk gem "rszr" (resize gambar/avatar).
#
# Fix SSL: root cert lama "DST Root CA X3" sudah expired (30 Sep 2021).
RUN sed -i \
      -e 's|deb.debian.org|archive.debian.org|g' \
      -e 's|security.debian.org|archive.debian.org|g' \
      -e '/stretch-updates/d' \
      /etc/apt/sources.list \
    && echo 'Acquire::Check-Valid-Until "false";' > /etc/apt/apt.conf.d/99no-check-valid-until \
    && apt-get update && apt-get install -y \
       nodejs git build-essential default-libmysqlclient-dev \
       imagemagick poppler-utils ca-certificates shared-mime-info \
       libimlib2-dev \
    && sed -i '/^mozilla\/DST_Root_CA_X3\.crt$/s/^/!/' /etc/ca-certificates.conf \
    && update-ca-certificates -f \
    && rm -rf /var/lib/apt/lists/*

# PENTING: sebelum COPY, folder app/ harus sudah di-patch (lihat NOTES.md):
#   - Gemfile: baris `gem 'tcr', git: ...` dihapus (dead git ref, gem test-only)
#   - Gemfile.lock: mimemagic (0.3.5) -> mimemagic (0.3.10) + nokogiri/rake sebagai sub-dependency
COPY app/ /opt/zammad
WORKDIR /opt/zammad

# Cek dulu BUNDLED WITH di app/Gemfile.lock sebelum build (harus cocok dengan angka di bawah).
RUN gem install bundler -v '~> 1.17' && \
    bundle install --full-index --without development test && \
    bundle exec rake assets:precompile RAILS_ENV=production

EXPOSE 3000
CMD ["bash", "-c", "rm -f tmp/pids/server.pid; bundle exec rails server -b 0.0.0.0"]
