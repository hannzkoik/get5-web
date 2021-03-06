get5-web BETA
===========================

[![Build Status](https://travis-ci.org/splewis/get5-web.svg?branch=master)](https://travis-ci.org/splewis/get5-web)

This is an experimental web panel meant to be used in conjunction with the [get5](https://github.com/splewis/get5) CS:GO server plugin. It provides a more convenient way of managing matches and match servers.


## How to use it:
1. Create your game servers on the "Add a server" page by giving their ip, port, and rcon password
2. Create teams on the "Create a Team" page by listing the steamids for each of the players
3. Go to the "Create a Match" page, selecting the teams, server, and rules for the match

Once you do this, the site will send an rcon command to the game server ``get5_loadmatch_url <webserver>/match/<matchid>/config``, which will load the match config onto the gameserver automatically for you. Stats and game status will automatically be updated on the webpage.

As the match owner, you will be able to cancel the match. Additionally, on its matchpage there is a dropdown to run admin commands: add players to the teams if a ringer is needed, pause the match, load a match backup, list match backups, and run any rcon command.

Note: when using this web panel, the CS:GO game servers **must** be have both the core get5 plugin and the get5_apistats plugin. They are [released](https://github.com/splewis/get5/releases) together. This means the server must also be running the [Steamworks](https://forums.alliedmods.net/showthread.php?t=229556) and [SMJansson](https://forums.alliedmods.net/showthread.php?t=184604) extensions.

**WARNING**: This should be considered BETA software - it should not be considered STABLE and will contain BUGS. Changes in the master branch may be incompatible until it is given a 1.0.0 release tag.


## Screenshots

![Match Creation Page](/screenshots/create_match.png?raw=true "Match Creation Page")

![Match Stats Page](/screenshots/match_stats.png?raw=true "Match Stats Page")

![Teams Page](/screenshots/teams.png?raw=true "Teams Page")

![Team Creation Page](/screenshots/team_edit.png?raw=true "Team Creation Page")

## Requirements:
- python2.7
- MySQL (other databases will likely work, but aren't guaranteed to)
- a linux web server capable of running Flask applications ([see deployment options](http://flask.pocoo.org/docs/0.11/deploying/))


## Installation
```
# Clone the repository
git clone https://github.com/splewis/get5-web
cd get5-web

# Create a virtualenv, activate it, and install dependencies
virtualenv venv
source venv/bin/activate
pip install -r requirements.txt

# Now setup your config file
cd instance
cp prod_config.py.default prod_config.py
```
Now you can edit ``get5-web/instance/prod_config.py``, where you should change:
- ``SQLALCHEMY_DATABASE_URI``
- ``STEAM_API_KEY``
- ``SECRET_KEY``

You can also add a login whitelist if you wish (only users on the list will be able to login) and admin users:
```
WHITELISTED_IDS = [
    '76561198064755913',
]

ADMIN_IDS = [
    '76561198064755913',
]
```

Admins will be able to create "public" teams and servers - which can be used to create a match by any logged in user. If ``ADMINS_ACCESS_ALL_MATCHES`` is set to True, admins will also be able to send any rcon commands to servers and cancel any match.

Finally, initialize the database:
```
./manager.py db upgrade
```

If you want logos avaliable to use, you should upload them to the ``get5/static/img/logos`` directory as .png files. A good source is http://csgo-data.com/.


### Deployed using Apache2 with mod_wsgi

See the [official flask documentation](http://flask.pocoo.org/docs/0.11/deploying/mod_wsgi/) for installation requirements.

This assumes you cloned the repository inside ``/var/www``.
```
cd /var/www/get5-web
chown www-data logs
```

And create ``/var/www/get5-web/get5.wsgi``, with contents:
```
#!/usr/bin/python

activate_this = '/var/www/get5-web/venv/bin/activate_this.py'
execfile(activate_this, dict(__file__=activate_this))

import sys
import logging
logging.basicConfig(stream=sys.stderr)

folder = "/var/www/get5-web"
if not folder in sys.path:
    sys.path.insert(0, folder)
sys.path.insert(0,"")

from get5 import app as application
import get5
get5.register_blueprints()
```

Here is an example apache2 conf for /etc/apache2/sites-avaliable:
```
<VirtualHost *:80>
		ServerName get5.splewis.net
		ServerAdmin sean@splewis.net
		WSGIScriptAlias / /var/www/get5-web/get5.wsgi

		<Directory /var/www/get5>
			Order deny,allow
			Allow from all
		</Directory>

		Alias /static /var/www/get5-web/get5/static
		<Directory /var/www/get5-web/get5/static>
			Order allow,deny
			Allow from all
		</Directory>

		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

**Note: due to a limitation in how API requests are sent from the game server to the web server,
you must have a domain associated with the web panel. Just hosting it at an <ip:port> will not work and
will cause match updates from the game server to not reach the web panel.**

## How do the game server and web panel communicate?

1. When a server is added the web server will send ``get5_web_avaliable`` command through rcon that will check for the appropriate get5 plugins to be installed on the server
2. When a match is assigned to a server, the ``get5_loadmatch_url`` command  is used through rcon to tell the websever a file to download the get5 match config from
3. When stats begin to update (map start, round end, map end, series end), the game server plugins will send HTTP requests to the web server, using a per-match API token set in the ``get5_web_api_key`` cvar when the match was assigned to the server


## Other useful commands:

Autoformatting:
```
cd get5
autopep8 -r get5 --in-place
autopep8 -r get5 --diff # should have no output
```

Linting errors:
```
cd get5
pyflakes *.py
```

Testing:
You must also setup a ``test_config.py`` file in the ``instance`` directory.
```
./test.sh
```

Manually running a test instance: (for development purposes)
```
python2.7 main.py
```
