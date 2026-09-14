FROM ruby:3.4.9-bookworm

# Hop terakhir - perubahan minor dari hop 6.0->7.0 (patch version Ruby 3.4.8->3.4.9,
# Rails 8.0.4->8.0.5.1). Base OS tetap Bookworm, tag Docker Hub 3.4.9-bookworm sudah
# dikonfirmasi ada, tidak perlu ganti base OS lagi.
#
# PERUBAHAN dari hop sebelumnya:
#   - Node.js wajib >=24 (naik dari >=20) - install via NodeSource setup_24.x
#   - pnpm tetap otomatis lewat corepack, cuma versi pin naik ke pnpm@10.33.3
#   - pkg-config + libimlib2-dev tetap wajib (gem rszr, pelajaran dari hop 6.0->7.0)
#   - CI=true tetap wajib sebelum pnpm install (pelajaran dari hop 6.0->7.0)
RUN apt-get update && apt-get install -y \
       curl gnupg git build-essential libpq-dev \
       imagemagick poppler-utils ca-certificates shared-mime-info \
       pkg-config libimlib2-dev \
    && curl -fsSL https://deb.nodesource.com/setup_24.x | bash - \
    && apt-get install -y nodejs \
    && corepack enable \
    && rm -rf /var/lib/apt/lists/*

# PENTING: sebelum COPY, folder app/ harus source Zammad 7.1.3 (git clone --branch 7.1.3).
COPY app/ /opt/zammad
WORKDIR /opt/zammad

ENV CI=true
RUN pnpm install --frozen-lockfile

RUN gem install bundler -v '2.6.9' && \
    bundle install --without development test

EXPOSE 3000
CMD ["bash", "-c", "rm -f tmp/pids/server.pid; bundle exec rake assets:precompile RAILS_ENV=production; bundle exec rails server -b 0.0.0.0"]
