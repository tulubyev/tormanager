# Tor Manager

Ruby gem that provides a Tor interface with functionality: 

- Start and stop and monitor a Tor process. 
The Tor Process is monitored using [Eye](https://github.com/kostya/eye). 
- Retrieve the current Tor IP address and get new ip address upon request.
- Proxy web client requests through Tor.

## Installation

Install Tor: 

`sudo apt-get install tor`

Add this line to your application's Gemfile:

```ruby
gem 'tormanager'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install tormanager

## Usage

### Tor Process 

This section shows how to start, stop and manage Tor processes. 

Start a Tor process with default settings:

```ruby
require 'tormanager'

tor_process = TorManager::TorProcess.new
tor_process.start

#you can get the control password for the Tor process if you want: 
tor_process.settings[:control_password]
```

By default, the Tor process will use using port `9050` and control port `50500` and
will be spawned using [Eye](https://github.com/kostya/eye) ensuring that the process remains in a healthy state. 
Eye will generate a config file based on the default [template](https://github.com/joshweir/tormanager/blob/master/lib/tormanager/eye/tor.template.eye.rb). 
Eye will invoke the Tor process with command which looks like this:

    tor --SocksPort 9050 --ControlPort 50500 --CookieAuthentication 0 --HashedControlPassword "<hashedpassword from the :control_password>" --NewCircuitPeriod 60

Eye will restart the Tor process if a cpu percentage exceeds 10% for 3 consecutive readings (checked every 30 seconds). Eye will 
restart the Tor process if its memory exceeds 200MB for 3 consecutive readings (checked every 60 seconds). By default, will do minimal logging and 
will create a random `:control_password` which is used to generate the `HashedControlPassword` used to change the Tor password on request. 
More information can be found in the `TorManager::TorProcess` parameters table below.

You can also spawn a Tor process with control over parameters:

```ruby
tor_process = 
  TorManager::TorProcess.new tor_port: 9051, 
                             control_port: 50501, 
                             pid_dir: '/my/pid/dir',
                             log_dir: '/my/log/dir',
                             tor_data_dir: '/my/tor/datadir',
                             tor_new_circuit_period: 120,
                             max_tor_memory_usage_mb: 400,
                             max_tor_cpu_percentage: 15,
                             control_password: 'mycontrolpass',
                             eye_logging: true,
                             tor_logging: true
tor_process.start
```

See the table below for more info.	
	
When done with the Tor process, `stop` it: 

```ruby
tor_process.stop 
```
	
The following table describes the `TorManager::TorProcess` parameters:
	
| Parameter | Default Value | Description |
| --- | --- | --- |
| `:tor_port` | `9050` | The listening port of the Tor process. |
| `:control_port` | `50500` | The control port of the Tor process. |
| `:pid_dir` | `/tmp` | Eye will create a pid file in this location which stores the pid of the Tor process. The name of the pid file is of format: `tormanager-tor-<tor_port>-<parent_pid>.pid` **_†_**. |
| `:log_dir` | `/tmp` | If `:eye_logging` is `true`, Eye will create a log (`tormanager.eye.log`) in this location. If `:tor_logging` is `true`, Tor `stdall` is redirected to log (`tormanager-tor-<tor_port>-<parent_pid>.log` **_†_**) in this location. |
| `:tor_data_dir` | `nil` | If specified, Tor will use this directory location as the `--DataDirectory`. |
| `:tor_new_circuit_period` | `60` | In seconds, specifies the `--NewCircuitPeriod` that Tor should use. |
| `:max_tor_memory_usage_mb` | `200` | In megabytes, Eye will restart the Tor process if its memory exceeds this value for 3 consecutive readings (checked every 60 seconds). |
| `:max_tor_cpu_percentage` | `10` | Percentage value, Eye will restart the Tor process if it's cpu percentage exceeds this value for 3 consecutive readings (checked every 30 seconds). |
| `:eye_tor_config_template` | `tormanager/eye/tor.template.eye.rb` | Specify your own eye config template, it is recommended to use the default [template](https://github.com/joshweir/tormanager/blob/master/lib/tormanager/eye/tor.template.eye.rb) as your starting point.  |
| `:control_password` | Randomly generated by default. | By default, will do minimal logging and will create a random `:control_password` which is used to generate the `HashedControlPassword` used to change the Tor password on request. |
| `:tor_log_switch` | `nil` | If specified, sets the Tor `--Log` switch, for example a value of `notice syslog` will add `--Log "notice syslog"` to the Tor command. |
| `:eye_logging` | `nil` | If set to `true` will enable Eye logging in the `:log_dir` location, eye log: `tormanager.eye.log`  |
| `:tor_logging` | `nil` | If set to `true`, Tor `stdall` is redirected to log (`tormanager-tor-<tor_port>-<parent_pid>.log` **_†_**) in this location. |
| `:dont_remove_tor_config` | `nil` | By default, an eye configuration file is generated for the current Tor instance based on the `:eye_tor_config_template` stored in the `:log_dir` with name `tormanager.tor.<tor_port>.<parent_pid>.eye.rb` **_†_**. This generated file is removed when the Tor process is stopped by default. Setting `:dont_remove_tor_config` to `true` will not remove this file. |

**_†_** where `<tor_port>` is the `:tor_port` of the Tor process and `<parent_pid>` is the pid of the ruby process spawning the Tor process.

To stop any Tor instances that have been previously started by Tor Manager but were not stopped (say in the event of a parent process crash) **_††_**: 

```ruby
TorManager::TorProcess.stop_obsolete_processes 
```
	
Query whether Tor Manager has any Tor processes running on a particular port **_††_**: 

```ruby
TorManager::TorProcess.tor_running_on? port: 9050 
```

Query whether Tor Manager has any Tor processes running on a particular port associated to a particular parent ruby pid **_††_**:

```ruby
TorManager::TorProcess.tor_running_on? port: 9050, parent_pid: 12345
```

**_††_** Note that this command applies only to Tor processes that were started by Tor Manager. 
Tor processes that have been started external to Tor Manager will not be impacted.  

### Proxy Through Tor, Query IP Address, Change IP Address

Once you have a `TorProcess` started, you can: 

- [Proxy web client requests through Tor.](#proxy-through-tor)
- [Query the current Tor endpoint IP address.](#query-ip-address) 
- [Change the Tor endpoint IP address.](#change-ip-address) 

The remaining examples assume that you have instantiated a `TorProcess` ie: 

```ruby
tor_process = TorManager::TorProcess.new
tor_process.start
```

#### Proxy Through Tor

```ruby
tor_proxy = TorManager::Proxy.new tor_process: tor_process 
tor_proxy.proxy do 
  tor_ip = RestClient::Request.execute(
             method: :get,
             url: 'http://bot.whatismyipaddress.com').to_str
end
my_ip = RestClient::Request.execute(
             method: :get,
             url: 'http://bot.whatismyipaddress.com').to_str
```
	
Note that in the above code the `RestClient::Request` returning `tor_ip` is routed through the Tor endpoint because it is yielded 
through the `TorManager::Proxy#proxy` block. The following request returning `my_ip` will not be routed through the Tor endpoint 
hence returning your current IP address.

You could route [Capybara](https://github.com/teamcapybara/capybara) requests through the proxy (just make sure you use the `:poltergeist` driver and set the proxy 
to use the Tor socks proxy): 

```ruby
Capybara.default_driver = :poltergeist
Capybara.register_driver :poltergeist do |app|
  Capybara::Poltergeist::Driver.new(app, {
    :phantomjs => Phantomjs.path,
    :phantomjs_options => ["--proxy-type=socks5", "--proxy=127.0.0.1:9050"]
  })
end
tor_proxy = TorManager::Proxy.new tor_process: tor_process 
tor_proxy.proxy do 
  tor_ip = Capybara.visit('http://bot.whatismyipaddress.com').text
end
```

#### Query IP Address 

Query the Tor endpoint for the current IP address: 

```ruby
tor_proxy = TorManager::Proxy.new tor_process: tor_process 	
tor_ip_control = 
  TorManager::IpAddressControl.new tor_process: tor_process, 
                                   tor_proxy: tor_proxy 
tor_ip_control.get_ip									   
```
	
When the `TorManager::IpAddressControl#get_ip` method is called, the IP address is stored in instance variable: 

```ruby
tor_ip_control.ip 
```
	
#### Change IP Address

Get a new IP address: 

```ruby
tor_proxy = TorManager::Proxy.new tor_process: tor_process 	
tor_ip_control = 
  TorManager::IpAddressControl.new tor_process: tor_process, 
                                   tor_proxy: tor_proxy 
tor_ip_control.get_new_ip									   
```

When the `TorManager::IpAddressControl#get_new_ip` method is called, the IP address is stored in instance variable: 

```ruby
tor_ip_control.ip
```
	
## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/joshweir/tormanager.


## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).
