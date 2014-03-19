BOSH CLI Local Setup


BOSH CLI is a command line interface used to interact with MicroBOSH and BOSH. Before you can use MicroBOSH or BOSH you need to install BOSH Command Line Interface. The following steps install BOSH CLI. You can install on either a physical or virtual machine.

Prerequisites
 •Install Ruby and RubyGems. Requires Ruby 1.9.3 or Ruby 2.0.0.
 •Install a Git client to pull down BOSH repositories from GitHub.

Install Local BOSH

Install the BOSH CLI gem:
$ gem install bosh_cli_plugin_micro --pre

 If you are using the rbenv Ruby environment manager, refresh the list of gems that rbenv knows about:
$ rbenv rehash

By giving the path to the Gemfile in your cloned BOSH repository, you can use the BOSH CLI gems without install them with the gem install command.

 Next Step: Install Micro BOSH

MicroBOSH is a single VM that includes all of the BOSH components. You use Micro BOSH to deploy BOSH. Installation steps for Micro BOSH are specific to the IaaS layer you are using. Go back to the Deploying Cloud Foundry page and select the page with your specific IaaS to continue.
