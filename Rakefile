require 'yaml'    # stdlib
require 'erb'     # stdlib
require 'aws-sdk' # gem

task :default do
  puts "Hi there! Run 'rake -T' to see what I can do!"
end

# loads the @config, @profiles, and sets up the @ec2 environment.
task :setup do
  @config = YAML.load_file('config.yml')
  @profiles = YAML.load_file('profiles.yml')

  AWS.config( :access_key_id => @config['access_key_id'], :secret_access_key => @config['secret_access_key'] )
  @ec2 = AWS::EC2.new
end

desc 'Lists all available profiles and their dependencies.'
task :profiles => :setup do
  @profiles.each do |p, fs|
    puts "#{p}: #{fs.join(', ')}"
  end
end

desc 'Starts a server with a given PROFILE.'
task :start => :setup do
  # required parameter
  profile = get_env!('PROFILE', 'Please specify the PROFILE for your new server.')

  # build up the user-data string for the EC2 instance
  user_data =  build_profile( profile )

  # build up the configuration options
  opts = {}
  opts[:image_id]       = env_or_config( 'image_id' )
  opts[:instance_type]  = env_or_config( 'size' )
  opts[:key_name]       = env_or_config( 'key_name' )
  opts[:user_data]      = user_data

  instance = @ec2.instances.create( opts )
  instance.add_tag( 'Profile', :value => profile )

  puts "Started instance: #{instance.instance_id}"
  puts "         profile: #{profile}"
  puts "        image_id: #{opts[:image_id]}"
  puts "   instance_type: #{opts[:instance_type]}"
  puts "        key_name: #{opts[:key_name]}"
end

desc 'Lists running servers.'
task :servers => :setup do

  # convenience method for formatting data
  def puts_line( *data )
    puts "%-10s  %-15s  %-10s  %-13s  %s" % data
  end

  puts_line "PROFILE", "IP", "INSTANCE", "STATUS", "STARTED"
  @ec2.instances.each do |i|
    puts_line i.tags['Profile'], i.public_ip_address, i.instance_id, i.status, i.launch_time
  end

end


desc "Terminates a specified INSTANCE"
task :terminate => :setup do
  instance_id = get_env!('INSTANCE', "Please specify the INSTANCE to terminate.")
  instance = @ec2.instances[instance_id]

  unless instance.exists?
    puts "I'm sorry, I couldn't find instance #{instance_id}. Try the 'servers' task to list out the IDs."
    exit 1
  end

  print "Terminate the server running the '#{instance.tags['Profile']}' profile at #{instance.public_ip_address}? [y/N] "
  if STDIN.gets.chomp =~ /y/i
    puts "Terminating."
    instance.terminate
  else
    puts "Not terminating."
  end
end

desc "Prints out an assembled PROFILE for inspection"
task :inspect => :setup do
  profile = get_env!("PROFILE", "Please specify a PROFILE to inspect.")
  
  puts build_profile( profile )
end

# ERB HELPERS ------------------------------------------------------------


# Logs a message *at runtime* on the server into ~/punter.log
def runtime_log( msg )
  escaped_msg = msg.gsub('"', '\"') # escape quotes
  "\necho \"#{escaped_msg}\" >> /punter.log\n"
end



# ETC ---------------------------------------------------------------------

# preferentially pulls from ENV, then @config
def env_or_config( key )
  env_key = key.upcase                 # foo -> FOO
  cfg_key = "default_#{key.downcase}"  # foo -> default_foo

  get_env( env_key ) || @config[cfg_key]
end

# assembles all of the fragments for a given profile, returning the result as a string.
# exits on failure to build the profile.
def build_profile( name )
  fragments = @profiles[name]

  if fragments.nil?
    puts "Could not load profile named '#{name}'; check your profiles.yml."
    exit 1
  end

  if fragments.empty?
    puts "No fragments found for profile named '#{name}'; check your profiles.yml."
    exit 1
  end

  header = "#!/bin/bash\n\n"
  header << runtime_log("STARTED PROFILE #{name}")

  body = fragments.collect { |f| load_fragment(f) }.join("\n")

  footer = "\n\n"
  footer << runtime_log("COMPLETED PROFILE #{name}. Have fun!")

  header + body + footer
end


# Inserts standard pre- and post-body content to a fragment before assembly:
# - comments about where fragments begin and end.
# - appending to a log file with status updates.
def decorate_fragment( fragment, name )
  out = "\n\n# BEGIN #{name} ------------ \n"
  out << runtime_log( "Started fragment '#{name}'" )
  out << fragment
  out << runtime_log( "Finished fragment '#{name}'" )
  out << "\n\n# END #{name} ------------\n"
end


# loads a fragment from a file; if it's an ERB file, it'll compute the results and return that.
# exits on failure to load the fragment.
def load_fragment( name )

  fragment = nil

  sh_target = File.join('fragments', name + '.sh')
  if File.exists?( sh_target )
    fragment = File.open( sh_target ).read
  end

  erb_target = sh_target + '.erb'
  if File.exists?( erb_target )
    template = File.open( erb_target ).read
    fragment = ERB.new( template ).result()
  end

  # if we got this far with a nil fragment, exit with an error.
  if fragment.nil?
    puts "Couldn't find #{sh_target} or #{erb_target}, exiting."
    exit 1
  end

  # wrap the fragment in with standard decorations.
  decorate_fragment( fragment, name )
end 


# returns an environment variable, or prints out a helpful message.
# exits on failure to find the variable.
def get_env!( name, err = nil )
  v = get_env(name)

  if v
    v
  else
    if err
      puts err
    else
      puts "Please specify environment variable '#{n}'."
    end
    exit 1
  end
end


# returns an environment variable or nil.
def get_env( n )
  ENV[n]
end

