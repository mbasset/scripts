require 'rake'
require 'rainbow'
require 'fileutils'

task :compute_code do
	count = 0
	while $<.gets
		count += 1 unless $_ =~ /^\s*($|#)/   # Any number of spaces followed by EOL or #
	end
	puts "#{count} lines of real code"
end

desc "Runs commands common to unroole version 2 code to help deploy new version
	of each gem. Specifiying ONLY environment variable will attempt to push only
	that gem.

	Avaliable ENV variables:
		ONLY=<engine_names>
		SKIP=<engine_names>
		NO_VERSION_BUMP=1 - Skips version bumping
		NO_BUNDLE=1 - Skips bundling
		NO_PUSH=1 - Skip git push
		MAJOR_VERSION_BUMP - Push the major version instead of bugfix
		MINOR_VERSION_BUMP - Push the minor version instead of bugfix
"
task :version_bump do
	engines_to_update = []
	engines_to_skip = []

	if ENV["ONLY"]
		engines_to_update = ENV["ONLY"].split(",")
	end

	if ENV["SKIP"]
		engines_to_skip = ENV["SKIP"].split(",")
	end

	Dir[*engines_to_update].uniq.each do |dir_name|
		if File.directory?(dir_name) && ![".", ".."].include?(dir_name)
			print "Trying folder: #{dir_name} ... ".color(:yellow)

			if engines_to_skip.include?(dir_name)
				puts "skip"
				next
			end

			system("cd #{dir_name} && git pull")

			rversion_file = `cd #{dir_name} && find . -name "version.rb"`
			version_file = File.expand_path(dir_name + rversion_file[1..-1].to_s).to_s.chomp if rversion_file

			if rversion_file && version_file && File.exists?(version_file) && File.file?(version_file)

				matched_version = nil
				new_version = nil
				File.open(version_file, File::RDWR) do |f|
					out = ""
					f.each do |line|
						old_version = line.match(/'?"?(\d+).(\d+).(\d+).?.*'?"?/)
						if old_version && old_version.size >= 4
							matched_version = old_version
							new_version = old_version
							unless ENV["NO_VERSION_BUMP"]
								major_ver = matched_version[1].to_i
								minor_ver = matched_version[2].to_i
								bugfix_ver = matched_version[3].to_i

								if ENV["MAJOR_VERSION_BUMP"]
									major_ver += 1
									minor_ver = 0
									bugfix_ver = 0
								elsif ENV["MINOR_VERSION_BUMP"]
									minor_ver += 1
									bugfix_ver = 0
								else
									bugfix_ver += 1
								end
								new_version = "#{major_ver}.#{minor_ver}.#{bugfix_ver}"
								line.gsub!(/\d+.\d+.\d+/, new_version)
							end
						end
						out << line
					end

					# re-write entire file
					f.pos = 0
					f.print out
					f.truncate(f.pos)
				end

				unless new_version
					raise Exception.new("Could not determine old gem version.")
				end

				# Git auto-commit and push
				Dir.chdir(dir_name)
				begin
					unless ENV["NO_VERSION_BUMP"]

						unless ENV["NO_BUNDLE"]
							system("mkdir -p tmp")
							system("bundle update > tmp/autobundle.log")
							system("git add Gemfile.lock")
						end

						system("git add .")
						system("git commit -m \" Version bump engine to #{new_version}\"")
						system("git tag v#{new_version}")

						unless ENV["NO_PUSH"]
							system("git push")
							system("git push --tags")
						end
					else

						system("bundle update > tmp/autobundle.log")
						system("git add Gemfile.lock")
						system("git add .")
						system("git commit -m \" Pushing new engine version #{new_version}\"")
						system("git tag v#{new_version}")

						unless ENV["NO_PUSH"]
							system("git push")
							system("git push --tags")
						end
					end

					system('find . -name \*.gem -print | xargs rm -f')
					system('find . -name \*.orig -print | xargs rm -f')
					system('find . -name \*.DS_Store -print | xargs rm -f')
					system("gem build *.gemspec")
					system("gem inabox *.gem")
					system('rm -rf *.gem')
				rescue Interrupt => e
					puts "Asked for stop.".color(:yellow)
					exit
				rescue Exception => e
					puts e.to_s
					puts e.backtrace.join("\n")
					puts "FULL STOP ON UNKNOWN ERROR!".color(:red)
					exit
				end
				Dir.chdir("..")

				puts "#{version_file} bumped to version #{matched_version} -> #{new_version}".color(:green).bright
			else
				puts "fail on #{version_file}".color(:red)
			end

		end
	end
end

desc "Change all the branches of the apps in a directory."
task :change_branch do

	if ENV["BRANCH"]
		puts "checking out branch #{ENV['BRANCH']}"
		Dir["*"].sort.each do |dir_name|
			if File.directory?(dir_name) && ![".", ".."].include?(dir_name)
				result = `cd \"#{dir_name}\" && git stash && git checkout #{ENV['BRANCH']} >/dev/null 2>&1`
	puts ($?.to_i == 0 ? "OK".color(:green) : "NA".color(:yellow)) + " #{dir_name}"
			end
		end
	end

end

desc "Updates all git repo's that are children of the current directory."
task :update_git do
	updates = {}

	def update_status(updates, dir_name)
		system("clear")
		updates.sort.each do |dir, status|
			puts status + " " + dir + (dir == dir_name ? " <--" : "")
		end
	end

	while true
		Dir["*"].sort.each do |dir_name|
			if File.directory?(dir_name) && ![".", ".."].include?(dir_name) && File.directory?(dir_name + "/" + ".git")

				updates[dir_name] = "UPD".color(:yellow).bright.background(:blue)

				update_status(updates, dir_name)

				begin
					result = `cd \"#{dir_name}\" && git fetch --all && git pull -q 2>/dev/null`
					updates[dir_name] = ($?.to_i == 0 ? "OK ".color(:green) : "RMU".color(:red))
				rescue Interrupt => e
					exit
				rescue Exception => e
					puts "unhandled exception: " + e.inspect
				ensure
					exit if $?.to_i == 1
				end

				update_status(updates, dir_name)
				sleep(1)
			else
				updates.delete(dir_name)
			end
		end
	end

end

desc "Prune all remote branches from repos."
task :prune_branches do
	updates = {}

	def update_status(updates, dir_name)
		system("clear")
		updates.sort.each do |dir, status|
			puts status + " " + dir + (dir == dir_name ? " <--" : "")
		end
	end

	while true
		Dir["*"].sort.each do |dir_name|
			if File.directory?(dir_name) && ![".", ".."].include?(dir_name) && File.directory?(dir_name + "/" + ".git")

				updates[dir_name] = "UPD".color(:yellow).bright.background(:blue)

				update_status(updates, dir_name)

				begin
					result = `cd \"#{dir_name}\" && git remote prune origin 2>/dev/null`
					updates[dir_name] = ($?.to_i == 0 ? "OK ".color(:green) : "RMU".color(:red))
				rescue Interrupt => e
					exit
				rescue Exception => e
					puts "unhandled exception: " + e.inspect
				ensure
					exit if $?.to_i == 1
				end

				update_status(updates, dir_name)
				sleep(1)
			else
				updates.delete(dir_name)
			end
		end
	end

end