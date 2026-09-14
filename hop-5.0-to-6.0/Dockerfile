FROM ruby:3.1.3-buster

# Debian Buster sudah EOL, redirect ke archive.debian.org (pola sama seperti hop sebelumnya).
#
# PENTING - beda dari hop sebelumnya: Zammad 6.0.0 mengadopsi Vite (package.json
# mensyaratkan "node": ">=16"), jadi nodejs bawaan apt Debian Buster (versi 10.x, terlalu
# lama) TIDAK cukup. Install Node.js 18.x LTS lewat NodeSource, plus yarn (dibutuhkan
# vite_ruby saat assets:precompile - tanpa ini gagal dengan "Errno::ENOENT: yarn").
RUN sed -i \
      -e 's|deb.debian.org|archive.debian.org|g' \
      -e 's|security.debian.org|archive.debian.org|g' \
      /etc/apt/sources.list \
    && echo 'Acquire::Check-Valid-Until "false";' > /etc/apt/apt.conf.d/99no-check-valid-until \
    && apt-get update && apt-get install -y \
       curl gnupg git build-essential default-libmysqlclient-dev \
       imagemagick poppler-utils ca-certificates shared-mime-info \
       libimlib2-dev \
    && curl -fsSL https://deb.nodesource.com/setup_18.x | bash - \
    && apt-get install -y nodejs \
    && npm install -g yarn \
    && rm -rf /var/lib/apt/lists/*

# PENTING: sebelum COPY, folder app/ harus source Zammad 6.0.0 (git clone --branch 6.0.0).
# tcr/vcr sudah require: false tanpa git: source - tidak perlu di-patch seperti hop 1.
COPY app/ /opt/zammad
WORKDIR /opt/zammad

# yarn install (populate node_modules untuk Vite) TIDAK butuh koneksi database, jadi aman
# dijalankan di build time (beda dari assets:precompile yang harus di runtime).
RUN yarn install --frozen-lockfile

# Bundler 2.4.1 (BUNDLED WITH di Gemfile.lock 6.0.0) - beda dari hop sebelumnya (2.2.20).
#
# assets:precompile TIDAK dijalankan di sini (build time) - pola sama seperti hop 4.0->5.0,
# dipindah ke CMD/command runtime karena Zammad's models_searchable.rb initializer butuh
# koneksi database asli saat boot (baru tersedia saat container benar-benar jalan).
RUN gem install bundler -v '2.4.1' && \
    bundle install --without development test

EXPOSE 3000
CMD ["bash", "-c", "rm -f tmp/pids/server.pid; bundle exec rake assets:precompile RAILS_ENV=production; bundle exec rails server -b 0.0.0.0"]
