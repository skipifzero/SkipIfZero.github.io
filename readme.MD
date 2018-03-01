# SkipIfZero's website
This is the repo for the official website of SkipIfZero. It's also hosted on Github and can be found at the following urls:

	http://skipifzero.com
	http://www.skipifzero.com

Any changes made to the master branch will immediately show up on the website. Feel free to make a pull request if you spot a typo or bug.

## Running locally

The website is powered by [Jekyll](jekyllrb.com/).

### macOS

1, `gem install jekyll bundler`

2, Had to install nokogiri with `sudo gem install nokogiri -n /Library/Ruby/Gems/2.3.0/gems/ -- --use-system-libraries --with-xml2-include=/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include/libxml2`

3, Inside this directory: `bundle install`, `bundle update`

4, Run using `bundle exec jekyll serve`

