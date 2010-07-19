# Status
This project is incomplete use at your own risk

## Notes
This project assumes that you are able to compile code on your machine and use git.  See NOTES.md for notes on how
I got things going so far.

# Intention
A good way to install what is needed and nothing more.  When it is possible to use a package managed by the linux 
distribution without installing junk or using an undesirable version then the package will be used.  This management 
application will install or uninstall source and distribution packages.

## QuickStart
Ensure you have a build environment, I am using debian so you may need to translate for your environment...

Installation notes:

    sudo apt-get install gcc git-core ruby
    mkdir -p ~/src
    git clone http://github.com/joshaven/NGINX_Plus.git ~/src/NGINX_Plus
    cd ~/src/NGINX_Plus
    ./server
    # From a web browser, brows to http://server_ip_or_name:5000
    # Use the nice web gui to add/remove/update features
    
Update notes:

    cd ~/src/NGINX_Plus
    git pull
    cilantro
    # From a web browser, brows to http://server_ip_or_name:5000/
    # Use the nice web gui to add/remove/update features
    