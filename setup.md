Check Chat GPT 



https://search.brave.com/search?q=update+ruby+version+to+3.2.0+ubuntu&summary=1&conversation=0900eda5b73d3b5dc41f730dd25fec3eef8aecho 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
source ~/.bashrc   

==========================================
Installation Steps:
==========================================
1. Install prerequisites

	# 1. Install dependencies
	sudo apt-get update
	sudo apt-get install libtool libyaml-dev build-essential libssl-dev libreadline-dev zlib1g-dev autoconf bison

	# 2. Install Ruby 3.2.0
	rbenv install 3.2.0

	# 3. Set it as the global version
	rbenv global 3.2.0

	# 4. Verify the installation
	ruby -v   

	# 5. Install bundler
		bundle install

	# 6. Install Jekyll
		echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
		echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
		echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
		source ~/.bashrc

		gem install jekyll bundler

	# 7. Verify the installation
		jekyll -v

	# 8. Execute the Site locally
		bundle exec jekyll serve

2. Auto-reload while editing (optional but useful)Auto-reload while editing (optional but useful)
	
	bundle exec jekyll serve --livereload
