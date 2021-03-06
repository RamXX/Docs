# How to Setup the BOSH CLI on Mac OS 10.10 Yosemite

## 
## Step 1: Install *brew*

1. Install install __XCode Select__ with `xcode-select --install`
2. Install **brew** from [here](http://brew.sh/) following their instructions. 
3. From a *Terminal* window, run `brew update && brew upgrade` to verify that all packages and modules are up to date.

## Step 2: Install *chruby* and *bundler*

1. Execute `brew install chruby`
2. Add the init variables to your `.profile` by executing `echo "source /usr/local/opt/chruby/share/chruby/chruby.sh" >> ~/.profile`
3. Force the selection of the proper ruby version every time with `echo "chruby ruby-2.1.2" >> ~/.profile`
4. Install `rubby-install` with `brew install ruby-install`
5. Install the latest supported version of Ruby with `ruby-install ruby 2.1.2`
6. Manually load the environmental variables in the current shell with `source /usr/local/opt/chruby/share/chruby/chruby.sh`
7. Select the proper version with `chruby ruby-2.1.2`
8. Install MySQL (a requisite for the *bundler*) with `brew install mysql`
9. Install Postgres (pre-requisite for *bundler*) with `brew install postgres`
10. Install `pg` (pre-requisite for *bundler*) with `gem install pg`
11. Install the Ruby bundler with `gem install bundler`

## Step 3: Install the BOSH CLI

1. Execute `gem install bosh_cli`

## Step 4: Install the Micro BOSH plugin

1. Execute `gem install bosh_cli_plugin_micro`
2. Test the system by attempting to list all the latest available public stemcells `bosh public stemcells`


## Step 5: (Optional) Install the BOSH source code
1. Create a local workspace in your user directory with `mkdir ~/workspace && cd ~/workspace`
2. Download BOSH with `git clone https://github.com/cloudfoundry/bosh.git`
3. Bundle the package with `cd bosh ; bundle`

