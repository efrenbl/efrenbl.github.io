source 'https://rubygems.org'

# Use the latest GitHub Pages compatible Jekyll version
gem 'jekyll', '~> 3.9.0'
gem 'minima', '~> 2.5'

group :jekyll_plugins do
  gem 'jekyll-feed', '~> 0.12'
  gem 'jekyll-sitemap'
  gem 'jekyll-seo-tag'
end

# GitHub Pages compatibility - use specific version
gem "github-pages", "~> 228", group: :jekyll_plugins

# Windows and JRuby platform-specific gems
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

# Performance-booster for watching directories on Windows
gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]

# Lock `http_parser.rb` gem to compatible version on JRuby
gem "http_parser.rb", "~> 0.6.0", :platforms => [:jruby]

# Additional gems for GitHub Pages compatibility
gem "webrick", "~> 1.7" # Required for Ruby 3.0+
gem "csv", "~> 3.0" # Required for Ruby 3.4+
