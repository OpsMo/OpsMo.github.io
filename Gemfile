# frozen_string_literal: true

source "https://rubygems.org"

#gem "jekyll-theme-chirpy", "~> 6.4", ">= 6.4.2"
gem "jekyll-theme-chirpy", "~> 7.0"

# Uncomment the line below if you are using GitHub Pages
# gem "github-pages"

group :jekyll_plugins do
  gem "jekyll-admin", "~> 0.11.0"
  gem "sinatra", "~> 2.0"
  gem "rack", "~> 2.0"
end


group :test do
  gem "html-proofer", "~> 4.4"
end

platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]
gem "http_parser.rb", "~> 0.6.0", :platforms => [:jruby]

gem "webrick"
