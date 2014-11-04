# How to Setup BOSH CLI on Mac OS 10.10 Yosemite

## 
## Step 1: Install *brew*

1. If you haven't already done so, install **brew** from [here](http://brew.sh/).
2. From a *Terminal* window, run `brew update && brew upgrade`
3. If you see any permission issues, make sure your current user ID owns `/usr/local/Cellar` and all its subdirectories. You can take ownership by executing `sudo chown -R $LOGNAME:admin /usr/local/Cellar`

## Step 2: Install *chruby* and *bundler*

1. Execute `brew install chruby`
2. Load the environmental variables in the current shell with `source /usr/local/opt/chruby/share/chruby/chruby.sh`
2. Add the init variables to your `.profile` by executing `echo "source /usr/local/opt/chruby/share/chruby/chruby.sh" >> ~/.profile`
3. Install the latest supported version of Ruby with `ruby-install ruby 2.1.2`
4. Select the proper version with `chruby ruby-2.1.2`
5. Install MySQL (a requisite for the *bundler*) with `brew install mysql`
6. Install Postgres (pre-requisite for *bundler*) with `brew install postgres`
6. Install `pg` (pre-requisite for *bundler*) with `gem install pg`
5. Install the Ruby bundler with `gem install bundler`

## Step 3: Install the BOSH CLI

1. If you haven't already done so, install xcode select with `xcode-select --install`
2. Execute `gem install bosh_cli`

## Step 4: Install BOSH
1. Create a local workspace in your user directory with `mkdir ~/workspace && cd ~/workspace`
2. Download BOSH with `git clone https://github.com/cloudfoundry/bosh.git`
3. Bundle the package with `cd bosh ; bundle`