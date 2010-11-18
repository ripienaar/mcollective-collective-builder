require 'facter'
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
    instance_home = "#{BASEDIR}/collective/#{identity}"

    return true if member_running?(identity)

    FileUtils.cd(instance_home)

    system("ruby -I #{instance_home}/lib mcollectived.rb --config etc/server.cfg --pidfile #{BASEDIR}/pids/#{identity}.pid")

    FileUtils.cd(BASEDIR)
end

def stop_member(identity)
    pid = member_running?(identity)

    return unless pid

    Process.kill(2, pid.to_i)

    FileUtils.rm("#{BASEDIR}/pids/#{identity}.pid")
end

def get_members
    return [] unless File.exist?("#{BASEDIR}/collective")

    Dir.entries("#{BASEDIR}/collective").select{|m| (m[0,1] != "." && m != "base")}
end

def setup_base
    basedir = "#{BASEDIR}/collective/base"
    FileUtils.mkdir_p basedir
    FileUtils.mkdir_p "pids"
    FileUtils.mkdir_p "logs"
    FileUtils.mkdir_p "client"

    system("git clone git://github.com/puppetlabs/marionette-collective.git #{basedir}")
end

def create_member(collective, identity, stompserver, stompuser, stomppass, stompport, version, instance_home=nil)
    instance_home = "#{BASEDIR}/collective/#{identity}" if instance_home.nil?
    templatedir   = "#{BASEDIR}/templates"
    pluginsource  = "#{BASEDIR}/plugins"
    clonedir      = "#{BASEDIR}/collective/base"

    FileUtils.mkdir_p instance_home
    FileUtils.cd(instance_home)

    system("git clone file:///#{clonedir} .")
    system("git checkout #{version}")

    render_template("#{templatedir}/server.cfg.erb", "#{instance_home}/etc/server.cfg", binding)
    render_template("#{templatedir}/client.cfg.erb", "#{instance_home}/etc/client.cfg", binding)

    FileUtils.cp("etc/facts.yaml.dist", "etc/facts.yaml")
    FileUtils.cp("#{pluginsource}/security/none.rb", "plugins/mcollective/security/none.rb")

    puts "=" * 40
    puts "Created a new instance #{identity} in #{instance_home}"
    puts "=" * 40
end

desc "List the collective members and their statusses"
task :list  do

    puts "=" * 40
    puts

    get_members.each do |member|
        puts "#{member}:\t#{member_running?(member) ? 'running' : 'stopped'}"
    end
end

desc "Starts all collective members"
task :start do
    get_members.each do |member|
        start_member(member)
        puts "Started #{member} status: #{member_running?(member)}"
    end
end

desc "Stops all collective members"
task :stop do
    get_members.each do |member|
        stop_member(member)
    end
end

desc "Create a new collective member in the collective subdirectory"
task :create do
    collective  = ask("Collective Name", "MC_NAME", "mcollectivedev")
    stompserver = ask("Stomp Server", "MC_SERVER", "stomp")
    stompuser   = ask("Stomp User", "MC_USER", "mcollective")
    stomppass   = ask("Stomp Password", "MC_PASSWORD", "secret")
    stompport   = ask("Stomp Port", "MC_PORT", "6163")
    version     = ask("MCollective Version", "MC_VERSION", "0.4.10")
    count       = ask("Instances To Create", "MC_COUNT", 10).to_i
    hostname    = `hostname -f`.chomp

    # TODO: validate all this stuff

    setup_base

    count.times do |i|
        create_member(collective, "#{hostname}-#{i}", stompserver, stompuser, stomppass, stompport, version)
    end

    create_member(collective, "client", stompserver, stompuser, stomppass, stompport, version, "#{BASEDIR}/client")
end

desc "Shut down and remove the collective"
task :clean => [:stop] do
    CLEANFILES = ["collective", "pids", "logs", "client"]
    FileUtils.rm_rf(CLEANFILES)
end
