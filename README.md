FORMAT: X-1A

# Onboarding
*Contributors:*

Step by step onboarding environment setup and configuration

## Prerequisites

- Make sure you get your `*@go-jek.com` setup
- Create an account on bitbucket
- Create an account on Slack by visiting `gojek.slack.com/signup`
- Ask yourself to be invited to gojek's apiary

## Environment

### Brew

`ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"`

### Git

`brew install -v git`

### Mysql

`brew install -v mysql`

Copy the default my-default.cnf file to the MySQL Homebrew Cellar directory where it will be loaded on application start:

`cp -v $(brew --prefix mysql)/support-files/my-default.cnf $(brew --prefix)/etc/my.cnf`


Configure MySQL to allow for the maximum packet size, only appropriate for a local or development server. Also, we'll keep each InnoDB table in separate files to keep ibdataN-type file sizes low and make file-based backups, like Time Machine, easier to manage multiple small files instead of a few large InnoDB data files. This is the first of many multi-line single commands. The following is a single, multi-line command; copy and paste the entire block at once:

```
cat >> $(brew --prefix)/etc/my.cnf <<'EOF'

# Echo & Co. changes
max_allowed_packet = 1073741824
innodb_file_per_table = 1
EOF
```

Uncomment the sample option for innodb_buffer_pool_size to improve performance:

`sed -i '' 's/^#[[:space:]]*\(innodb_buffer_pool_size\)/\1/' $(brew --prefix)/etc/my.cnf`

Now we need to start MySQL using OS X's launchd. This used to be an involved process with launchctl commands, but now we can leverage the excellent brew services command:

`brew tap homebrew/services`

`brew services start mysql`


By default, MySQL's root user has an empty password from any connection. You are advised to run mysql_secure_installation and at least set a password for the root user:

`$(brew --prefix mysql)/bin/mysql_secure_installation`

### Apache (optional)

Start by stopping the built-in Apache, if it's running, and prevent it from starting on boot. This is one of very few times you'll need to use sudo:

`sudo launchctl unload /System/Library/LaunchDaemons/org.apache.httpd.plist 2>/dev/null`

The formula for building Apache is not in the default Homebrew repository that you get by installing Homebrew. While we can use the format of brew install external-repo/formula, if an external formula relies on another external formula, you have to use the brew tap command first. I know, it's weird. So, we need to tap homebrew-dupes because "homebrew-apache/httpd22" relies on "homebrew-dupes/zlib". Whew:

`brew tap homebrew/dupes`


Let's install Apache 2.2 with the event MPM, and we'll use Homebrew's OpenSSL library since it's more up-to-date than OS X's:

`brew install -v homebrew/apache/httpd22 --with-brewed-openssl --with-mpm-event`

In order to get Apache and PHP to communicate via PHP-FPM, we'll install the mod_fastcgi module:

`brew install -v homebrew/apache/mod_fastcgi --with-brewed-httpd22`

To prevent any potential problems with previous mod_fastcgi setups, let's remove all references to the mod_fastcgi module (we'll re-add the new version later):

`sed -i '' '/fastcgi_module/d' $(brew --prefix)/etc/apache2/2.2/httpd.conf`

Add the logic for Apache to send PHP to PHP-FPM with mod_fastcgi, and reference that we'll want to use the file ~/Sites/httpd-vhosts.conf to configure our VirtualHosts. The parenthesis are used to run the command in a subprocess, so that the exported variables don't persist in your terminal session afterwards. Also, you'll see export USERHOME a few times in this guide; I look up the full path for your user home directory from the operating system wherever a full path is needed in a configuration file and "~" or a literal "$HOME" would not work.

```
(export USERHOME=$(dscl . -read /Users/`whoami` NFSHomeDirectory | awk -F"\: " '{print $2}') ; export MODFASTCGIPREFIX=$(brew --prefix mod_fastcgi) ; cat >> $(brew --prefix)/etc/apache2/2.2/httpd.conf <<EOF

# Echo & Co. changes

# Load PHP-FPM via mod_fastcgi
LoadModule fastcgi_module    ${MODFASTCGIPREFIX}/libexec/mod_fastcgi.so

<IfModule fastcgi_module>
  FastCgiConfig -maxClassProcesses 1 -idle-timeout 1500

  # Prevent accessing FastCGI alias paths directly
  <LocationMatch "^/fastcgi">
    <IfModule mod_authz_core.c>
      Require env REDIRECT_STATUS
    </IfModule>
    <IfModule !mod_authz_core.c>
      Order Deny,Allow
      Deny from All
      Allow from env=REDIRECT_STATUS
    </IfModule>
  </LocationMatch>

  FastCgiExternalServer /php-fpm -host 127.0.0.1:9000 -pass-header Authorization -idle-timeout 1500
  ScriptAlias /fastcgiphp /php-fpm
  Action php-fastcgi /fastcgiphp

  # Send PHP extensions to PHP-FPM
  AddHandler php-fastcgi .php

  # PHP options
  AddType text/html .php
  AddType application/x-httpd-php .php
  DirectoryIndex index.php index.html
</IfModule>

# Include our VirtualHosts
Include ${USERHOME}/Sites/httpd-vhosts.conf
EOF
)
```

We'll be using the file ~/Sites/httpd-vhosts.conf to configure our VirtualHosts, but the ~/Sites folder doesn't exist by default in newer versions of OS X. We'll also create folders for logs and SSL files:

`mkdir -pv ~/Sites/{logs,ssl}`

Let's populate the ~/Sites/httpd-vhosts.conf file. The biggest difference from my previous guides are that you'll see the port numbers are 8080/8443 instead of 80/443. OS X 10.9 and earlier had the ipfw firewall which allowed for port redirecting, so we would send port 80 traffic "directly" to our Apache. But ipfw is now removed and replaced by pf which "forwards" traffic to another port. We'll get to that later, but know that "8080" and "8443" are not typos but are acceptable because of later port forwarding. Also, I've now added a basic SSL configuration (though you'll need to acknowledge warnings in your browser about self-signed certificates):

`touch ~/Sites/httpd-vhosts.conf`

```
(export USERHOME=$(dscl . -read /Users/`whoami` NFSHomeDirectory | awk -F"\: " '{print $2}') ; cat > ~/Sites/httpd-vhosts.conf <<EOF
#
# Listening ports.
#
#Listen 8080  # defined in main httpd.conf
Listen 8443

#
# Use name-based virtual hosting.
#
NameVirtualHost *:8080
NameVirtualHost *:8443

#
# Set up permissions for VirtualHosts in ~/Sites
#
<Directory "${USERHOME}/Sites">
    Options Indexes FollowSymLinks MultiViews
    AllowOverride All
    <IfModule mod_authz_core.c>
        Require all granted
    </IfModule>
    <IfModule !mod_authz_core.c>
        Order allow,deny
        Allow from all
    </IfModule>
</Directory>

# For http://localhost in the users' Sites folder
<VirtualHost _default_:8080>
    ServerName localhost
    DocumentRoot "${USERHOME}/Sites"
</VirtualHost>
<VirtualHost _default_:8443>
    ServerName localhost
    Include "${USERHOME}/Sites/ssl/ssl-shared-cert.inc"
    DocumentRoot "${USERHOME}/Sites"
</VirtualHost>

#
# VirtualHosts
#

## Manual VirtualHost template for HTTP and HTTPS
#<VirtualHost *:8080>
#  ServerName project.dev
#  CustomLog "${USERHOME}/Sites/logs/project.dev-access_log" combined
#  ErrorLog "${USERHOME}/Sites/logs/project.dev-error_log"
#  DocumentRoot "${USERHOME}/Sites/project.dev"
#</VirtualHost>
#<VirtualHost *:8443>
#  ServerName project.dev
#  Include "${USERHOME}/Sites/ssl/ssl-shared-cert.inc"
#  CustomLog "${USERHOME}/Sites/logs/project.dev-access_log" combined
#  ErrorLog "${USERHOME}/Sites/logs/project.dev-error_log"
#  DocumentRoot "${USERHOME}/Sites/project.dev"
#</VirtualHost>

#
# Automatic VirtualHosts
#
# A directory at ${USERHOME}/Sites/webroot can be accessed at http://webroot.dev
# In Drupal, uncomment the line with: RewriteBase /
#

# This log format will display the per-virtual-host as the first field followed by a typical log line
LogFormat "%V %h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combinedmassvhost

# Auto-VirtualHosts with .dev
<VirtualHost *:8080>
  ServerName dev
  ServerAlias *.dev

  CustomLog "${USERHOME}/Sites/logs/dev-access_log" combinedmassvhost
  ErrorLog "${USERHOME}/Sites/logs/dev-error_log"

  VirtualDocumentRoot ${USERHOME}/Sites/%-2+
</VirtualHost>
<VirtualHost *:8443>
  ServerName dev
  ServerAlias *.dev
  Include "${USERHOME}/Sites/ssl/ssl-shared-cert.inc"

  CustomLog "${USERHOME}/Sites/logs/dev-access_log" combinedmassvhost
  ErrorLog "${USERHOME}/Sites/logs/dev-error_log"

  VirtualDocumentRoot ${USERHOME}/Sites/%-2+
</VirtualHost>
EOF
)
```

You may have noticed that ~/Sites/ssl/ssl-shared-cert.inc is included multiple times; create that file and the SSL files it needs:

```
(export USERHOME=$(dscl . -read /Users/`whoami` NFSHomeDirectory | awk -F"\: " '{print $2}') ; cat > ~/Sites/ssl/ssl-shared-cert.inc <<EOF
SSLEngine On
SSLProtocol all -SSLv2 -SSLv3
SSLCipherSuite ALL:!ADH:!EXPORT:!SSLv2:RC4+RSA:+HIGH:+MEDIUM:+LOW
SSLCertificateFile "${USERHOME}/Sites/ssl/selfsigned.crt"
SSLCertificateKeyFile "${USERHOME}/Sites/ssl/private.key"
EOF
)
```

```
openssl req \
  -new \
  -newkey rsa:2048 \
  -days 3650 \
  -nodes \
  -x509 \
  -subj "/C=US/ST=State/L=City/O=Organization/OU=$(whoami)/CN=*.dev" \
  -keyout ~/Sites/ssl/private.key \
  -out ~/Sites/ssl/selfsigned.crt
```

Start Homebrew's Apache and set to start on login:

`brew services start httpd22`

RUN WITH PORT 80

You may notice that httpd.conf is running Apache on ports 8080 and 8443. Manually adding ":8080" each time you're referencing your dev sites is no fun, but running Apache on port 80 requires root. The next two commands will create and load a firewall rule to forward port 80 requests to 8080, and port 443 requests to 8443. The end result is that we don't need to add the port number when visiting a project dev site, like "http://projectname.dev/" instead of "http://projectname.dev:8080/".

The following command will create the file /Library/LaunchDaemons/co.echo.httpdfwd.plist as root, and owned by root, since it needs elevated privileges:

```
sudo bash -c 'export TAB=$'"'"'\t'"'"'
cat > /Library/LaunchDaemons/co.echo.httpdfwd.plist <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
${TAB}<key>Label</key>
${TAB}<string>co.echo.httpdfwd</string>
${TAB}<key>ProgramArguments</key>
${TAB}<array>
${TAB}${TAB}<string>sh</string>
${TAB}${TAB}<string>-c</string>
${TAB}${TAB}<string>echo "rdr pass proto tcp from any to any port {80,8080} -> 127.0.0.1 port 8080" | pfctl -a "com.apple/260.HttpFwdFirewall" -Ef - &amp;&amp; echo "rdr pass proto tcp from any to any port {443,8443} -> 127.0.0.1 port 8443" | pfctl -a "com.apple/261.HttpFwdFirewall" -Ef - &amp;&amp; sysctl -w net.inet.ip.forwarding=1</string>
${TAB}</array>
${TAB}<key>RunAtLoad</key>
${TAB}<true/>
${TAB}<key>UserName</key>
${TAB}<string>root</string>
</dict>
</plist>
EOF'
```

This file will be loaded on login and set up the 80->8080 and 443->8443 port forwards, but we can load it manually now so we don't need to log out and back in:

`sudo launchctl load -Fw /Library/LaunchDaemons/co.echo.httpdfwd.plist`


### PHP (optional)

The following is for the latest release of PHP, version 5.6. If you'd like to use 5.3, 5.4 or 5.5, simply change the "5.6" and "php56" values below appropriately.

`brew install -v homebrew/php/php56`

Set timezone and change other PHP settings (sudo is needed here to get the current timezone on OS X) to be more developer-friendly, and add a PHP error log (without this, you may get Internal Server Errors if PHP has errors to write and no logs to write to):

`(export USERHOME=$(dscl . -read /Users/`whoami` NFSHomeDirectory | awk -F"\: " '{print $2}') ; sed -i '-default' -e 's|^;\(date\.timezone[[:space:]]*=\).*|\1 \"'$(sudo systemsetup -gettimezone|awk -F"\: " '{print $2}')'\"|; s|^\(memory_limit[[:space:]]*=\).*|\1 512M|; s|^\(post_max_size[[:space:]]*=\).*|\1 200M|; s|^\(upload_max_filesize[[:space:]]*=\).*|\1 100M|; s|^\(default_socket_timeout[[:space:]]*=\).*|\1 600|; s|^\(max_execution_time[[:space:]]*=\).*|\1 300|; s|^\(max_input_time[[:space:]]*=\).*|\1 600|; $a\'$'\n''\'$'\n''; PHP Error log\'$'\n''error_log = '$USERHOME'/Sites/logs/php-error_log'$'\n' $(brew --prefix)/etc/php/5.6/php.ini)`

Fix a pear and pecl permissions problem:

`chmod -R ug+w $(brew --prefix php56)/lib/php`

The optional Opcache extension will speed up your PHP environment dramatically, so let's install it. Then, we'll bump up the opcache memory limit:

```
brew install -v php56-opcache
/usr/bin/sed -i '' "s|^\(\;\)\{0,1\}[[:space:]]*\(opcache\.enable[[:space:]]*=[[:space:]]*\)0|\21|; s|^;\(opcache\.memory_consumption[[:space:]]*=[[:space:]]*\)[0-9]*|\1256|;" $(brew --prefix)/etc/php/5.6/php.ini
```

Finally, let's start PHP-FPM:

`brew services start php56`

Optional: At this point, if you want to switch between PHP versions, you'd want to: brew services stop php56 && brew unlink php56 && brew link php54 && brew services start php54. No need to touch the Apache configuration at all!


### DNSMasq

A difference now between what I've shown before, is that we don't have to run on port 53 or run dnsmasq as root. The end result here is that any DNS request ending in .dev reply with the IP address 127.0.0.1:

```
brew install -v dnsmasq
echo 'address=/.dev/127.0.0.1' > $(brew --prefix)/etc/dnsmasq.conf
echo 'listen-address=127.0.0.1' >> $(brew --prefix)/etc/dnsmasq.conf
echo 'port=35353' >> $(brew --prefix)/etc/dnsmasq.conf
```

Similar to how we run Apache and PHP-FPM, we'll start DNSMasq:

```
brew services start dnsmasq
```

With DNSMasq running, configure OS X to use your local host for DNS queries ending in .dev:

```
sudo mkdir -v /etc/resolver
sudo bash -c 'echo "nameserver 127.0.0.1" > /etc/resolver/dev'
sudo bash -c 'echo "port 35353" >> /etc/resolver/dev'
```

To test, the command ping -c 3 fakedomainthatisntreal.dev should return results from 127.0.0.1. If it doesn't work right away, try turning WiFi off and on (or unplug/plug your ethernet cable), or reboot your system.

### NodeJs

`brew install node`


### Bower

`$ npm install -g bower`

### Grunt

`$ sudo npm install -g grunt-cli`

### Mongo

`$ brew install mongodb`

you can explore other available installation options. The most important are:

//Install with Open SSL support
`$ brew install mongodb --with-openssl`

Check your mongodb installation

```
$ cd /usr/local/etc/
$ cat mongod.conf
systemLog:
  destination: file
  path: /usr/local/var/log/mongodb/mongo.log
  logAppend: true
storage:
  dbPath: /usr/local/var/mongodb
net:
  bindIp: 127.0.0.1
```

**Run MongoDB**

To execute MongoDB you can use the next command from the terminal:

`$ mongod --config /usr/local/etc/mongod.conf`

Or launch MongoDB in background followed by the command –fork

`$ mongod --config /usr/local/etc/mongod.conf --fork`

However, if you want to keep MongoDB running at any time,  even when you reboot the system, you should use the following commands:

```
//Start mongod main process on session start:
$ ln -sfv /usr/local/opt/mongodb/*.plist ~/Library/LaunchAgents

//Start MongoDB now, in background, and keep it running
$ launchctl load ~/Library/LaunchAgents/homebrew.mxcl.mongodb.plist
```

In addition, you will be able to use the following modifiers to change the default configurations:

```
//Select manually the data directory
$ mongod --dbpath <path to data directory>

//Select manually the log file
$ mongod --logpath <path to .log config file>

//Select manually the configguration file
$ mongod --config <path to .conf config file>
```

**Start using MongoDB**

From now, MongoDB is available in your system and you can start MongoDB Shell from anywhere. You just have to open a terminal and use the mongo command to start:

```
$ mongo

MongoDB shell version: 2.6.5
connecting to: test
Welcome to the MongoDB shell.
For interactive help, type "help".
For more comprehensive documentation, see http://docs.mongodb.org/
Questions? Try the support group http://groups.google.com/group/mongodb-user
>
```

## IDE

### Sublime

i am using a sublime text 3 as my development IDE you can do ST2, although the plugins provided here is ST3s

#### Package Control

In order to install all of the plugins listed below, we need to first install [package control](https://packagecontrol.io/installation)

The simplest method of installation is through the Sublime Text console.
The console is accessed via the `ctrl+` shortcut or the `View > Show Console` menu.
Once open, paste the appropriate Python code for your version of Sublime Text into the console.

This following phyton code is for ST3

    import urllib.request,os,hashlib; h = '2915d1851351e5ee549c20394736b442' + '8bc59f460fa1548d1514676163dafc88'; pf = 'Package Control.sublime-package'; ipp = sublime.installed_packages_path(); urllib.request.install_opener( urllib.request.build_opener( urllib.request.ProxyHandler()) ); by = urllib.request.urlopen( 'http://packagecontrol.io/' + pf.replace(' ', '%20')).read(); dh = hashlib.sha256(by).hexdigest(); print('Error validating download (got %s instead of %s), please try manual install' % (dh, h)) if dh != h else open(os.path.join( ipp, pf), 'wb' ).write(by)

with package control installed,
you can install the packages by doing a `command + shift + p`, type `install` and hit the `return` key
which will bring up the package list box

#### Sublime­CodeIntel

https://github.com/SublimeCodeIntel/SublimeCodeIntel

#### Sublime Linter

https://sublime.wbond.net/packages/SublimeLinter

From ver­sion 3 and up, Sublime Linter has become modular.
This means, that you have to install main pack­age first,
and then plugin/module for every language you need support for.
Each plugin has it’s own set of requirements, so please make sure you read them thoroughly.

[SublimeLinter-php](https://packagecontrol.io/packages/SublimeLinter-php)
[SublimeLinter-jshint](https://packagecontrol.io/packages/SublimeLinter-jshint)
[SublimeLinter-json](https://packagecontrol.io/packages/SublimeLinter-json)
[SublimeLinter-csslint](https://packagecontrol.io/packages/SublimeLinter-csslint)

#### SideBar Enhancements

https://sublime.wbond.net/packages/SideBarEnhancements

#### VCS Gutter

https://sublime.wbond.net/packages/VCS%20Gutter

#### Trailing­Spaces

https://github.com/SublimeText/TrailingSpaces


#### Bracket Highlighter

https://github.com/facelessuser/BracketHighlighter

#### Code formatter

https://github.com/akalongman/sublimetext-codeformatter

#### Alignment

http://wbond.net/sublime_packages/alignment

#### Key bindings

    [
        { "keys": ["super+shift+alt+t"], "command": "open_terminal" },
        { "keys": ["super+shift+t"], "command": "open_terminal_project_folder" },
        { "keys": ["ctrl+super+r"], "command": "reveal_in_side_bar" },
        { "keys": ["super+shift+f"], "command": "reindent", "args": {"single_line": false}},
        { "keys": ["super+ctrl+a"], "command": "alignment"},
        { "keys": ["super+shift+c"], "command": "code_formatter"}
    ]

#### Settings

    {  
        "bold_folder_labels":true,
        "color_scheme":"Packages/User/SublimeLinter/Monokai Soda (SL).tmTheme",
        "detect_indentation":false,
        "draw_indent_guides":true,
        "draw_white_space":"all",
        "folder_exclude_patterns":[  
            ".svn",
            ".git",
            ".hg",
            "CVS",
            "_build",
            "dist",
            "build",
            "site"
        ],
        "font_size":12,
        "highlight_line":true,
        "highlight_modified_tabs":true,
        "ignored_packages":[  
            "Vintage"
        ],
        "line_padding_bottom":1,
        "rulers":[  
            72,
            79
        ],
        "save_on_focus_lost":true,
        "tab_size":4,
        "theme":"Brogrammer.sublime-theme",
        "todo":{  
            "case_sensitive":true,
            "patterns":{  
                "CHANGED":"CHANGED[\\s]*?:+(?P<changed>\\S.*)$",
                "FIXME":"FIX ?ME[\\s]*?:+(?P<fixme>\\S.*)$",
                "NOTE":"NOTE[\\s]*?:+(?P<note>.*)$",
                "TODO":"TODO[\\s]*?:+(?P<todo>.*)$"
            }
        },
        "translate_tabs_to_spaces":true,
        "trim_trailing_white_space_on_save":true,
        "use_tab_stops":true,
        "word_separators":"./\\()\"'-:,.;<>~!@#$%^&*|+=[]{}`~?",
        "word_wrap":false
    }


### Sublime Themes

i personally dig the [brogrammer theme](https://packagecontrol.io/packages/Theme%20-%20Brogrammer)

To install themes, just use package control. So the process would be:

`ctrl + shift + p` or `cmd + shift + p`

Search for the package you want
Install it
Set the theme in `Preferences -> Settings – User`
To set a theme in the `Preferences -> Settings – User` file, just add a line that looks like:

    {
        "theme": "Brogrammer.sublime-theme"
    }


We’ll be providing the install name for each theme here. Let’s get started!

### iTerm2 (optional)

Since we're going to be spending a lot of time in the command-line, let's install a better terminal than the default one. Download and install [iTerm2](https://www.iterm2.com/downloads.html).

In Finder, drag and drop the **iTerm Application** file into the **Applications folder**.

You can now launch iTerm, through the **Launchpad** for instance. Let's just quickly change some preferences.

Colors and Font Settings

* Set hotkey to open and close the terminal to command + option + i
* Go to profiles -> Default -> Check silence bell
* Download the [Solarized dark iterm colors](https://github.com/altercation/solarized/tree/master/iterm2-colors-solarized) or use any theme you like.
* Then set these to your default profile colors.
* Change the cursor text and cursor color to yellow make it more visible
* Change the font to 14pt [Source Code Pro Lite](https://github.com/adobe-fonts/source-code-pro/releases/latest).


### Zsh (optional)

We'll install zsh for all the features offered by oh-my-zsh. The installation and usage is really intutive. The Env.sh is a config file we maintain so as to not pollute the ~/.zshrc too much. Env.sh holds aliases, exports, path changes etc.

`$ brew install zsh zsh-completions`

Install oh-my-zsh on top of zsh to getting additional functionality

`$ curl -L https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh | sh`

if still in the default shell, change default shell to zsh manually

`$ chsh -s /bin/zsh`

edit the .zshrc by opening the file in a text editor

```
ZSH_THEME=pygmalion
alias zshconfig="subl ~/.zshrc"
alias envconfig="subl ~/Projects/config/env.sh"
plugins=(git colored-man colorize github jira vagrant virtualenv pip python brew osx zsh-syntax-highlighting)
```

**Env.sh**

```
#!/bin/zsh

# PATH
export PATH="/usr/local/share/python:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
export EDITOR='subl -w'
# export PYTHONPATH=$PYTHONPATH
# export MANPATH="/usr/local/man:$MANPATH"

# Virtual Environment
export WORKON_HOME=$HOME/.virtualenvs
export PROJECT_HOME=$HOME/Projects
source /usr/local/bin/virtualenvwrapper.sh

# Owner
export USER_NAME="YOUR NAME"
eval "$(rbenv init -)"

# FileSearch
function f() { find . -iname "*$1*" ${@:2} }
function r() { grep "$1" ${@:2} -R . }

#mkdir and cd
function mkcd() { mkdir -p "$@" && cd "$_"; }

# Aliases
alias cppcompile='c++ -std=c++11 -stdlib=libc++'
```
### Robomongo (optional)
we gonna handling a lot query of mongo so.., we should use this to less the stress.
visit http://robomongo.org/ download it , install and ready to use.
remember mongo port is on 27017
and install the mongodb first, if u try connect to your local ^^!

## Tools

### Slack

### Sourcetree

### Postman

### Sequelpro

### Skype
