# macOS AMP stack: How to install Apache, PHP and MySQL development environment with Homebrew

This guide will walk you through the steps required to install a basic Apache, PHP and MySQL development environment using [homebrew](http://brew.sh). Basically, all you'll need to do is copy the commands below into Terminal. Copy one block at a time.

## Credits

-   [Echo & Co.'s blog](https://echo.co/blog/os-x-1010-yosemite-local-development-environment-apache-php-and-mysql-homebrew)
-   [How to run multiple PHP versions simultaneously under OS X El Capitan using standard Apache](https://medium.com/@wvervuurt/how-to-run-multiple-php-versions-simultaneously-under-os-x-el-capitan-using-standard-apache-98351f4cec67)
-   [Install icu4c version 63 with Homebrew](https://stackoverflow.com/a/55828190)

## Before we begin...

:bangbang: Due to iTerm2 paste speed issue or keyboard buffer or whatnot, **we have to use macOS native terminal** to paste and execute all the commands below. It has been observed that pasting a very long text in iTerm2 causes some weird random behaviour.

We're going to need to install Homebrew, a super awesome package manager for OS X. Installation instructions are available on the [homebrew website](http://brew.sh/).

Then, before we do anything else, update homebrew and then install `homebrew/services` to manage our Apache, PHP, MySQL and other daemons more easily.

```sh
brew update
brew tap homebrew/services
```

**The scripts in this guide should work in both zsh and bash.** If you notice that something does not work well, first make sure that you are running the commands in macOS terminal instead of other terminal apps. If it still gives you trouble, you may try running them in bash. Starting from Catalina, the default shell is zsh. To change the default back to bash, run the following:

```sh
# Only run this if zsh does not work well for you.
chsh -s /bin/bash
```

Now we're ready to begin.

## MariaDB

Let's start by installing MariaDB server. You may also choose to install MySQL or Percona MySQL. Their installation steps are similar to the ones below. :bangbang: **However, please make sure that you install only one MySQL equivalent DB server, otherwise the database may become corrupted.**

```sh
brew install -v mariadb
```

Duplicate the my.cnf file from the template.

```sh
cp $(brew --prefix)/etc/my.cnf.default $(brew --prefix)/etc/my.cnf.d/my.cnf
```

The following will configure MySQL to allow for the maximum packet size, only appropriate for a local or development server. Also, we'll keep each InnoDB table in separate files to keep ibdataN-type file sizes low and make file-based backups, like Time Machine, easier to manage multiple small files instead of a few large InnoDB data files. This is the first of many multi-line single commands. The following is a single, multi-line command; copy and paste the entire block at once:

```
cat >> $(brew --prefix)/etc/my.cnf.d/my.cnf <<_EOF_

# Configuration tweak
max_allowed_packet = 1073741824
innodb_file_per_table = 1
innodb_buffer_pool_size = 4096M
max_connections = 128
max_user_connections = 128
max_connect_errors = 1000000
# read https://github.com/Homebrew/legacy-homebrew/issues/47335
table_open_cache = 250
# read https://expressionengine.com/blog/mysql-5.7-server-os-x-has-gone-away
interactive_timeout = 300
wait_timeout = 300
open_files_limit = 4096

# For audit purpose
#server_audit_events=query_dml
##server_audit_events=query
#server_audit_file_path=/usr/local/var/log/mysqld/audit.log
#server_audit_logging=ON

# For recovery purpose
#innodb_force_recovery=3
#innodb_purge_threads=1
_EOF_
```

Start the service:

```sh
brew services start mariadb
```

Secure the installation, and do take note of the answers to all the follow up questions in the comment below:

```sh
sudo mysql_secure_installation
# Switch to unix_socket authentication [Y/n] n
# Change the root password? [Y/n] y
# Change your password to: root
# Remove anonymous users? [Y/n] y
# Disallow root login remotely? [Y/n] y
# Remove test database and access to it? [Y/n] y
# Reload privilege tables now? [Y/n] y
```

Test MySQL by logging in as root. Insert the root password when prompted.

```sh
mysql -uroot -p
```

## Percona MySQL (alternative, guide has not been updated)

If you have installed MariaDB above, you can skip this section. **Make sure that you install only one MySQL equivalent DB server, otherwise the database may become corrupted.**

```sh
# Install percona-server via homebrew
brew install -v percona-server
```

Configure my.cnf by copying the template my.cnf.default file in the `$(brew --prefix)/etc` folder (`$(brew --prefix)/etc/my.cnf` may exist for Percona and you may see error when running this command):

```sh
cp -v $(brew --prefix)/etc/my.cnf.default $(brew --prefix)/etc/my.cnf
```

The following will configure MySQL to allow for the maximum packet size, only appropriate for a local or development server. Also, we'll keep each InnoDB table in separate files to keep ibdataN-type file sizes low and make file-based backups, like Time Machine, easier to manage multiple small files instead of a few large InnoDB data files. This is the first of many multi-line single commands. The following is a single, multi-line command; copy and paste the entire block at once:

```sh
cat >> $(brew --prefix)/etc/my.cnf <<_EOF_

# Configuration tweak
max_allowed_packet = 1073741824
innodb_file_per_table = 1
innodb_buffer_pool_size = 4096M
max_connections = 128
max_user_connections = 128
max_connect_errors = 1000000
# read https://github.com/Homebrew/legacy-homebrew/issues/47335
table_open_cache = 250
# read https://expressionengine.com/blog/mysql-5.7-server-os-x-has-gone-away
interactive_timeout = 300
wait_timeout = 300
open_files_limit = 4096

# For audit purpose
#server_audit_events=query_dml
##server_audit_events=query
#server_audit_file_path=/usr/local/var/log/mysqld/audit.log
#server_audit_logging=ON

# For recovery purpose
#innodb_force_recovery=3
#innodb_purge_threads=1
_EOF_

```

Now we need to start MySQL service using brew services:

```sh
brew services start percona-server
```

By default, MySQL's root user has an empty password from any connection. You are advised to run **mysql_secure_installation** and at least set a password for the root user:

```sh
sudo mysql_secure_installation
```

Test MySQL by logging in as root. Insert the root password when prompted.

```sh
mysql -uroot -p
```

## Apache

Start by stopping the built-in Apache, if it's running, and prevent it from starting on boot. This is one of very few times you'll need to use sudo:

```sh
sudo launchctl unload -w /System/Library/LaunchDaemons/org.apache.httpd.plist 2>/dev/null
```

Let's install latest Apache (should be 2.4.33 as of 2018/06/21):

```sh
brew install -v httpd
```

Make sure that index.php is considered before index.html as DirectoryIndex:

```sh
sed -i '' 's/^\([[:space:]]*DirectoryIndex \)index.html$/\1index.php index.html/' $(brew --prefix)/etc/httpd/httpd.conf
```

Apache 2.4 also disables needed modules by default, so we need to enable them. Do read the comment in the script below why these modules are enabled.

```sh
# We will be running PHP-FPM, so we need proxy module for Apache to pass incoming request to PHP-FPM server.
# Using PHP-FPM also allows us to have different PHP versions for different sites.
sed -i '' 's/^#\(LoadModule proxy_module\)/\1/' $(brew --prefix)/etc/httpd/httpd.conf
sed -i '' 's/^#\(LoadModule proxy_fcgi_module\)/\1/' $(brew --prefix)/etc/httpd/httpd.conf
# HTTP proxy is to allow reverse proxy, which can be handy for various setup, including fronting the node.js server.
sed -i '' 's/^#\(LoadModule proxy_http_module\)/\1/' $(brew --prefix)/etc/httpd/httpd.conf
# Apache rewrite is very handy when we need to configure custom rules in .htaccess files.
sed -i '' 's/^#\(LoadModule rewrite_module\)/\1/' $(brew --prefix)/etc/httpd/httpd.conf
# SSL module is required to support https connection.
sed -i '' 's/^#\(LoadModule ssl_module\)/\1/' $(brew --prefix)/etc/httpd/httpd.conf
# This allows for multiple vhost (?)
sed -i '' 's/^#\(LoadModule vhost_alias_module\)/\1/' $(brew --prefix)/etc/httpd/httpd.conf
# We use macro to simplify vhost definition.
sed -i '' 's/^#\(LoadModule macro_module\)/\1/' $(brew --prefix)/etc/httpd/httpd.conf
```

Include ~/Sites/httpd-vhosts.conf

```sh
(export USERHOME=$(dscl . -read /Users/`whoami` NFSHomeDirectory | awk -F"\: " '{print $2}') ; cat >> $(brew --prefix)/etc/httpd/httpd.conf <<_EOF_

# Include our VirtualHosts
Include ${USERHOME}/Sites/httpd-vhosts.conf
_EOF_
)

```

We'll be using the file ~/Sites/httpd-vhosts.conf to configure our VirtualHosts, but the ~/Sites folder doesn't exist by default in newer versions of OS X.

We'll also create folders for logs and SSL files:

```sh
mkdir -pv ~/Sites/{logs,ssl}
```

Let's populate the ~/Sites/httpd-vhosts.conf file. Take note that homebrew apache is listening to port 8080/8443 instead of 80/443. We'll get to that later, but know that "8080" and "8443" are not typos but are acceptable because of later port forwarding. We will also add basic SSL configuration with self-signed certificates, which means that you'll need to acknowledge warnings in your browser:

```sh
(export USERHOME=$(dscl . -read /Users/`whoami` NFSHomeDirectory | awk -F"\: " '{print $2}') ; cat > $USERHOME/Sites/httpd-vhosts.conf <<_EOF_
#
# Define variables
#
Define SITES_FOLDER \${HOME}/Sites

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
# Set proxy timeout to be 300s.
#
ProxyTimeout 300

#
# Set up permissions for VirtualHosts in ~/Sites
#
<Directory "\${SITES_FOLDER}">
    Options FollowSymLinks MultiViews
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
    DocumentRoot "\${SITES_FOLDER}"
</VirtualHost>
<VirtualHost _default_:8443>
    ServerName localhost
    Include "\${SITES_FOLDER}/ssl/ssl-shared-cert.inc"
    DocumentRoot "\${SITES_FOLDER}"
</VirtualHost>

#===============================================================================

#
# VirtualHosts macro definition
#
<Macro VHost \$project \$version \$docroot \$alias>
    <VirtualHost *:8080>
        ServerName \$project.localhost
        ServerAlias \$alias
        CustomLog "\${SITES_FOLDER}/logs/\$project.localhost-access_log" combined
        ErrorLog "\${SITES_FOLDER}/logs/\$project.localhost-error_log"
        DocumentRoot "\$docroot"
        ProxyPassMatch ^/(.*\.php(/.*)?)\$ fcgi://127.0.0.1:90\$version\$docroot/\$1
    </VirtualHost>
    <VirtualHost *:8443>
        ServerName \$project.localhost
        ServerAlias \$alias
        Include "\${SITES_FOLDER}/ssl/ssl-shared-cert.inc"
        CustomLog "\${SITES_FOLDER}/logs/\$project.localhost-access_log" combined
        ErrorLog "\${SITES_FOLDER}/logs/\$project.localhost-error_log"
        DocumentRoot "\$docroot"
        ProxyPassMatch ^/(.*\.php(/.*)?)\$ fcgi://127.0.0.1:90\$version\$docroot/\$1
    </VirtualHost>
</Macro>

#
# How to use VHost macro
#
# Use VHost {projectname} {phpversion} {docroot} {alias}
# where:
# -   {projectname}: the name of the project
# -   {phpversion}: 2 digits of php version, eg 74 or 80 or 81
# -   {docroot}: absolute path of the docroot
# -   {alias}: e.g. "project.dev www.project.dev"
#
# Examples below

# example
Use VHost example 74 \${SITES_FOLDER}/example/docroot "example.dev www.example.localhost"
# --> local base urls are: example.localhost, example.dev, www.example.localhost
# --> PHP version 7.4 will be used for this site when being accessed from browser
# --> the docroot (where index.php is located) is at \${SITES_FOLDER}/example/docroot

# php80project
Use VHost php80project 80 \${SITES_FOLDER}/project/docroot " "
# --> local base urls are: php80project.localhost
# --> PHP version 8.0 will be used for this site when being accessed from browser
# --> the docroot (where index.php is located) is at \${SITES_FOLDER}/php80project/docroot

#===============================================================================

#
# VirtualHosts reverse proxy macro definition
#
<Macro VHostReverseProxy \$project \$target \$alias>
    <VirtualHost *:8080>
        ServerName \$project.localhost
        ServerAlias \$alias

        ProxyPass "/" "\$target"
        ProxyPassReverse "/" "\$target"

        CustomLog "\${SITES_FOLDER}/logs/\$project.localhost-access_log" combined
        ErrorLog "\${SITES_FOLDER}/logs/\$project.localhost-error_log"
    </VirtualHost>

    <VirtualHost *:8443>
        ServerName \$project.localhost
        ServerAlias \$alias
        Include "\${SITES_FOLDER}/ssl/ssl-shared-cert.inc"

        ProxyPass "/" "\$target"
        ProxyPassReverse "/" "\$target"

        CustomLog "\${SITES_FOLDER}/logs/\$project.localhost-access_log" combined
        ErrorLog "\${SITES_FOLDER}/logs/\$project.localhost-error_log"
    </VirtualHost>
</Macro>

#
# How to use VHostReverseProxy macro for reverse proxy
#
# Use VHostReverseProxy {projectname} {target} {alias}
# where:
# -   {projectname}: the name of the project
# -   {target}: target URL complete with protocol, e.g. "http://localhost:3000/"
# -   {alias}: e.g. "project.dev www.project.dev"
#
# Examples below

# nextjsproject
Use VHostReverseProxy nextjsproject "http://localhost:3000/" "frontend.nextjs.localhost"
# --> local base urls are: nextjsproject.localhost, frontend.nextjs.localhost
# --> route all request at the urls above to http://localhost:3000/

#===============================================================================

#
# Automatic VirtualHosts (NO PHP PROXY, ONLY HTML)
#
# A directory at \${SITES_FOLDER}/webroot can be accessed at http://webroot.localhost
# In Drupal, uncomment the line with: RewriteBase /
#

# This log format will display the per-virtual-host as the first field followed by a typical log line
LogFormat "%V %h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combinedmassvhost

# Auto-VirtualHosts with .dev
<VirtualHost *:8080>
    ServerName localhost
    ServerAlias *.localhost

    CustomLog "\${SITES_FOLDER}/logs/localhost-access_log" combinedmassvhost
    ErrorLog "\${SITES_FOLDER}/logs/localhost-error_log"

    VirtualDocumentRoot \${SITES_FOLDER}/%-2+
</VirtualHost>
<VirtualHost *:8443>
    ServerName localhost
    ServerAlias *.localhost
    Include "\${SITES_FOLDER}/ssl/ssl-shared-cert.inc"

    CustomLog "\${SITES_FOLDER}/logs/localhost-access_log" combinedmassvhost
    ErrorLog "\${SITES_FOLDER}/logs/localhost-error_log"

    VirtualDocumentRoot \${SITES_FOLDER}/%-2+
</VirtualHost>
_EOF_
)
```

You may have noticed that ~/Sites/ssl/ssl-shared-cert.inc is included multiple times; create that file and the SSL files it needs:

```sh
(export USERHOME=$(dscl . -read /Users/`whoami` NFSHomeDirectory | awk -F"\: " '{print $2}') ; cat > ~/Sites/ssl/ssl-shared-cert.inc <<_EOF_
SSLEngine On
SSLProtocol all -SSLv2 -SSLv3
SSLCipherSuite ALL:!ADH:!EXPORT:!SSLv2:RC4+RSA:+HIGH:+MEDIUM:+LOW
SSLCertificateFile "${USERHOME}/Sites/ssl/selfsigned.crt"
SSLCertificateKeyFile "${USERHOME}/Sites/ssl/private.key"
_EOF_
)
openssl req \
  -new \
  -newkey rsa:2048 \
  -days 3650 \
  -nodes \
  -x509 \
  -subj "/C=US/ST=State/L=City/O=Organization/OU=$(whoami)/CN=*.localhost" \
  -keyout ~/Sites/ssl/private.key \
  -out ~/Sites/ssl/selfsigned.crt
```

### Start Apache

Start Homebrew's Apache and set to start on login:

```sh
brew services start httpd
```

### Run with Port 80

You may notice that httpd.conf is running Apache on ports 8080 and 8443. Manually adding ":8080" each time you're referencing your dev sites is no fun, but running Apache on port 80 requires root. The next two commands will create and load a firewall rule to forward port 80 requests to 8080, and port 443 requests to 8443. The end result is that we don't need to add the port number when visiting a project dev site, like "http://projectname.localhost/" instead of "http://projectname.localhost:8080/".

The following command will create the file /Library/LaunchDaemons/co.echo.httpdfwd.plist as root, and owned by root, since it needs elevated privileges:

```sh
sudo $(echo "$SHELL") -c 'export TAB=$'"'"'\t'"'"'
cat > /Library/LaunchDaemons/co.echo.httpdfwd.plist <<_EOF_
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
_EOF_'
```

This file will be loaded on login and set up the 80->8080 and 443->8443 port forwards, but we can load it manually now so we don't need to log out and back in:

```sh
sudo launchctl load -Fw /Library/LaunchDaemons/co.echo.httpdfwd.plist
```

### Test that Apache is working fine

Let's create a simple HTML site to test if our Apache setup above is working.

```sh
mkdir -p ~/Sites/test
cat > ~/Sites/test/index.html <<_EOF_
<h1>Testing Apache installation</h1>
_EOF_
```

Let's also use this chance to trust our self-signed certificate we just created above.

```sh
open ~/Sites/ssl/selfsigned.crt
# Key in your password to import the self-signed cert into Keychain.
# Double click *.localhost certificate. Expand Trust accordion.
# Change all the values to "Always Trust". Close the window and key in your password when prompted.
```

Finally, let's open the site in the browser. Retrace your steps above if it doesn't work.

```sh
open https://localhost/test/index.html
```

## PHP & PHP-FPM

The following is to install PHP 8.0 FPM. For other versions, simply change the first line environment variable assignment accordingly, as long as it is available in homebrew. Check available version by running `brew search php`. To install the latest version, you still need to replace with the version number, e.g. if the latest version is `8.0`, change the first line to `export PHPVERSION=8.0`.

```sh
export PHPVERSION=8.0
brew install -v php@$PHPVERSION
```

Set timezone and change other PHP settings (sudo is needed here to get the current timezone on macOS) to be more developer-friendly, and add a PHP error log (without this, you may get Internal Server Errors if PHP has errors to write and no logs to write to):

```sh
(export USERHOME=$(dscl . -read /Users/`whoami` NFSHomeDirectory | awk -F"\: " '{print $2}') ; sed -i '-default' -e 's|^;\(date\.timezone[[:space:]]*=\).*|\1 \"'$(sudo systemsetup -gettimezone|awk -F"\: " '{print $2}')'\"|; s|^\(memory_limit[[:space:]]*=\).*|\1 1024M|; s|^\(post_max_size[[:space:]]*=\).*|\1 200M|; s|^\(upload_max_filesize[[:space:]]*=\).*|\1 100M|; s|^\(default_socket_timeout[[:space:]]*=\).*|\1 600|; s|^\(max_execution_time[[:space:]]*=\).*|\1 300|; s|^\(max_input_time[[:space:]]*=\).*|\1 600|; $a\'$'\n''\'$'\n''; PHP Error log\'$'\n''error_log = '$USERHOME'/Sites/logs/php-error_log'$'\n' $(brew --prefix)/etc/php/$PHPVERSION/php.ini)
```

Fix a pear and pecl permissions problem:

```sh
chmod -R ug+w $(brew --prefix php@$PHPVERSION)/lib/php
```

The optional Opcache extension will speed up your PHP environment dramatically, so let's install it. Then, we'll bump up the opcache memory limit:

```sh
sed -i '' "s|^;\(opcache\.enable[[:space:]]*=[[:space:]]*\)[0-1]|\11|; s|^;\(opcache\.memory_consumption[[:space:]]*=[[:space:]]*\)[0-9]*|\1256|;" $(brew --prefix)/etc/php/$PHPVERSION/php.ini
```

At this point, let's start PHP-FPM and link the commandline php to the version we just installed:

```sh
brew services start php@$PHPVERSION && brew link php@$PHPVERSION --force
```

Check the version and where `php` is pointing to:

```sh
php --version && which php && ls -al /usr/local/bin/php
```

Optional: At this point, if you want to switch PHP CLI to a different version, you can run `brew unlink` and `brew link`. E.g. assuming you have installed both 7.4 and 8.0 and you want to switch from 8.0 to 7.4, run: `brew unlink php@8.0 && brew link php@7.4 && brew services start php@7.4`. No need to touch the Apache configuration at all!

So far we have installed and configured PHP _CLI_ with a specific version. Now it's time to configure PHP-FPM to listen to a specific port, e.g. PHP 7.4 listens to port 9074, PHP 8.0 listens to port 9080. This way we can have multiple PHP-FPM running at the same time to allow different sites running on different PHP versions.

```sh
sed -i '' "s|^\(listen[[:space:]]*=[[:space:]]*127\.0\.0\.1:\)[0-9]*|\190${PHPVERSION//./}|;" $(brew --prefix)/etc/php/$PHPVERSION/php-fpm.d/www.conf
```

And restart the PHP server.

```sh
brew services start php@$PHPVERSION
```

We're now done with a single PHP version setup. To install the other versions, simply repeat all the steps in this section, starting with changing the `PHPVERSION` environment variable.

### Xdebug 3

First make sure that you are linking the correct PHP, and therefore PECL version:

```sh
php --version && which pecl && ls -al /usr/local/bin/pecl
```

Install xdebug using PECL:

```sh
pecl install xdebug
```

Add a custom xdebug.ini file (need to be updated for v3.x), please take note to change the intended PHP version first:

```sh
export PHPVERSION=7.4
```

```sh
sed -i '' "s|^zend_extension=\"xdebug.so\"||;" $(brew --prefix)/etc/php/$PHPVERSION/php.ini
(export USERHOME=$(dscl . -read /Users/`whoami` NFSHomeDirectory | awk -F"\: " '{print $2}') ; cat > $(brew --prefix)/etc/php/$PHPVERSION/conf.d/xdebug.ini <<_EOF_
[xdebug]
zend_extension="xdebug.so"
xdebug.max_nesting_level=256
xdebug.show_exception_trace=0
xdebug.collect_params=0
xdebug.mode=debug
xdebug.discover_client_host = 1
xdebug.client_host=127.0.0.1
xdebug.client_port=9003
xdebug.remote_handler = dbgp
xdebug.start_with_request=trigger
xdebug.log="${USERHOME}/Sites/logs/xdebug.log"
_EOF_
)
```

#### Set up Sublime Text 4 for Xdebug

1. Install [Xdebug Client](https://packagecontrol.io/packages/Xdebug%20Client) package.
2. Update port to 9003 in the User Xdebug setting.
3. Update the project setting and add:
    ```
    {
      ...
      "settings": {
        "xdebug": {
          "url": "https://test.localhost/",
        },
      },
      ...
    }
    ```


## DNSMasq

The end result here is that any DNS request ending in .localhost reply with the IP address 127.0.0.1:

```sh
brew install -v dnsmasq
echo 'address=/.localhost/127.0.0.1' > $(brew --prefix)/etc/dnsmasq.conf
echo 'listen-address=127.0.0.1' >> $(brew --prefix)/etc/dnsmasq.conf
echo 'port=35353' >> $(brew --prefix)/etc/dnsmasq.conf
```

Similar to how we run Apache and PHP-FPM, we'll start DNSMasq:

```sh
brew services start dnsmasq
```

With DNSMasq running, configure OS X to use your local host for DNS queries ending in .localhost:

```
sudo mkdir -v /etc/resolver && \
sudo $(echo "$SHELL") -c 'echo "nameserver 127.0.0.1" > /etc/resolver/localhost' && \
sudo $(echo "$SHELL") -c 'echo "port 35353" >> /etc/resolver/localhost'
```

To test, the command ping -c 3 fakedomainthatisntreal.localhost should return results from 127.0.0.1. If it doesn't work right away, try turning WiFi off and on (or unplug/plug your ethernet cable), or reboot your system.

## Congratulations! :tada: :confetti_ball:

You have finished the setup. What you need to do now is to restart your Mac and test out if the installation is working well. As of 20221124, the materials below is not relevant for PHP ≥ 7.4 and latest Homebrew version.

## Extra: Setup multiple PHP version and serve sites using different PHP version

### Special note on openssl and icu4c libraries as of 20200604

_Seems to only happen when you are using PHP 5.x, 7.x._

You may encounter errors like the following when running php:

```
dyld: Library not loaded: /usr/local/opt/openssl/lib/libcrypto.1.0.0.dylib
```

or

```
dyld: Library not loaded: /usr/local/opt/icu4c/lib/libicui18n.66.dylib
```

These were caused by the different versions of openssl and icu4c libraries required by different PHP versions.

-   php@5.6 from exolnet/homebrew-deprecated requires < openssl@1.1 (1.0.2t seems to work) and icu4c v64.2.
-   php@7.2 requires icu4c v66.1.
-   php@7.3 and php@7.4 requires icu4c v67.1.

#### openssl

[This formula](https://github.com/tebelorg/Tump/releases/download/v1.0.0/openssl.rb) points to 1.0.2t version. As the bottle name is different between openssl and openssl@1.1, there is no issue installing and keeping both.

```sh
brew install https://github.com/tebelorg/Tump/releases/download/v1.0.0/openssl.rb
```

Use `brew switch` to link the libraries.

```sh
# to link to 1.0.2t
brew switch openssl 1.0.2t
# to link to 1.1.1g
brew switch openssl@1.1 1.1.1g
```

Note: brew link / unlink does not work for openssl and icu4c.

#### icu4c

This is tricky. There is no good workaround for installing different versions of icu4c. Creating local tap and using `brew extract` seem to be the ideal case but it doesn't work (see [here](https://docs.brew.sh/Versions), [here](https://github.com/glensc/homebrew-tap/pull/1), and [here](https://github.com/Homebrew/brew/issues/6059)).

First identify which version you need to download.

```sh
cd $(brew --prefix)/Homebrew/Library/Taps/homebrew/homebrew-core/Formula
git log --follow icu4c.rb
```

Take note of the commit SHA for different versions. To prevent uninstallation of other versions, we need to link the library to the version we are about to install.

```sh
# create and switch to 67.1 stub
mkdir $(brew --prefix)/Cellar/icu4c/67.1 && brew switch icu4c 67.1
# install icu4c v67.1
brew reinstall icu4c
# create and switch to 66.1 stub
mkdir $(brew --prefix)/Cellar/icu4c/66.1 && brew switch icu4c 66.1
# install icu4c v66.1
brew reinstall https://raw.githubusercontent.com/Homebrew/homebrew-core/22fb699a417093cd1440857134c530f1e3794f7d/Formula/icu4c.rb
# create and switch to 64.2 stub
mkdir $(brew --prefix)/Cellar/icu4c/64.2 && brew switch icu4c 64.2
# install icu4c v64.2
brew reinstall https://raw.githubusercontent.com/Homebrew/homebrew-core/896d1018c7a4906f2c3fa1386aaf283497db60a2/Formula/icu4c.rb
```

Next just use `brew switch` in order to link to different versions. So to switch to different PHP versions, first unlink the current version `brew unlink php@{version}`, next switch to:

-   PHP 5.6: `brew switch icu4c 64.2 && brew switch openssl 1.0.2t && brew link php@5.6 --force`
-   PHP 7.2: `brew switch icu4c 66.1 && brew link php@7.2 --force`
-   PHP 7.3: `brew switch icu4c 67.1 && brew link php@7.3 --force`

