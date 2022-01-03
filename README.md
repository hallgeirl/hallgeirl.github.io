On Windows, edit the following file:
`C:\Ruby30-x64\lib\ruby\gems\3.0.0\gems\eventmachine-1.2.7-x64-mingw32\lib\eventmachine.rb` and add the line: `require 'em/pure_ruby'` to the top.

To install:
`bundle install` and `bundle update`

To run: 
`bundle exec jekyll serve .`
