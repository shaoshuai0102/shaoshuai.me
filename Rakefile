task :default => :build

task :build do
  puts "# jekyll build:"
  puts `jekyll build`
end

task :rsync do
  puts "# rsync:"
  outputs = `rsync -avze ssh --delete ./_site/ shaoshuai0102@shaoshuai.me:~/shaoshuai.me/`
  puts outputs
end

task :deploy => [:build, :rsync]
