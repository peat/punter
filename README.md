# Punter

This is a simple tool for starting and stopping pre-configured servers in Amazon's EC2 cloud. It's specifically geared towards "one off" deployment and light weight prototyping.

If you're looking for sophisticated deployment and dependency management, I suggest you look at OpsCode Chef -- it's open source and quite nice at managing bigger deployments.

Punter has four basic tasks:

- Starting servers
- Stopping servers
- Listing running servers
- Listing server configurations

Out of the box, it's biased towards starting Ubuntu 12.04 LTS images on "small" EC2 instances with EBS storage. If you're not sure what that means, check out the [AWS Instance Types](http://aws.amazon.com/ec2/#instance).

_Punter_ works by stitching together shell scripts found in the `fragments/` directory, and running them on the server after it boots. Pretty simple!

## Installation

Punter is a set of Ruby rake tasks; as such, please install Ruby 1.9.2 (or newer) and the `bundler` gem, then run `bundle install` in the Punter root directory to install the other Ruby dependencies.

Then, update the `config.yml` file with your AWS credentials ([located here](https://portal.aws.amazon.com/gp/aws/securityCredentials#access_credentials)), specifically `access_key_id` and `secret_access_key`.

Second, install Ruby 1.9.2 (or better), and the `bundler` gem. You should have a To install the other dependencies:

```bash
$ bundle install
```

## Running

To see the available configuration profiles:

```bash
$ rake profiles
basic: basic
ruby: basic, ruby
lamp: basic, lamp
```

This lists out the names of each configuration profiles, and the script fragments they use.

In the first case, `basic` is the name of a profile that just runs the `basic` fragment. Second, the `ruby` profile stitches together the `basic` and `ruby` fragments. The `lamp` profile stitches together the `basic` and `lamp` fragments. Simple, 'eh?

```bash
$ rake start PROFILE=basic
```

This will start a server with the `basic` profile. Easy.

What servers are currently running?

```bash
$ rake servers
NAME       IP              INSTANCE      STARTED
basic      123.45.67.89    i-12345678    September 12, 2012
basic      234.56.78.9     i-45678901    September 5, 2012
```

This will list out all of the servers running on your AWS account, with their assigned names, IPs, AWS instance identifiers, and when they were started.

Maybe you want to terminate a server?

```bash
$ rake terminate INSTANCE=i-12345678
Are you sure you want to kill server "basic" running at "123.45.67.89"? [y/N] y
Terminating instance i-12345678.
```

It's interactive, and gives you a bit of information so that you can avoid gnarly UH OH moments.

## Logging

You can check the status of a new instance by looking at the `/punter.log` file on the server. It's not particularly detailed, but it can help you figure out what the system is doing, and if it's completed setting up the environment. For example, the log for a `ruby` profile looks like this:

```
STARTED PROFILE ruby
Started fragment 'basic'
Finished fragment 'basic'
Started fragment 'ruby'
Finished fragment 'ruby'
COMPLETED PROFILE ruby. Have fun!
```

_Punter_ automatically adds the profile and fragment starting and finishing messages.

## Fragments

"Fragments" are the building blocks of the server scripts. They are are `bash` scripts that are concatenated together, then run as _root_ on the box, immediately after it boots. The `basic` fragment is the simplest of the bunch:

```bash
# update packages
apt-get update
apt-get -y upgrade

# install packages we always want
apt-get -y install git-core tmux
```

All fragments can be found in the `fragments` directory.

Fragments also support ERB templating. Here's `fragments/credit.sh.erb`:

```ruby
<%= runtime_log("User #{`whoami`.chomp} started this server at #{Time.now}") %>
```

Please note that the ERB is evaluated _before_ you start the server.

## Profiles

A _profile_ is simply a list of fragments to put together, and is specified in the `profiles.yaml` file. For example:

```yaml
ruby:
  - basic
  - ruby
```

This specifies that the `ruby` profile is dependent on the `basic` and `ruby` fragments. _Punter_ will look for both `.sh` and `.sh.erb` fragments for a given name, so don't worry about the extension.

To inspect the the completely assembled and rendered profile before you start a server with it, you can inspect it. For example:

```bash
$ rake inspect PROFILE=credit
```

That displays the entire `credit` profile, including a rendered ERB fragment.

## Contributing

You're welcome to contribute fragments and other updates; just send over a pull request, and tell me if you're cool with the (Apache License 2.0)[http://www.apache.org/licenses/LICENSE-2.0.html].
