FROM ruby:2.7.4-buster

# Debian Buster sudah EOL, jadi repo apt normal sudah tidak menyediakan paketnya lagi.
# Pola sama seperti hop 3.4.0->4.0 (redirect ke archive.debian.org), tapi PERLU DICEK ULANG
# saat build pertama: nama baris repo update/security di sources.list Buster berbeda dari
# Stretch (bukan "stretch-updates", cek isi /etc/apt/sources.list dulu kalau apt-get update gagal).
RUN sed -i \
      -e 's|deb.debian.org|archive.debian.org|g' \
      -e 's|security.debian.org|archive.debian.org|g' \
      /etc/apt/sources.list \
    && echo 'Acquire::Check-Valid-Until "false";' > /etc/apt/apt.conf.d/99no-check-valid-until \
    && apt-get update && apt-get install -y \
       nodejs git build-essential default-libmysqlclient-dev \
       imagemagick poppler-utils ca-certificates shared-mime-info \
       libimlib2-dev \
    && rm -rf /var/lib/apt/lists/*

# PENTING: sebelum COPY, folder app/ harus source Zammad 5.0.0 (git clone --branch 5.0.0).
# Berbeda dari hop sebelumnya, mimemagic & tcr TIDAK perlu di-patch lagi (sudah dibenahi
# Zammad sendiri di Gemfile.lock 5.0.0) - lihat NOTES.md untuk detail.
COPY app/ /opt/zammad
WORKDIR /opt/zammad

# Bundler versi 2.2.20 (BUNDLED WITH di Gemfile.lock 5.0.0) - BEDA dari hop sebelumnya yang
# pakai 1.17.x. Cek dulu app/Gemfile.lock kalau versi resmi berbeda dari ini.
#
# CATATAN PENTING: assets:precompile TIDAK dijalankan di sini (build time) seperti hop
# sebelumnya. Mulai Rails 6.0, initializer custom Zammad (models_searchable.rb) benar-benar
# melakukan query DB saat boot - bukan cuma butuh file config/database.yml ADA, tapi harus
# bisa CONNECT beneran. Saat build image, belum ada database asli yang bisa diakses (volume
# mount baru terjadi saat container run). Solusinya: assets:precompile dipindah ke CMD,
# dijalankan saat container start - saat itu database.yml asli sudah ter-mount dan MariaDB
# beneran bisa diakses lewat host.docker.internal.
RUN gem install bundler -v '2.2.20' && \
    bundle install --without development test

EXPOSE 3000
CMD ["bash", "-c", "rm -f tmp/pids/server.pid; bundle exec rake assets:precompile RAILS_ENV=production; bundle exec rails server -b 0.0.0.0"]
