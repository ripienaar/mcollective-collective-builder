require 'erb'

STDOUT.sync = true
BASEDIR = File.dirname(__FILE__)

def ask(prompt, env, default="")
    return ENV[env] if ENV.include?(env)

    print "#{prompt} (#{default}): "
    resp = STDIN.gets.chomp

    resp.empty? ? default : resp
end

def render_template(template, output, scope)
    tmpl = File.read(template)
    erb = ERB.new(tmpl, 0, "<>")
    File.open(output, "w") do |f|
        f.puts erb.result(scope)
    end
end

def get_member_pid(identity)
    pidfile = "#{BASEDIR}/pids/#{identity}.pid"

    return nil unless File.exist?(pidfile)

    File.read(pidfile)
end

def member_running?(identity)
    pid = get_member_pid(identity)

    return false unless pid

    # TODO: actually verify the cmdline is mcollective?
    return pid.to_i if File.exist?("/proc/#{pid}")

    false
end

def start_member(identity)
    raise "You need to create a collective first using rake create" if get_members.size == 0

    instance_home = "#{BASEDIR}/collective/#{identity}"

    return true if member_running?(identity)

    FileUtils.cd(instance_home)

    system("ruby -I #{instance_home}/lib mcollectived.rb --config etc/server.cfg --pidfile #{BASEDIR}/pids/#{identity}.pid")

    FileUtils.cd(BASEDIR)
end

def stop_member(identity)
    raise "You need to create a collective first using rake create" if get_members.size == 0

    pid = member_running?(identity)

    return unless pid

    Process.kill(2, pid.to_i)

    FileUtils.rm("#{BASEDIR}/pids/#{identity}.pid")
end

def get_members
    return [] unless File.exist?("#{BASEDIR}/collective")

    Dir.entries("#{BASEDIR}/collective").select{|m| (m[0,1] != "." && m != "base")}
end

def setup_base(gitrepo, branch)
    basedir = "#{BASEDIR}/collective/base"
    FileUtils.mkdir_p basedir
    FileUtils.mkdir_p "pids"
    FileUtils.mkdir_p "logs"
    FileUtils.mkdir_p "client"

    system("git clone -q #{gitrepo} #{basedir} --branch #{branch}")
end

def get_random_file(type)
    templatedir   = "#{BASEDIR}/templates"
    files = Dir.entries("#{templatedir}/#{type}").select{|f| f[0,1] != "."}

    File.join(templatedir, type, files[rand(files.size)])
end

def create_member(collective, identity, stompserver, stompuser, stomppass, stompport, stompssl, version, instance_home=nil)
    instance_home = "#{BASEDIR}/collective/#{identity}" if instance_home.nil?
    templatedir   = "#{BASEDIR}/templates"
    pluginsource  = "#{BASEDIR}/plugins"
    clonedir      = "#{BASEDIR}/collective/base"

    FileUtils.mkdir_p instance_home
    FileUtils.cd(instance_home)

    system("git clone -q file:///#{clonedir} .")
    system("git checkout -q #{version}")

    render_template("#{templatedir}/server.cfg.erb", "#{instance_home}/etc/server.cfg", binding)
    render_template("#{templatedir}/client.cfg.erb", "#{instance_home}/etc/client.cfg", binding)

    puts "Created a new instance #{identity} in #{instance_home}"
    puts "=" * 40
end

def copy_facts
    get_members.each do |member|
        FileUtils.cp(get_random_file("facts"), "#{BASEDIR}/collective/#{member}/etc/facts.yaml")
    end
end

def copy_classes
    get_members.each do |member|
        FileUtils.cp(get_random_file("classes"), "#{BASEDIR}/collective/#{member}/etc/classes.txt")
    end
end

def copy_plugins
    FileUtils.cd BASEDIR
    pluginsource  = "#{BASEDIR}/plugins"

    get_members.each do |member|
        FileUtils.cp_r("#{pluginsource}/.", "#{BASEDIR}/collective/#{member}/plugins/mcollective")
    end

    FileUtils.cp_r("#{pluginsource}/.", "#{BASEDIR}/client/plugins/mcollective")
end

def stop_all
    get_members.each do |member|
        puts "Stopping member #{member}"
        stop_member(member)
    end
end

def start_all
    raise "You need to create a collective first using rake create" if get_members.size == 0

    get_members.each do |member|
        start_member(member)
        puts "Started #{member} status: #{member_running?(member)}"
    end
end

def status
    get_members.each do |member|
        puts "#{member}:\t#{member_running?(member) ? 'running' : 'stopped'}"
    end
end

desc "Sets up a subshell to use the new collective"
task :shell do
    raise "Don't know what shell to run you have no SHELL environment variable" unless ENV.include?("SHELL")
    raise "You need to create a collective first using rake create" if get_members.size == 0

    ENV["MCOLLECTIVE_EXTRA_OPTS"]= "--config=#{BASEDIR}/client/etc/client.cfg"
    ENV["RUBYLIB"] = "#{BASEDIR}/client/lib"

    puts
    puts "Running #{ENV['SHELL']} to start a subshell with MCOLLECTIVE_EXTRA_OPTS and RUBYLIB set"
    puts
    puts "Please run the following once started: "
    puts
    puts "    PATH=`pwd`/client:$PATH"
    puts
    puts "To return to your normal shell and collective just type exit"
    puts

    system("#{ENV["SHELL"]}")
end

desc "List the collective members and their statusses"
task :list  do
    status
end

desc "List the collective members and their statusses"
task :status do
    status
end

desc "Starts all collective members"
task :start do
    raise "You need to create a collective first using rake create" if get_members.size == 0

    start_all
end

desc "Stops all collective members"
task :stop do
    stop_all
end

desc "Copies new plugins and restarts all collective members"
task :update do
    copy_plugins
    copy_facts
    copy_classes
    stop_all
    start_all
end

desc "Restarts all collective members"
task :restart do
    stop_all
    start_all
end

desc "Create a new collective member in the collective subdirectory"
task :create do
    collective  = ask("Collective Name", "MC_NAME", "mcollectivedev")
    stompserver = ask("Stomp Server", "MC_SERVER", "stomp")
    stompport   = ask("Stomp Port", "MC_PORT", "6163")
    stompssl    = ask("Stomp SSL (y|n)", "MC_SSL", "n")
    stompuser   = ask("Stomp User", "MC_USER", "mcollective")
    stomppass   = ask("Stomp Password", "MC_PASSWORD", "secret")
    gitrepo     = ask("GIT Source Repository", "MC_SOURCE", "git://github.com/puppetlabs/marionette-collective.git")
    branch      = ask("Remote branch name", "MC_SOURCE_BRANCH", "master")
    version     = ask("MCollective Version", "MC_VERSION", branch == "master"? "1.0.0" : branch)
    count       = ask("Instances To Create", "MC_COUNT", 10).to_i
    countstart  = ask("Instance Count Start", "MC_COUNT_START", 0).to_i
    hostname    = `hostname -f`.chomp

    # TODO: validate all this stuff

    setup_base(gitrepo, branch)

    count.times do |i|
        create_member(collective, "#{hostname}-#{countstart + i}", stompserver, stompuser, stomppass, stompport, stompssl, version)
    end

    create_member(collective, "client", stompserver, stompuser, stomppass, stompport, stompssl, version, "#{BASEDIR}/client")

    copy_plugins
    copy_facts
    copy_classes

    puts
    puts "Created a collective with #{count} members:"
    puts
    puts "  Collective Name: #{collective}"
    puts "       Node Names: #{hostname}-{#{countstart}-#{count + countstart - 1}}"
    puts "    Stomp Version: #{version}"
    puts "     Stomp Server: stomp://#{stompuser}:#{stomppass}@#{stompserver}:#{stompport}"
    puts
    puts "To recreate this collective use this command:"
    puts
    puts "   MC_NAME=#{collective} MC_SERVER=#{stompserver} MC_USER=#{stompuser} \\"
    puts "   MC_PASSWORD=#{stomppass} MC_PORT=#{stompport} MC_VERSION=#{version} \\"
    puts "   MC_COUNT=#{count} MC_COUNT_START=#{countstart} MC_SSL=#{stompssl} \\"
    puts "   MC_SOURCE=#{gitrepo} \\"
    puts "   MC_SOURCE_BRANCH=#{branch} rake create"
    puts
    puts "The collective instances are stored in collective/* and a client is setup in client/"
    puts
    puts "Use rake start to start the collective, rake -T to see commands available to start,"
    puts "stop and update it."
    puts
end

desc "Shut down and remove the collective"
task :clean => [:stop] do
    CLEANFILES = ["collective", "pids", "logs", "client"]
    FileUtils.rm_rf(CLEANFILES)
end
