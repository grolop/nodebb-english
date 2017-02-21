
Ubuntu
======

This installation guide is optimized for **Ubuntu 16.04 LTS** and will install NodeBB with MongoDB as the database.
Fully patched LTS and equivalent **production** versions of software are assumed and used throughout.

---------------------

Install Node.js
---------------------

Naturally, NodeBB is driven by Node.js, and so it needs to be installed. Node.js is a rapidly evolving platform and so
installation of an LTS version of Node.js is recommended to avoid unexpected breaking changes in the future as part of
system updates. The `Node.js LTS Plan <https://github.com/nodejs/LTS>`_ details the LTS release schedule including
projected end-of-life.

To start, add the nodesource repository per the `Node.js Ubuntu instructions <https://nodejs.org/en/download/package-manager/#debian-and-ubuntu-based-linux-distributions>`_
and install Node.js:

.. code:: bash

	$ curl -sL https://deb.nodesource.com/setup_6.x | sudo -E bash -
	$ sudo apt install -y nodejs

Verify installation of Node.js and npm:

.. code:: bash

	$ node -v
	$ npm -v

Install MongoDB
---------------------

MongoDB is the default database for NodeBB. As noted in the `MongoDB Support Policy <https://www.mongodb.com/support-policy>`_
versions older than **3.x** are officially **End of Life** as of October 2016. This guide assumes installation of
**3.2.x**. If `Redis <https://redis.io>`_ or another database instead of MongoDB the
:doc:`Configuring Databases <../../configuring/databases>` section has more information.

Up to date detailed installation instructions can be found in the `MongoDB manual <https://docs.mongodb.com/v3.2/tutorial/install-mongodb-on-ubuntu/>`_.
Although out of scope for this guide, some MongoDB production deployments leverage clustering, sharding and replication
for high availibility and performance reasons. Please refer to the MongoDB `Replication <https://docs.mongodb.com/v3.2/replication/>`_
and `Sharding <https://docs.mongodb.com/v3.2/sharding/>`_ topics for further reading.

Abbreviated instructions below:

.. code:: bash

	$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv EA312927
	$ echo "deb http://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.2.list
	$ sudo apt update && sudo apt install -y mongodb-org

Start the service and verify service status:

.. code:: bash

	$ sudo service mongod start
	$ sudo service mongod status

If everything has been installed correctly the service status should show as ``active (running)``.

Configure MongoDB
---------------------

General MongoDB administration is done through the MongoDB Shell ``mongo``. A default installation of MongoDB listens
on port ``27017`` and is accessible locally. Access the shell:

.. code:: bash

  $ mongo

Switch to the built-in ``admin`` database:

.. code::

	> use admin

Create an administrative user (**not** the ``nodebb`` user) scoped to the ``admin`` database to manage MongoDB once
authorization has been enabled:

.. code::

	> db.createUser( { user: "<Enter a username>", pwd: "<Enter a secure password>", roles: [ { role: "readWriteAnyDatabase", db: "admin" }, { role: "userAdminAnyDatabase", db: "admin" } ] } )

To initially create a database that doesn't exist simply ``use`` it. Add a new database called ``nodebb``:

.. code::

  > use nodebb

The database will be created and context switched to ``nodebb``. Next create the `nodebb` user and add the appropriate
privileges:

.. code::

  > db.createUser( { user: "nodebb", pwd: "<Enter a secure password", roles: [ { role: "readWrite", db: "nodebb" }, { role: "clusterMonitor", db: "admin" } ] } )

The ``readWrite`` permission allows NodeBB to store and retrieve data from the ``nodebb`` database. The
``clusterMonitor`` permission provides NodeBB read-only access to query database server statistics which are then
exposed in the NodeBB Administrative Control Panel (ACP).

Exit the Mongo Shell:

.. code::

	> quit()

Enable database authorization in the MongoDB configuration file ``/etc/mongod.conf`` by uncommenting the line
``security`` and enabling authorization:

.. code:: yaml

  security:
    authorization: "enabled"

Restart MongoDB and verify the administrative user created earlier can connect:

.. code:: bash

	$ sudo service mongod restart
	$ mongo -u your_username -p your_password --authenticationDatabase=admin

If everything is configured correctly the Mongo Shell will connect. Exit the shell.

Install NodeBB
---------------------

First, the remaining dependencies should be installed if not already present:

.. code:: bash

	$ sudo apt-get install -y git build-essential

Next, clone NodeBB into an appropriate location. Here the ``opt`` directory is used:

.. code:: bash

	$ cd /opt
	$ sudo git clone -b v1.x.x https://github.com/NodeBB/NodeBB.git nodebb

This clones the NodeBB repository from the ``v1.x.x`` branch to ``/opt/nodebb``. A list of alternative branches are
available in the `NodeBB Branches <https://github.com/NodeBB/NodeBB/branches>`_ GitHub page.

Obtain all of the dependencies required by NodeBB and initiate the setup script:

.. code:: bash

  $ cd nodebb
  $ sudo npm install --production
  $ sudo ./nodebb setup

A series of questions will be prompt with defaults in parenthesis. The default settings are for a local server listening
on the default port ``4567`` with a MongoDB instance listening on port ``27017``. When prompted be sure to configure the
MongoDB username and password that was configured earlier for NodeBB. Once connectivity to the database is confirmed the
setup will prompt that initial user setup is running. Since this is a fresh NodeBB install a forum administrator must be
configured. Enter the desired administrator information. This will culminate in a ``NodeBB Setup Completed.`` message.

A configuration file :doc:`config.json <../../configuring/config>` will be created in the root of the nodebb directory,
in this case ``/opt/nodebb/config.json``. This file can be modified should you need to make changes such as changing the
database location or credentials used to access the database.

Next create a ``nodebb`` system user and give the account permissions over the ``/opt/nodebb`` folder and all
subdirectories. This will ensure that NodeBB can configure plugins and update.

.. code:: bash

	$ sudo adduser --system --group nodebb
	$ sudo chown -R nodebb:nodebb /opt/nodebb

The last setup item is to configure NodeBB to start automatically. Modern linux systems have adopted
`systemd <https://en.wikipedia.org/wiki/Systemd>`_ as the default init system. Configure nodebb to start via a systemd
unit file at the location ``/lib/systemd/system/nodebb.service``:

.. code::

	[Unit]
	Description=NodeBB forum for Node.js.
	Documentation=http://nodebb.readthedocs.io/en/latest/
	After=system.slice multi-user.target

	[Service]
	Type=simple
	User=nodebb

	StandardOutput=syslog
	StandardError=syslog
	SyslogIdentifier=nodebb

	Environment=NODE_ENV=production
	WorkingDirectory=/opt/nodebb
	ExecStart=/usr/bin/node loader.js --no-daemon --no-silent
	Restart=always

	[Install]
	WantedBy=multi-user.target

Finally, enable and start NodeBB:

.. code:: bash

	$ sudo systemctl enable nodebb
	$ sudo service nodebb start
	$ sudo service nodebb status

If everything has been installed and configured correctly the service status should show as ``active``. Assuming this
install was done on a Ubuntu Server edition without a desktop, launch a web browser from another host and navigate to
the address that was configured during the NodeBB setup via IP address or domain name. The default forum should load and
be ready for general usage and customization.
