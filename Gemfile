source "https://rubygems.org"

# Jekyll 버전 지정
gem "jekyll", "~> 4.4.1"

# 기본 테마
gem "minima", "~> 2.5"

# Jekyll 플러그인
group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.12"
  gem "jekyll-paginate"
  gem "jekyll-seo-tag"
  # Lunr.js 검색은 수동으로 구현할 것이므로 플러그인은 사용하지 않음
end

# Windows/JRuby 호환성을 위한 gem
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

# Windows 성능 향상
gem "wdm", "~> 0.1", :platforms => [:mingw, :x64_mingw, :mswin]

# JRuby 호환성
gem "http_parser.rb", "~> 0.6.0", :platforms => [:jruby]

# Jekyll 4.3.0 이상을 위한 웹서버
gem "webrick", "~> 1.8"