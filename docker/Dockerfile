FROM ruby:3.4-slim

ENV BUNDLE_PATH=/usr/local/bundle \
    JEKYLL_ENV=development

WORKDIR /srv/jekyll

RUN apt-get -o Acquire::Retries=5 update \
  && apt-get -o Acquire::Retries=5 install -y --no-install-recommends \
    build-essential \
    git \
    nodejs \
  && rm -rf /var/lib/apt/lists/*

COPY Gemfile ./

RUN bundle install

EXPOSE 4000

CMD ["sh", "-c", "git config --global --add safe.directory /srv/jekyll && bundle exec jekyll serve --host 0.0.0.0 --livereload --force_polling"]
