##############################################################################
# Rakefile - Configuration file for rake (http://rake.rubyforge.org/)
# Time-stamp: <Dim 2013-12-01 17:59 svarrette>
#
# Copyright (c) 2011 Sebastien Varrette <Sebastien.Varrette@uni.lu>
# .             http://varrette.gforge.uni.lu
#                       ____       _         __ _ _
#                      |  _ \ __ _| | _____ / _(_) | ___
#                      | |_) / _` | |/ / _ \ |_| | |/ _ \
#                      |  _ < (_| |   <  __/  _| | |  __/
#                      |_| \_\__,_|_|\_\___|_| |_|_|\___|
#
# Use 'rake -T' to list the available actions
#
# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program.  If not, see <http://www.gnu.org/licenses/>.
##############################################################################

require "rubygems"
require "bundler/setup"
require "stringex"

#require 'rubygems' # required for checking gem presence
#require 'erb'      # required for module generation
require 'yaml'     # required for config file parsing

### Configure colors ###
begin
    require 'term/ansicolor'
    COLOR = true
rescue Exception => e
    puts "/!\\ cannot find the 'term/ansicolor' library"
    puts "    Consider installing it by 'gem install term-ansicolor' or 'rake setup'"
    COLOR = false
end

# if RUBY_VERSION < '1.9'
#     abort "*** ERROR *** You should have ruby >= 1.9 ( Hash is ordered starting Ruby 1.9).\n" +
#         "*** ERROR *** Consider to use RVM (http://beginrescueend.com/) " +
#         " to switch easily from one version to another"
# end


TOP_SRCDIR  = File.expand_path(File.join(File.dirname(__FILE__), "."))
REPONAME    = File.basename( TOP_SRCDIR )
INCLUDE_DIR = "include"
POW_DIR     = File.expand_path( File.join(ENV['HOME'], '.pow') )
DEBUG      = ARGV.include?('DEBUG')

CONFIGFILE = File.join(TOP_SRCDIR, INCLUDE_DIR, 'config.yaml')

::CONFIG = YAML::load_file( CONFIGFILE )

# Inheritance from Octopress Rakefile
## -- Rsync Deploy config -- ##
# Be sure your public key is listed in your server's ~/.ssh/authorized_keys file
ssh_user       = "#{::CONFIG[:rsync][:ssh][:user]}@#{::CONFIG[:rsync][:ssh][:host]}"
ssh_port       = "#{::CONFIG[:rsync][:ssh][:port]}"
document_root  = "#{::CONFIG[:rsync][:document_root]}"
rsync_delete   = ::CONFIG[:rsync][:opts][:delete]
rsync_args     = "#{::CONFIG[:rsync][:opts][:extra]}"  # Any extra arguments to pass to rsync
deploy_default = "#{::CONFIG[:rsync][:cmd]}"

# This will be configured for you when you run config_deploy
#deploy_branch  = "gh-pages"

## -- Misc Configs -- ##

public_dir      = "#{::CONFIG[:octopress][:public_dir]}"     # compiled site directory
source_dir      = "#{::CONFIG[:octopress][:source_dir]}"     # source file directory
blog_index_dir  = '#{::CONFIG[:octopress][:blog_index_dir]}' # directory for your blog's index page (if you put your index in source/blog/index.html, set this to 'source/blog')
deploy_dir      = "#{::CONFIG[:octopress][:deploy_dir]}"     # deploy directory (for Github pages deployment)
stash_dir       = "#{::CONFIG[:octopress][:stash_dir]}"      # directory to stash posts for speedy generation
posts_dir       = "#{::CONFIG[:octopress][:posts_dir]}"      # directory for blog files
themes_dir      = "#{::CONFIG[:octopress][:themes_dir]}"     # directory for theme files
new_post_ext    = "#{::CONFIG[:octopress][:new_post_ext]}"   # default new post file extension when using the new_post task
new_page_ext    = "#{::CONFIG[:octopress][:new_page_ext]}"   # default new page file extension when using the new_page task
server_port     = "#{::CONFIG[:octopress][:server_port]}"    # port for preview server eg. localhost:4000




namespace "deploy" do

    #############   deploy:site #############
    desc "Deploy the website via rsync"
    task :rsync do #=> :generate do
        # Check if preview posts exist, which should not be published
        if File.exists?(".preview-mode")
            info "Found posts in preview mode, regenerating files ..."
            File.delete(".preview-mode")
            Rake::Task[:generate].execute
        end

		srcdir = "#{TOP_SRCDIR}/#{public_dir}"
		dstdir = "#{document_root}"
		#rsync_args    = "#{rsync_args} --rsync-path='sudo rsync'"
		rsync_mode    = (rsync_delete ? "--delete" : "--update")
		rsync_exclude = "--exclude '.git*' --exclude '.template*' "
		if File.exists?('./rsync-exclude')
			rsync_exclude = " --exclude-from '#{File.expand_path('./rsync-exclude')}'"
		end

		run %{
           rsync -e 'ssh -p #{ssh_port} -o ConnectTimeout=5' -avz #{rsync_mode} #{rsync_exclude} #{rsync_args}  #{srcdir}/ #{ssh_user}:#{dstdir}
           notify Octopress "Deployment of FoC website completed"
        }

    end

end



###########################
desc "Generate jekyll site"
task :generate do
    raise "### You haven't set anything up yet. First run `rake setup` to set up Octopress." unless File.directory?(source_dir)
    info "Generating Site with Jekyll"
    run %{
       compass compile --css-dir #{source_dir}/stylesheets
       jekyll
       notify Octopress "Generation of FoC website completed"
    }
end


##############################################################################
namespace "collect" do
    #################   collect:software   #####################
    desc "Get the list of configured Environment Modules from Gaia"
    task :software do
        run %{
            rsync include/scripts/modules2yaml.py access-gaia.uni.lu:/tmp/
            ssh access-gaia.uni.lu python /tmp/modules2yaml.py > source/status/module_status.yaml
        }
    end
    #################   collect:status   #####################
    desc "Get the cluster status files from {chaos,gaia} frontends"
    task :status do
        run %{
            rsync access-gaia.uni.lu:/usr/share/gaia_live_status.yaml source/status/gaia_live_status.yaml
            rsync access-chaos.uni.lu:/usr/share/chaos_live_status.yaml source/status/chaos_live_status.yaml
        }
    end
end


##############################################################################
namespace "git" do

    #################   git:init   ################################
    desc "Initialize your local clone of the repository for the git-flow management"
    task :init do
        git_flow_init()
    end

end




namespace :new do
    ##############################################################################
    # usage rake new_page[my-new-page] or rake new_page[my-new-page.html] or rake new_page (defaults to "new-page.markdown")
    desc "Create a new page in #{source_dir}/(filename)/index.#{new_page_ext}"
    task :page, :filename do |t, args|
        raise "### You haven't set anything up yet. First run `rake setup` to set up an Octopress theme." unless File.directory?(source_dir)
        args.with_defaults(:filename => 'new-page')
        page_dir = [source_dir]
        if args.filename.downcase =~ /(^.+\/)?(.+)/
            filename, dot, extension = $2.rpartition('.').reject(&:empty?)         # Get filename and extension
            title = filename
            page_dir.concat($1.downcase.sub(/^\//, '').split('/')) unless $1.nil?  # Add path to page_dir Array
            if extension.nil?
                page_dir << filename
                filename = "index"
            end
            extension ||= new_page_ext
            page_dir = page_dir.map! { |d| d = d.to_url }.join('/')                # Sanitize path
            filename = filename.downcase.to_url

            run %{ mkdir -p #{page_dir} }
            file = "#{page_dir}/#{filename}.#{extension}"
            if File.exist?(file)
                abort("rake aborted!") if ask("#{file} already exists. Do you want to overwrite?", ['y', 'n']) == 'n'
            end
            puts "Creating new page: #{file}"
            open(file, 'w') do |page|
                page.puts "---"
                page.puts "layout: page"
                page.puts "title: \"#{title}\""
                page.puts "date: #{Time.now.strftime('%Y-%m-%d %H:%M')}"
                page.puts "comments: false"
                page.puts "sharing: true"
                page.puts "footer: true"
                page.puts "toc: true"
                page.puts "---"
            end
        else
            puts "Syntax error: #{args.filename} contains unsupported characters"
        end
    end

    ##############
    desc "Create a new post in #{source_dir}/#{posts_dir}"
    task :post, :title do |t, args|
        if args.title
            title = args.title
        else
            title = get_stdin("Enter a title for your post: ")
        end
        raise "### You haven't set anything up yet. First run `rake install` to set up an Octopress theme." unless File.directory?(source_dir)
        run %{ mkdir -p #{source_dir}/#{posts_dir} }
        filename = "#{source_dir}/#{posts_dir}/#{Time.now.strftime('%Y-%m-%d')}-#{title.to_url}.#{new_post_ext}"
        if File.exist?(filename)
            abort("rake aborted!") if ask("#{filename} already exists. Do you want to overwrite?", ['y', 'n']) == 'n'
        end
        puts "Creating new post: #{filename}"
        open(filename, 'w') do |post|
            post.puts "---"
            post.puts "layout: post"
            post.puts "title: \"#{title.gsub(/&/,'&amp;')}\""
            post.puts "date: #{Time.now.strftime('%Y-%m-%d %H:%M')}"
            post.puts "author: #{ENV['GIT_AUTHOR_NAME']}"
            post.puts "comments: true"
            post.puts "categories: "
            post.puts "---"
        end
    end

end



##############################################################################
desc "Setup your local copy of the repository and install the octopress theme"
task :setup  => [ :submodules ]  do |t, args|
    # args.with_defaults(:theme => 'classic')
    # theme = (args[:theme] == 'theme' ? 'classic' : args[:theme])
    #File.directory?()

    info "setup (i.e. bootstrap i.e. install) your local copy of the repository"
    Rake::Task["git:init"].invoke
    run %{
       cd #{TOP_SRCDIR}
       rvm --create use default@ulhpc_www
       gem install bundler
       bundle install
    }
    if (`uname`.chomp == 'Darwin')
        unless File.directory?("#{POW_DIR}")
            info "install [POW](http://pow.cx)"
            run %{ curl get.pow.cx | sh  }
        end
        unless File.symlink?("#{POW_DIR}/#{REPONAME}")
            info "setup POW to use your local repository on 'rake watch'"
            run %{ ln -sf #{TOP_SRCDIR} #{POW_DIR}/#{REPONAME} }
        end
    end
end


##############################################################################
desc "Init and update submodules."
task :submodules do
    puts "======================================================"
    puts "Downloading FoC Website submodules...please wait "
    puts "======================================================"

    run %{
      cd #{TOP_SRCDIR}
      git submodule init
      git submodule update
      git submodule foreach 'git fetch origin; git checkout master; git reset --hard origin/master; git submodule update --recursive; git clean -dfx'
    }
    #git clean -dfx
end

namespace :theme do
    ##############################################################################
    desc "Install an octopress theme (eventiually installed previously as source via 'rake theme:source:install')"
    task :install do |t, args|
        theme = list_theme(themes_dir)
        if File.directory?(source_dir) || File.directory?("sass")
            abort("rake aborted!") if ask("A theme is already installed, proceeding will overwrite existing files. Are you sure?", ['y', 'n']) == 'n'
            [ source_dir, "sass"].each do |d|
                run %{
                   rsync -avz #{d} #{d}.before_#{theme}
                } if File.directory?( "#{d}" )
            end
        end
        info "Copying '#{theme}' theme into ./#{source_dir} and ./sass"
        run %{
           mkdir -p #{source_dir}/#{posts_dir} #{public_dir} sass
           rsync -avz --exclude "*custom*" #{themes_dir}/#{theme}/source/ #{source_dir}
           rsync -avz --exclude "*custom*" #{themes_dir}/#{theme}/sass/ sass
        }
		if ask("also copy the custom parts of the theme '#{theme}' ? ", ['y', 'n']) == 'y'
			run %{ 
            rsync -avz  #{themes_dir}/#{theme}/source/ #{source_dir}
            rsync -avz  #{themes_dir}/#{theme}/sass/ sass
            }
		end 
   end

    ################################
    desc "Clean completely a theme"
    task :clean do
        theme = list_theme(themes_dir)
        info("removing all components of the \'#{theme}\' theme?")
        really_continue?("Yes")
        basedir="#{themes_dir}/#{theme}"
        error("Unexisting theme directory '#{themes_dir}/#{theme}'") unless File.directory?("#{themes_dir}/#{theme}")
        to_remove = {
            :rm        => [],
            :custom    => [],
            :conflicts => {}
        }
        Dir["#{themes_dir}/#{theme}/**/*"].each do |elem|
            next unless elem =~ /sass|source/
            basedir_pattern = basedir.gsub(/\./, "\\\.")
            to_rm = "#{TOP_SRCDIR}/" + elem.gsub(/#{basedir_pattern}/, "").gsub(/^\//, '')
            to_remove[:rm]     << to_rm unless to_rm =~ /custom/
            to_remove[:custom] << to_rm if     to_rm =~ /custom/ && ! File.directory?("#{to_rm}")
            if File.exists?("#{to_rm}") & !  File.directory?("#{to_rm}")
                to_remove[:conflicts]["#{to_rm}"] = "#{TOP_SRCDIR}/#{elem}" unless FileUtils.identical?("#{elem}", "#{to_rm}")
            end
        end
        unless to_remove[:conflicts].empty?()
            warn "CONFLICTS found in the following files."
            puts to_remove[:conflicts].keys.to_yaml
            info "Proceed anyway? (You'll be asked to eventually delete the conflicting files)"
            really_continue?("no")
        end
        warn red("about to remove DEFINITIVELY the following files")
        puts to_remove[:rm].join(" ")
        really_continue?
        to_remove[:rm].each do |e|
            next if File.directory?(e)
            next if to_remove[:conflicts].include?("#{e}")
            run %{ rm -f #{e} }
        end
        to_remove[:custom].each do |c|
            info "custom file '#{c}' found"
            run %{
              head #{c}
            }
            next if ask_with_default_answer(red("delete the file #{c}?"), "yes") =~ /n/
            run %{
              rm -f #{c}
            }
        end
        to_remove[:conflicts].each do |s,c|
            info "conflicting file #{c} found"
            run %{diff -ru #{c} #{s} }
            next if ask_with_default_answer(red("delete the file #{c}?"), "yes") =~ /n/
            run %{
              rm -f #{c}
            }
        end
        warn "=> consider commiting your changes"
    end

    namespace :source do
        ##################################
        desc "Install a new theme source"
        task :install, :url do |t, args|
            if args.url
                url = args.url
            else
                url = get_stdin("Enter a git repository url for the theme: ")
            end
            abort("Not a git url") unless url =~ /\.git$/
            theme = File.basename("#{url}").gsub(/\.git$/, '')
            error("Empty theme name") unless theme
            error("The theme '#{theme}' already exists") if File.directory?("#{themes_dir}/#{theme}")
            info "about to install the theme '#{theme}' from the git repository #{url}"
            really_continue?
            run %{
               git submodule add #{url} #{themes_dir}/#{theme}
               git commit -s -m "[theme] add new theme '#{theme}' from git source" .gitmodules #{themes_dir}/#{theme}
            }
        end

        ##################################
        desc "Delete a theme source"
        task :delete  do |t, args|
            theme = list_theme(themes_dir)
            info("removing completely the sources of the \'#{theme}\' theme?")
            really_continue?("Yes")
            basedir="#{themes_dir}/#{theme}"
            error("Unexisting theme directory '#{basedir}'") unless File.directory?("#{basedir}")
            cmd=`git ls-files --error-unmatch --stage -- "#{basedir}" | grep -E '^160000'`
            unless $?.success?
                error "#{basedir} is not a submodule"
            end
            top_gitdir     = `git rev-parse --show-toplevel`
            submodule_path = `git ls-files --full-name "#{basedir}"`.chomp
            submodule_url  = `git config --get submodule."#{basedir}".url || echo "unknown"`.chomp
            error "Bad git path (#{submodule_path}) for the git submodule" unless File.directory?("#{submodule_path}")
            info("removing completely the git submodule '#{submodule_path}' (url: #{submodule_url})")
            really_continue?
            run %{
                cd #{top_gitdir}
                git config -f .gitmodules --remove-section submodule."#{submodule_path}" 2>/dev/null || echo -n ""
                git config --remove-section submodule."#{submodule_path}" 2>/dev/null || echo -n ""
                git rm --cached "#{submodule_path}"
                rm -rf "#{submodule_path}"
                git commit -s -m "[theme] remove theme submodule '#{submodule_path}' (git url: #{submodule_url})" .gitmodules #{submodule_path}
            }

        end
    end




end

##############################################################################
desc "Watch the site (using POW) and regenerate when it changes"
task :watch do
    raise "### You haven't set anything up yet. First run `rake setup` to set up your local site." unless File.directory?(source_dir)
    puts "Starting to watch source with Jekyll and Compass."

    run %{
        compass compile --css-dir #{source_dir}/stylesheets
    } unless File.exist?("#{source_dir}/stylesheets/screen.css")
    # system "compass compile --css-dir #{source_dir}/stylesheets" unless File.exist?("#{source_dir}/stylesheets/screen.css")
    jekyllPid  = Process.spawn({"OCTOPRESS_ENV"=>"preview"}, "jekyll --auto --limit_posts 1")
    compassPid = Process.spawn("compass watch")
    if (`uname`.chomp == 'Darwin')
        serverPid  = nil
        browserPid = Process.spawn("open http://#{REPONAME}.dev")
    elsif (`uname`.chomp == 'Linux')
        serverPid = Process.spawn("python3 -m http.server", :chdir => 'public')
        sleep(1)
        browserPid = Process.spawn("xdg-open http://localhost:8000")
    end
    trap("INT") {
        [jekyllPid, compassPid, serverPid].each { |pid| Process.kill(9, pid) rescue Errno::ESRCH }
        exit 0
    }

    [jekyllPid, compassPid].each { |pid| Process.wait(pid) }
end



#################################################################
desc "Provide various information on this Rakefile configuration"
task :info  do
    puts "TOP_SRCDIR   = #{TOP_SRCDIR}"
    puts "REPONAME     = #{REPONAME}"
    puts "POW_DIR       = #{POW_DIR}"
end

task :DEBUG do
end

private
def red(str)
    COLOR == true ? Term::ANSIColor.red(str) : str
end
def green(str)
    COLOR == true ? Term::ANSIColor.green(str) : str
end
def bold(str)
    COLOR == true ? Term::ANSIColor.bold(str) : str
end
def cyan(str)
    COLOR == true ? Term::ANSIColor.cyan(str) : str
end
def info(str)
    puts green("=> " + str)
end
def warn(str)
    puts cyan("/!\\ WARNING: " + str)
end
def error(str)
    abort red("*** ERROR *** " + str)
end
def run(cmds)
    puts bold("[Running] #{cmds}")
    #puts cmds.split(/\n */).inspect
    cmds.split(/\n */).each do |cmd|
        next if cmd.empty?
        system("#{cmd}") unless DEBUG
    end
end

# Execute a given command
def execute(cmd)
    sh %{#{cmd}} do |ok, res|
        if ! ok
            error("The command '#{cmd}' failed with exit status #{res.exitstatus}")
        end
    end
end

## Check for the presence of a given command
def command?(name)
    `which #{name}`
    $?.success?
end

##################################
### Octopress Theme management ###
##################################

## List the available themes
def list_theme(themes_dir)
    list = {
        0 => 'Exit'
    }
    index = 1
    Dir["#{themes_dir}/*"].each do |dir|
        if File.directory?(dir)
            entry = File.basename(dir)
            list[index] = entry
            index += 1
        end
    end
    puts list.to_yaml
    answer = ask_with_default_answer("=> Select the index of the theme you wish to select from the list", "0")
    raise SystemExit.new('exiting selection') if answer == '0'
    raise RangeError.new('Undefined index')   if Integer(answer) >= index
    return "#{list[Integer(answer)]}"
end


#############################
### GIT toolbox functions ###
#############################

## Get the current git branch
def git_branch?()
    # get_git_branches()[0]
    (`git branch --no-color 2>/dev/null  | grep -e "^*" | sed -e "s/^\* //"`).chomp
end

## Get an array of the local branches present (first element is always the
## current branch)
def get_git_branches()
    res = (`git branch --no-color | sort -r`).split
    res.delete("*")
    res
end

## Initialize the local repository to respect the git-flow organization
def git_flow_init()
    error("Unable to find 'grb' on your system") unless command?('grb')
    ::CONFIG[:gitflow][:branches].values.each do |branch|
        info("=> initialize remote tracking of the '#{branch}' branch")
        verbose(false) {
            run %{grb grab #{branch}}
        }
    end
    info("=> setup git-flow configuration")
    ::CONFIG[:gitflow][:branches].each do |t,branch|
        verbose(false) { run %{git config gitflow.branch.#{t} #{branch}} }
    end
    ::CONFIG[:gitflow][:prefix].each do |t,prefix|
        verbose(false) { run %{git config gitflow.prefix.#{t} #{prefix}} }
    end
end

## Run a given git flow command
def git_flow(name, type = 'feature', action = 'start', optional_args = '')
    error "Invalid git-flow type '#{type}'" unless ['feature', 'release', 'hotfix', 'support'].include?(type)
    error "Invalid action '#{action}'" unless ['start', 'finish'].include?(action)
    error "You must provide a name" if name == ''
    error "The name '#{name}' cannot contain spaces" if name =~ /\s+/
    sh "git flow #{type} #{action} #{optional_args} #{name}"
end

def git_flow_start(type, name, optional_args = '')
    git_flow(name, type, 'start', optional_args)
end
def git_flow_finish(type, name, optional_args = '')
    git_flow(name, type, 'finish', optional_args)
end

# These definitions are inherited from octopress
def ok_failed(condition)
    if (condition)
        w        puts green("OK")
    else
        puts red("FAILED")
    end
end

def get_stdin(message)
    print message
    STDIN.gets.chomp
end

## Ask a question
def ask_with_default_answer(question, default_answer='')
    print "#{question} "
    print "[Default: #{default_answer}]" unless default_answer == ''
    print ": "
    STDOUT.flush
    answer = STDIN.gets.chomp
    return answer.empty?() ? default_answer : answer
end

## Ask whether or not to really continue
def really_continue?(default_answer = 'Yes')
    pattern = (default_answer =~ /yes/i) ? '(Y|n)' : '(y|N)'
    answer = ask_with_default_answer( cyan("=> Do you really want to continue #{pattern}?"), default_answer)
    exit 0 if answer =~ /n.*/i
end

def ask(message, valid_options)
    if valid_options
        answer = get_stdin("#{message} #{valid_options.to_s.gsub(/"/, '').gsub(/, /,'/')}") while !valid_options.include?(answer)
    else
        answer = get_stdin(message)
    end
    answer
end

desc "list tasks"
task :list do
    puts "Tasks: #{(Rake::Task.tasks - [Rake::Task[:list]]).join(', ')}"
    puts "(type rake -T for more detail)\n\n"
end



#=======================================================================
# eof
#
# Local Variables:
# mode: ruby
# End:
